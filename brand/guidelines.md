# Kakaw brand

Kakaw lets a person ask their database a question in plain words. The name is the old word for cacao, so the identity is warm and made, not cold and corporate: cocoa, cream and one pod-gold accent. Voice: plain, friendly, concrete. Short sentences, no jargon, one real example instead of three claims.

## Rules

1. **Cream and cocoa do the work.** Page ground is `--kk-cream`, text is `--kk-cocoa`, cards and inputs sit on `--kk-raised` with a `--kk-line` border. Use tokens by name (`tokens.css`).
2. **Pod gold is the one accent.** It marks the highlight on one key word, the lead bar in a chart, and the pod glyph. It is 2.2:1 on cream, so it is for graphics and fills behind cocoa text, never for small text; use `--kk-pod-text` when gold must be text.
3. **No violet, no gradients, no glass.** Surfaces are flat. Depth comes from a 1px line, not shadows.
4. **Errors are red and labelled.** `--kk-error`, always with a text label such as `error:`, never colour alone.
5. **Fonts:** Bricolage Grotesque 800 for headings (tight tracking, -0.03em), Figtree for reading, IBM Plex Mono for labels, SQL and numbers. No other fonts. Smallest size 12px.
6. **Shape:** inputs and buttons 10px, cards 16px, status badges a full pill. Nothing else is round.
7. **The pod is the signature.** One shape recurs: a pod (a lens with a centre seam). It ends the wordmark, is the favicon and the app icon, and may mark an empty state. Never recolour it, rotate it or add effects.
8. **Emphasis:** one key word per heading gets the highlight (gold fill, cocoa text, 6px radius). Never two in a view.
9. **Motion:** short and quiet. Buttons step a shade on hover and press 1px on click; nothing bounces, tilts or loops. Respect reduced motion.

## Logo

The wordmark is `Kakaw` in Bricolage Grotesque 800 followed by the pod glyph at 0.62em tall, sitting on the baseline, 0.1em after the last letter. Files: `logos/mark.svg` (app icon, favicon), `logos/wordmark-on-light.svg`, `logos/wordmark-on-dark.svg`. In the product the wordmark is live text plus the glyph (`components/Logo.tsx`). Minimum width 88px; clear space one letter-k width on every side.

## Components

| Part | Spec |
| --- | --- |
| Primary button | Cocoa fill, cream text, 10px, lightens one step on hover, presses 1px |
| Secondary button | Raised fill, line border, tints to line on hover |
| Input | Raised fill, line border, cocoa border on focus, focus ring 2px cocoa at 20% |
| Card | Raised fill, 1px line, 16px radius, no shadow |
| Badge | Pill: cocoa fill = ready, line grey = in progress, error red with label = failed |
| Chart | Series order in `tokens.css`; lead series is pod gold, the rest follow |

## Open

- The pod glyph is a first draft; the logo files use live-font text, so outline them before print.
- Arabic is not designed for this brand yet.
