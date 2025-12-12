## Quick context

This repo is a mostly static checkout UI plus a tiny API proxy for payments. Primary concerns are front-end flows (UTM capture, TikTok pixel, modals/loaders) and a minimal server-side gateway that forwards requests to an external payments API.

Key locations:
- `front/` — main SPA-like frontend (`front/index.html`, `front/js/main.js`) with UI, modal logic and loaders.
- `checkout/` — payment pages and shared frontend helpers (`checkout/payment-api.js`) used to verify payments and build EMQ (Enhanced Match Quality) data for TikTok.
- `api/gateway.js` — lightweight ES module handler that proxies `acao=criar` and `acao=verificar` to an external payment endpoint.
- `index.html`, other `up*` folders — many static landing/upsell pages; routing is file-based (open HTML directly to debug).

## Big-picture architecture notes for the agent

- Frontend is static HTML + plain JS; many features are implemented client-side (UTM persistence in `localStorage` under `utm_params`, ttclid capture, EMQ hashing). Use browser/DevTools to step through flows.
- Payment verification has two integration points:
  - client → `pagamento/verifyPayment.php` (checkout pages expect a PHP endpoint; the PHP backend is not present in this repo)
  - optional Node handler `api/gateway.js` which implements similar behavior to the original PHP and forwards to a third-party payment API (see the hard-coded `GATEWAY_API_URL`).
- `api/gateway.js` is an ES module exporting an async handler (looks like a serverless / Next.js API style). It expects query params like `acao=criar` or `acao=verificar` and returns JSON.

## Developer workflows (what's practical here)

- Serve static pages locally: use any static server and open the HTML files directly. Example quick options:
  - `python -m http.server 8000` (from repo root)
  - `npx serve . -l 8000`
- The repo has no npm run scripts (see `package.json`). `api/gateway.js` won't run by itself; to test it locally you can either:
  - Deploy to a serverless platform that supports ES module API handlers (Vercel, Netlify functions), or
  - Wrap it in a tiny Express dev server (example: import the handler and call it from an Express route) — useful for local debugging.

## Project-specific conventions & patterns

- UTM handling and persistence:
  - The project stores UTM parameters in `localStorage` under the key `utm_params` and attempts to always append them to URLs so downstream pages and verification endpoints receive them.
  - `checkout/payment-api.js` contains helpers that ensure `ttclid` and other click IDs persist across pages.
- TikTok / EMQ conventions:
  - Events use hashed PII (SHA-256) for `email`, `phone_number`, and `external_id` before sending to TikTok (`checkout/payment-api.js` has `sha256`, `prepareEMQData`, `trackTikTokInitiateCheckout`, `trackTikTokIdentify`).
  - `ttq` is used as a queue or function; code defensively pushes to `window.ttq` if the pixel isn't loaded.
- Payment amounts:
  - Amounts are treated as strings, normalized to two decimals and then converted to integer cents before being sent to the gateway (see `api/gateway.js` logic that converts `valor` into cents).
- Phone formatting:
  - Client-side phone normalization uses E.164 heuristics (Brazil default `+55`) in `checkout/payment-api.js` (`formatPhoneToE164`).

## Integration points & gotchas (important for code changes)

- Secrets: `api/gateway.js` currently contains a long, hard-coded `GATEWAY_API_URL` (looks like a key/URL). Move this to environment variables before committing to any public repo or production deployment.
- Missing backends: many pages POST to `pagamento/verifyPayment.php` — the PHP backend is not present here. If you intend to run end-to-end locally, either provide the PHP endpoint, adapt calls to `api/gateway.js`, or mock the verification endpoints.
- Response shapes: `api/gateway.js` expects the external API to return `transactionId`, `pixCode`, and `status` for successful creates and `status` for verification. When writing changes, keep those fields consistent or adapt the front-end to the actual gateway response.

## Concrete examples for the agent

- To create a payment (what `api/gateway.js` expects):
  - GET or POST to `/api/gateway?acao=criar&nome=Nome&telefone=5511999999999&cpf=00000000000&valor=49.90&utm=utm_source=tt`
  - The handler will send a JSON body to the third party with `amount` in integer cents, `customer` with `name`, `email`, `phone`, `document`, and `paymentMethod: "PIX"`.
- To verify a payment: `/api/gateway?acao=verificar&payment_id=<transactionId>` → returns `{ status: '...' }` or `{ erro:1, erroMsg: '...' }`.

## What to change carefully

- Don't remove or change UTM/ttclid persisting behavior without ensuring downstream events still receive those params — attribution depends on it.
- If you refactor TikTok event helpers, preserve the EMQ hashing flow and the guarantee that fields are present (empty string when not available) — the code intentionally sends empty strings to keep coverage high.

## Quick checklist for PRs touching payments or tracking

1. Remove or replace hard-coded secrets (move to env).
2. Add a short manual test plan in the PR description: which HTML page to open, which flow to follow, and expected network calls (e.g., POST to `verifyPayment.php` or `/api/gateway`).
3. Note any changed response fields — update front-end consumers accordingly.

---
If you want, I can: (a) create a tiny Express dev server to run `api/gateway.js` locally for manual testing, (b) add a README section showing local run commands, or (c) scrub the hard-coded secret and replace it with a template using `process.env.GATEWAY_API_URL` — tell me which and I'll update the repo.

Please review and tell me if you'd like any part expanded (examples, local-run scripts, or a dev server added).
