# 04 Data and sync

## App database (PostgreSQL, Prisma)

Prisma migrations live in `backend/prisma/migrations`. The backend creates the database if it is missing and applies migrations at boot (`config/database.bootstrap.ts`).

```mermaid
erDiagram
    User ||--o{ Session : has
    User ||--o{ Account : has
    User ||--o{ DatabaseSource : owns
    User ||--o{ LlmConnection : owns
    User ||--o{ Conversation : owns
    User ||--o{ AgentRunTrace : owns

    DatabaseSource ||--o{ Conversation : "asked about"
    DatabaseSource ||--o{ DatabaseSourceTable : catalog
    DatabaseSource ||--o{ DatabaseSourceKnowledgeDocument : knowledge
    DatabaseSource ||--o{ QueryMemoryEntry : remembers
    DatabaseSource ||--o{ AgentRunTrace : traced

    DatabaseSourceTable ||--o{ DatabaseSourceColumn : has
    DatabaseSourceTable ||--o{ DatabaseSourceIndex : has
    DatabaseSourceTable ||--o{ DatabaseSourceRelation : "source of"

    LlmConnection ||--o{ Conversation : "used by"
    Conversation ||--o{ Message : contains
    Message ||--o{ MessageArtifact : has
    Message ||--o{ AgentRunTrace : "produced by"
    Conversation ||--o{ AgentRunTrace : traced

    DatabaseSource {
        string provider "postgresql, mongodb or sqlite"
        enum status "PENDING SYNCING READY FAILED"
        string connectionString "encrypted"
        string sqliteObjectKey "R2 key"
        string schemaFingerprint "sha256 of the schema"
    }
    LlmConnection {
        enum provider "OPENAI ANTHROPIC GOOGLE OPENROUTER OLLAMA LM_STUDIO"
        string encryptedApiKey "AES-256-GCM"
        string baseUrl "local and custom endpoints"
    }
    Message {
        enum role "USER ASSISTANT SYSTEM"
        enum status "PENDING STREAMING COMPLETED FAILED"
        json tokenUsage
        string[] suggestions
    }
    MessageArtifact {
        enum type "TEXT SQL TABLE CHART ERROR"
        json payload
    }
    AgentRunTrace {
        string requestId "unique"
        json steps
        json toolCalls
        json repairHistory
        string feedback "thumbs"
    }
    QueryMemoryEntry {
        string question
        string sql
        bytes embedding "384 float32"
        enum status "ACTIVE QUARANTINED RETIRED"
    }
```

Every record chains back to a `User` with `onDelete: Cascade`, so deleting an account removes everything it owns. Better Auth's own tables (`users`, `sessions`, `accounts`, `verifications`) share the same database.

### What each group is for

| Group | Tables | Purpose |
| --- | --- | --- |
| Identity | `users`, `sessions`, `accounts`, `verifications` | Better Auth |
| Connections | `database_sources`, `llm_connections` | What the user connected. Secrets are stored encrypted |
| Catalog | `database_source_tables`, `_columns`, `_indexes`, `_relations`, `_knowledge_documents` | The cached schema the agent prompts with. Columns keep a few sample values; short text columns (up to 50 distinct values, tables up to 200k rows, not secret-named) keep all of them in the same `sampleValues` field, and the prompt renders those as `values:` |
| Chat | `conversations`, `messages`, `message_artifacts` | The transcript. SQL, table and chart results are artifacts of an assistant message |
| Telemetry | `agent_run_traces` | One row per run: steps, tool calls, repair history, chart spec, cache info, thumbs feedback |
| Learning | `query_memory_entries` | Questions this source has answered correctly, with the SQL that ran |

## Connecting a database (onboarding and sync)

The frontend `proxy.ts` sends a new user through `/onboarding/database`, `/onboarding/connection` and `/onboarding/sync` before they can reach `/chat`.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant FE as Frontend
    participant BE as Backend
    participant R2 as R2 / S3
    participant AG as Agent
    participant UDB as User DB
    participant DB as App DB

    alt PostgreSQL or MongoDB
        U->>FE: connection string
        FE->>BE: POST /api/database-sources
        BE->>BE: check the format (scheme, MongoDB database name)
    else SQLite file
        U->>FE: choose .db file
        FE->>BE: POST /api/database-sources/uploads/presign
        BE-->>FE: presigned PUT url + object key
        FE->>R2: PUT file directly
        FE->>BE: POST /api/database-sources (object key)
    end

    BE->>DB: save source (encrypted), status SYNCING
    BE-->>FE: source, status SYNCING
    FE->>FE: sync page polls onboarding state

    BE->>AG: POST /api/chat/sync (60 s limit)
    AG->>UDB: introspect tables, columns, keys, indexes, row counts,<br/>3 sample rows, distinct values of short text columns
    AG->>AG: build knowledge documents, hash schema fingerprint
    AG-->>BE: catalog + fingerprint + sync metadata
    BE->>DB: replace catalog, status READY
    FE->>FE: sees READY, goes to /dashboard
```

If the sync fails the source becomes `FAILED` with `lastError`, and the sync page offers a retry (`POST /api/database-sources/:id/refresh`).

```mermaid
stateDiagram-v2
    [*] --> PENDING: created
    PENDING --> SYNCING: sync queued
    SYNCING --> READY: catalog saved
    SYNCING --> FAILED: agent error
    SYNCING --> FAILED: stuck over 5 min (sweeper)
    FAILED --> SYNCING: retry or refresh
    READY --> SYNCING: refresh
```

Only a `READY` source can start a chat.

### The schema fingerprint

`fingerprintCatalog` hashes only facts about the customer's schema: table and column names, types, nullability, primary keys, ordinal positions, indexes and foreign keys. Locally derived fields (such as normalised names) are left out on purpose, so changing how Kakaw normalises names never forces a re-sync of every source.

The fingerprint is the key that makes cached SQL safe:

- Query memory and the plan cache only reuse SQL written for the current fingerprint.
- After a refresh that changes the schema, old entries stop matching, because their SQL may name columns that no longer exist.

### SQLite path

The browser uploads straight to object storage with a presigned URL (max 25 MB, and `.db`, `.sqlite`, `.sqlite3` extensions checked at presign time). For each run the backend adds a presigned download URL. The agent downloads the file once into a temp cache (`/tmp/kakaw-sqlite`, keyed by the immutable object key, evicted after 7 days idle) and queries it read-only.

## Query memory (learning from answers)

Every completed run already stored its question and the SQL that produced the answer in `agent_run_traces`. Query memory turns that into something the next run can use.

```mermaid
flowchart TD
    subgraph write [After a completed run]
      T[AgentRunTrace saved] --> R[recordFromTrace]
      R -->|only the accepted sql_execution call,<br/>never a rejected attempt| E[(QueryMemoryEntry<br/>question, SQL, tables, embedding)]
    end

    subgraph read [Before the next run]
      Q[New question] --> X{Exact match?<br/>same source, same question,<br/>ACTIVE, same fingerprint}
      X -->|yes| P[exactPlan: skip SQL generation]
      Q --> S[Embed locally,<br/>cosine over source's entries]
      S -->|score at least 0.6, deduped by SQL,<br/>top few| EX[examples: few-shot in the prompt]
    end

    FB[Negative feedback via the API<br/>no UI control yet] -->|retire the entry<br/>that run created| E
    E --> X
    E --> S
```

- **Embeddings run in the backend process** (`Xenova/all-MiniLM-L6-v2`, 384 dimensions, about 3 ms per question). There is no embeddings API call, no extra key, and no customer schema leaves the infrastructure. If the model cannot load, retrieval quietly returns nothing and chat keeps working.
- **Isolation is structural**: entries belong to a `DatabaseSource`, which belongs to a user.
- One entry per question per source (`questionHash`), so asking again and getting it right overwrites a bad entry.
- Statuses: `ACTIVE`, `QUARANTINED`, `RETIRED`. Explicit negative feedback retires the entry a message created, through `sourceTraceId`.
- `npm run backfill:query-memory` rebuilds entries from existing traces.

## Secrets at rest

| Value | Protection |
| --- | --- |
| LLM API keys, database connection strings | AES-256-GCM in `SecretsService`, key from `LLM_CONNECTIONS_ENCRYPTION_KEY`. Only the last four characters of a key are shown to the user |
| Service secrets (`AGENT_SHARED_SECRET`, OAuth, encryption key) | Infisical, loaded into `process.env` before any module reads them |
| Session | HTTP-only cookie from better-auth, optionally shared across subdomains with `COOKIE_DOMAIN` |
