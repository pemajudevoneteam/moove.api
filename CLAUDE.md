# Moove API — Development Harness

Backend for the Moove Flutter app, running on **Cloudflare** (Workers + Hono + TypeScript, D1, R2, KV, Durable Objects).

The source of truth is [BACKEND_MASTER_PLAN.md](BACKEND_MASTER_PLAN.md). If this file and the plan disagree, the plan wins. Ask before deviating from it.

> Status: nothing is built yet. The repo only has a README. Work starts at Phase 0/1 of the plan.

## How to work (read this first)

1. **One phase at a time** (plan section 10). Never start the next phase early.
2. **At the end of each phase, stop.** Give a short summary, verification steps for a human, and wait for approval.
3. Read plan sections 1–4 before writing code.
4. Anything marked **VERIFY** in the plan (TomTom quota, caching rules, Cloudflare limits and prices) is unconfirmed. Check the provider's current docs before relying on it. Never guess.
5. If a task needs a decision the plan doesn't cover, ask. Don't invent one.

## Hard rules (never break)

- **No NFC, wallet, or card data on the server.** No columns, fields, logs or payloads for card numbers, NFC data or wallet balances. It stays on the device.
- **Every query is scoped by `user_id`**, enforced in the data-access layer, not in handlers.
- **Only `src/data/` touches D1, R2 or KV.** Route handlers call data functions. They contain no SQL and no binding calls.
- **The app never calls TomTom directly.** All provider calls go through the proxy Worker. The server key is never shipped in the app.
- **Do not store provider response bodies long term** (plan D7). Short TTL caches only, and store counts rather than payloads.
- **No secrets in the repo.** Use `wrangler secret put`. Never commit `.dev.vars` or real IDs for production.
- **Never touch production data or resources** while developing. Use local or staging.

## Conventions

- **IDs** are UUIDs. **Money** is integer sen, never floats. **Times** are integer Unix timestamps.
- Every row has `user_id` and `updated_at`.
- **Deletes are soft** (`deleted_at`) so sync can propagate them. A cron job purges them later.
- **Sync** is last-write-wins per row by `updated_at`.
- **Validate all input with zod.** Set request size limits.
- **Errors** use `{ "error": { "code": "...", "message": "..." } }`. The message is localized: `ms-MY` by default, `en` as fallback. Every user-facing error needs a Malay message.
- Text is UTF-8. Plates, currency (RM) and addresses follow Malaysian formats.
- No spatial SQL (SpatiaLite is unavailable on D1). Use lat/lon plus an indexed geohash column.

## Planned layout

```
backend/
  wrangler.toml
  package.json
  src/
    index.ts        # Hono app
    routes/         # one file per area
    data/           # data-access layer (only place that touches bindings)
    guards/         # QuotaGuard Durable Object, rate limiting
    providers/      # tomtom.ts, openfreemap.ts, valhalla.ts
  migrations/       # 0001_init.sql, ...
  test/
docs/
  backend_api.openapi.yaml
```

The Flutter app lives in a separate repository or folder. Do not mix app code into `backend/`.

## Commands

These will work once `backend/` exists (Phase 1). Run them from `backend/`.

```
npm install
npx wrangler dev                                        # local dev server
npm test                                                # run tests
npx wrangler d1 migrations apply moove-db --local       # apply migrations locally
npx wrangler deploy --env staging                       # deploy to staging
```

Production deploys (`--env production`, `--remote` migrations) are done by a human, not by an agent.

## Definition of done (every phase)

- Tests pass locally and in CI.
- New endpoints are in `docs/backend_api.openapi.yaml`.
- Works on staging against staging resources only.
- No secrets in the repo. No card, NFC or wallet data on the server.
- Every query is `user_id`-scoped. The data layer is the only code touching bindings.
- Malay messages exist for every user-facing error.
- A short summary and verification steps are written for the human reviewer.

## Humans own these

Cloudflare account, domain, 2FA, secrets, production deploys, TomTom keys and the paid-plan decision (plan section 9). If a task needs one of these, tell the human what is needed. Don't work around it.

## Git

- Work on a branch, not directly on `main`.
- Small commits with clear messages. Commit only when asked.
