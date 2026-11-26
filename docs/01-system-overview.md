# 01 System overview

## The services

| Service | Job | Talks to |
| --- | --- | --- |
| **Frontend** | The web app: login, onboarding, chat, docs, landing page | Backend (browser and server side) |
| **Backend** | The API and the only service with a user-facing session. Owns users, conversations, saved connections, query memory | App database, agent, object storage, Infisical |
| **Agent** | Turns a question into a query, runs it, writes the answer, plans a chart. Stateless apart from in-process caches | The user's database, the user's LLM, Infisical |
| **Monitoring** | Alloy collector that ships container logs to Grafana Cloud Loki | Docker socket, Grafana Cloud |

The backend is the gatekeeper. The browser never calls the agent, and the agent never reads the app database. Everything the agent needs for one turn arrives in a single signed request.

## Containers

```mermaid
flowchart TB
    subgraph Browser
      UI[React app]
    end

    subgraph Host["Dokploy host (Docker, Traefik)"]
      FE[frontend<br/>Next.js :3000]
      BE[backend<br/>NestJS :3001]
      AG[agent<br/>Node :8000]
      PG[(app Postgres)]
      AL[Alloy<br/>log collector]
    end

    subgraph External
      INF[Infisical<br/>secrets]
      R2[(R2 or S3<br/>SQLite uploads)]
      GC[Grafana Cloud<br/>Loki]
      UDB[(User's database)]
      LLM[[LLM provider]]
      OAUTH[Google and GitHub<br/>OAuth]
    end

    UI -->|pages| FE
    UI -->|API, cookies, CORS| BE
    FE -->|session check in proxy.ts| BE
    BE --> PG
    BE -->|HMAC signed, NDJSON| AG
    BE -->|presigned URLs| R2
    UI -->|PUT SQLite file, presigned| R2
    AG -->|download SQLite| R2
    AG --> UDB
    AG --> LLM
    BE -.->|test a connection| LLM
    BE --> OAUTH
    BE -.->|boot secrets| INF
    AG -.->|boot secrets| INF
    AL -.->|docker.sock logs| BE
    AL -.-> AG
    AL --> GC
```

## Why it is split this way

- **Agent separate from backend.** The agent dials arbitrary user-supplied hosts (a customer database, a custom LLM endpoint). Keeping that in its own container, with its own network guard, keeps that reach away from the app database and the session layer.
- **Backend resolves, agent executes.** The backend decrypts the user's keys and connection strings and builds a self-contained `AgentRunRequest`. The agent holds no user state between requests.
- **Streaming end to end.** The agent streams NDJSON events, the backend relays them (persisting as it goes), and the browser applies them to its cache as they arrive.

## How the services authenticate to each other

```mermaid
sequenceDiagram
    participant B as Backend
    participant A as Agent
    B->>B: timestamp = now<br/>signature = HMAC-SHA256(secret, timestamp + "." + body)
    B->>A: POST /api/chat/run<br/>x-kakaw-request-timestamp, x-kakaw-signature
    A->>A: reject if timestamp older than 5 minutes<br/>recompute HMAC, timingSafeEqual
    A-->>B: 200 NDJSON stream, or 4xx/5xx JSON
```

`AGENT_SHARED_SECRET` is loaded from Infisical by both services at boot. The agent exposes three routes:

| Route | Purpose |
| --- | --- |
| `GET /healthz` | Liveness, used by the Docker health check |
| `POST /api/chat/sync` | Introspect a database and return its catalog |
| `POST /api/chat/run` | Run one chat turn and stream events |

## Backend modules

```mermaid
flowchart LR
    APP[AppModule<br/>AuthGuard + RateLimitGuard<br/>+ GlobalExceptionFilter]
    APP --> AUTH[auth]
    APP --> USERS[users: /me]
    APP --> CHAT[chat]
    APP --> DS[database-sources]
    APP --> LC[llm-connections]
    APP --> PRISMA[prisma]
    CHAT --> RT[runtime-config]
    CHAT --> RET[retrieval]
    CHAT --> DK[database-knowledge]
    CHAT --> GW[agent-gateway]
    DS --> ST[storage: R2]
    DS --> DK
    DS --> GW
    RT --> DS
    RT --> LC
    RT --> DK
```

Every route needs a session unless marked `@Public()` (only `GET /api/health`). A global rate-limit guard budgets requests per user per minute; chat runs, syncs and connection tests have tighter limits. Counters are in memory, which is correct for one replica.

### HTTP API

| Area | Routes |
| --- | --- |
| Auth | `/api/auth/*` (better-auth: Google and GitHub sign-in, session) |
| Me | `GET /api/me/onboarding-state`, `/connections`, `/llm-connections`, `/chat-options`; `DELETE /api/me` |
| Database sources | `POST /api/database-sources`, `POST uploads/presign`, `POST :id/refresh`, `DELETE :id` |
| LLM connections | `POST /api/llm-connections`, `POST :id/activate`, `POST :id/deactivate`, `DELETE :id` |
| Chat | `GET/POST /api/chat/conversations`, `GET/DELETE conversations/:id`, `POST conversations/:id/runtime`, `POST conversations/:id/messages/stream`, `POST .../messages/:messageId/retry`, `PATCH .../traces/:traceId/feedback` |
| Health | `GET /api/health` |

## Supported inputs

| | Supported |
| --- | --- |
| User databases | PostgreSQL and MongoDB (connection string), SQLite (`.db`, `.sqlite`, `.sqlite3` upload) |
| LLM providers | OpenAI, Anthropic, Google, OpenRouter, Ollama, LM Studio |
| Sign-in | Google, GitHub |

The organization profile README also mentions MySQL and Snowflake. The code has no introspection or query path for either, so they are not supported yet.
