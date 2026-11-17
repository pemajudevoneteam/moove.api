# Moove Backend Master Plan

Date: 12 November 2026 · Status: **plan approved in discussion, nothing built yet**
Audience: AI coding agents (primary reference) and humans (setup and review).
Supersedes: [regional-map-transit-agent-blueprint.md](regional-map-transit-agent-blueprint.md) as the backend direction. Keep that file only for its data-source list and its stop-and-verify checkpoint idea.

> **How to use this document.** Agents: read sections 1–4 before writing any code, then work one phase at a time (section 10) and stop at each checkpoint. Humans: section 9 is the Cloudflare setup guide. Anything marked **VERIFY** is a fact from memory that must be checked against the provider's current docs before it is relied on.

---

## 1. Goals and non-goals

### Goals
1. A small, cheap, low-ops backend for the Flutter app, hosted mostly on **Cloudflare**.
2. Move the map engine to **TomTom** while **cutting TomTom requests** (free tier is limited) and keeping the key off the device.
3. Store user vehicle data (roadtax, insurance, maintenance, saved routes) and sync it across devices.
4. Support **Malay (ms-MY)** end to end.
5. Leave room for an **admin portal**, transit data, indoor navigation and offline packs later.
6. Survive growth: design now so the first migration is planned, not a crisis.

### Non-goals (for now)
- **No NFC or wallet card data on the server. Ever.** It stays on the device (secure storage). This keeps card data out of the backend and out of payment-data scope.
- No self-hosted PostGIS in the first release.
- No multimodal transit planner (OpenTripPlanner) until the transit phase.
- No scraping or permanent storage of Google, Waze or Apple data.

---

## 2. Decisions log

| # | Decision | Reason |
|---|---|---|
| D1 | Backend runs on Cloudflare Workers, D1, R2, KV, Durable Objects | Near-zero cost to start, no servers to patch, scales at the edge. |
| D2 | One small compute box is added **only** when routing or heavy ingestion needs it (Valhalla/OTP, GTFS jobs) | These cannot run on Workers. Not needed for phases 1–3. |
| D3 | The app never calls TomTom directly. All provider calls go through our proxy Worker | Keeps the API key secret, allows caching, quota control and fallback. |
| D4 | NFC and wallet data is local only | Decided by product owner. |
| D5 | Roadtax, insurance, maintenance and saved routes ("roadmap") are stored in D1 | Decided by product owner. |
| D6 | Guest-first accounts. Account only needed to sync or share | Matches the existing product plan ([DELIVERY_MASTER_PLAN.md](DELIVERY_MASTER_PLAN.md)). |
| D7 | Do not store provider response bodies long term until the TomTom terms are checked | Terms risk, see section 5.1. |
| D8 | Data-access layer isolates D1/DO specifics | So a later move to Postgres is a rewrite of one layer, not of the backend. |

---

## 3. Architecture

```
Flutter app (guest / local first, Hive + secure storage)
   │  HTTPS, JSON
   ▼
Cloudflare Worker  "moove-api"   (Hono, TypeScript)
   ├─ /v1/auth, /v1/sync, /v1/vehicles ...    ──►  D1   (relational user data)
   ├─ /v1/documents  ───────────────────────►  R2   (encrypted receipts/documents)
   ├─ /v1/maps/*  (TomTom proxy) ──► KV / Cache API (short-lived cache)
   │                              └► Durable Object "QuotaGuard" (counts, limits)
   │                              └► TomTom APIs, fallback: OpenFreeMap / Valhalla
   └─ Cron Triggers / Queues  ──────────────►  reminders, purge jobs, GTFS refresh

Admin portal (Cloudflare Pages)  ── Cloudflare Access (login gate) ──► same Worker, /admin/*

Later:  compute box (Valhalla / OTP / ingestion)  ◄── Queue ── Worker ──► R2 (outputs)
```

Principles:
- **Stateless Workers.** State lives in D1, R2, KV or Durable Objects.
- **One data-access layer** (`src/data/`). No SQL or binding calls inside route handlers.
- **Every row carries `user_id` and `updated_at`.** Needed for authorization, sync and later sharding.
- **IDs are UUIDs** (portable between databases).
- **Money is integer sen**, never floating point.

---

## 4. Data ownership

| Data | Where | Synced to server |
|---|---|---|
| NFC / wallet cards (last4, type, expiry, provider state) | Device secure storage | **No** |
| Vehicles | D1 | Yes |
| Roadtax | D1 | Yes |
| Insurance (policy number stored masked) | D1 | Yes |
| Maintenance records and line items | D1 | Yes |
| Receipts, documents (bytes) | R2, encrypted on the device before upload | Ciphertext only |
| Document metadata | D1 | Yes |
| Saved routes / trips ("roadmap") | D1 | Yes |
| Recents, saved places (Home, Work) | Device (Hive); optional sync later | Optional |
| TomTom usage counters | D1 / Durable Object (counts only) | Yes |

Privacy rules (Malaysia PDPA applies): consent screen before account creation, account deletion that also deletes R2 objects, an audit log, collect the minimum, mask policy and reference numbers (full values live only inside encrypted documents).

---

## 5. TomTom migration

### 5.1 Terms check (do this first)
Both TomTom and Google restrict storing and caching their results. **VERIFY** in the current TomTom terms and documentation:
- What may be cached, for how long, and what may never be stored.
- Whether search, geocode, routing and tile responses have different rules.
- The free-tier quota. The product owner's figure is "200k requests free"; confirm whether that is per day or per month, and for which APIs (tiles, search and routing are usually counted separately).
- Whether using TomTom data with MapLibre and showing it alongside OpenFreeMap data is allowed.

Until this is confirmed, apply **D7**: cache only for short TTLs allowed by the response headers, and store counts, not payloads.

### 5.2 What the proxy handles

| App need | TomTom API (VERIFY names) | Proxy endpoint |
|---|---|---|
| Place search and autocomplete | Search / Fuzzy Search | `GET /v1/maps/search` |
| Geocode an address | Geocoding | `GET /v1/maps/geocode` |
| Reverse geocode a point | Reverse Geocoding | `GET /v1/maps/reverse` |
| Car / motorcycle / walking route | Routing | `GET /v1/maps/route` |
| Live traffic and incidents | Traffic | `GET /v1/maps/traffic` |
| Map tiles / style | Vector tiles | Direct from the app, see 5.5 |

All requests pass `language=ms-MY` (or the user's locale) where supported.

### 5.3 Cache keys and TTLs (starting values, adjust to the terms)

Normalize before building the key:
- lowercase, trim, collapse spaces;
- expand Malay abbreviations (`jln`→`jalan`, `tmn`→`taman`, `kg`→`kampung`, `lrg`→`lorong`, `bdr`→`bandar`);
- round coordinates to a geohash cell (precision 6 for reverse geocode, 5 for "bias near me" search);
- include `language` and `profile` in the key.

| Request | Key | TTL |
|---|---|---|
| Reverse geocode | `rev:{geohash6}:{lang}` | longest allowed (start at 24 h) |
| Geocode | `geo:{normalized_query}:{lang}` | start at 24 h |
| Search | `srch:{normalized_query}:{geohash5}:{lang}` | start at 10 min |
| Route | `rt:{profile}:{origin_cell}:{dest_cell}:{options}` | 2–5 min |
| Traffic / incidents | not cached, or ≤ 30 s | seconds |

Layers: (1) Cloudflare Cache API or KV at the edge, (2) the device's own Hive cache for recents, (3) D1 counters only. The user's own saved places and confirmed contributions are **our data** and may be stored without restriction.

### 5.4 Quota guard and fallback
- A Durable Object `QuotaGuard` counts requests per provider, endpoint and day.
- At 80 % of the quota: log a warning and stop non-essential calls (autocomplete suggestions).
- At 95 %: serve cache only, then fall back: search → Photon/Nominatim, routing → Valhalla/OSRM, tiles → OpenFreeMap.
- The app already has a provider chain in [composite_routing_service.dart](../lib/data/datasources/remote/routing/composite_routing_service.dart). TomTom is added as one more provider in front of it, via the proxy, so failure falls through to the existing open providers.

### 5.5 Tiles and the API key
Tile requests are high volume. Options, in order of preference (VERIFY what TomTom allows):
1. Use TomTom tiles directly from the app with a **restricted key** (limited by app bundle ID and referrer in the TomTom dashboard), separate from the key the proxy holds.
2. Proxy tiles through a Worker only if the terms allow caching them.
Never ship the unrestricted search/routing key inside the app.

### 5.6 App-side migration steps
1. Add a `MapProvider` adapter interface in the app (search, reverse, route, style URL). Existing Map Lab providers become implementations.
2. Add a `TomTomProvider` that calls the proxy, not TomTom.
3. Keep OpenFreeMap as the fallback and as the default when the proxy is unreachable.
4. Switch the default per environment with a remote config flag so rollout can be reverted without an app release.

---

## 6. Database schema (D1, first draft)

Migrations live in `backend/migrations/` as numbered `.sql` files and are applied with Wrangler. SQLite types are used.

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE,                 -- null for guests
  locale TEXT NOT NULL DEFAULT 'ms-MY',
  created_at INTEGER NOT NULL,
  deleted_at INTEGER
);

CREATE TABLE vehicles (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  plate TEXT NOT NULL,
  nickname TEXT, make TEXT, model TEXT, year INTEGER,
  type TEXT CHECK (type IN ('car','motorcycle','truck','bus','other')),
  odometer_km INTEGER,
  archived_at INTEGER,
  updated_at INTEGER NOT NULL,
  deleted_at INTEGER,
  UNIQUE (user_id, plate)
);

CREATE TABLE roadtax (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  vehicle_id TEXT NOT NULL REFERENCES vehicles(id),
  valid_from INTEGER, valid_to INTEGER NOT NULL,
  amount_sen INTEGER,
  reference_no_masked TEXT,
  updated_at INTEGER NOT NULL, deleted_at INTEGER
);

CREATE TABLE insurance (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  vehicle_id TEXT NOT NULL REFERENCES vehicles(id),
  provider TEXT, cover_type TEXT,
  valid_from INTEGER, valid_to INTEGER NOT NULL,
  premium_sen INTEGER,
  policy_no_masked TEXT,             -- last 4 only
  updated_at INTEGER NOT NULL, deleted_at INTEGER
);

CREATE TABLE maintenance (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  vehicle_id TEXT NOT NULL REFERENCES vehicles(id),
  service_date INTEGER NOT NULL,
  odometer_km INTEGER,
  workshop TEXT, notes TEXT,
  total_sen INTEGER,
  payment_method TEXT,               -- label only, never card numbers
  updated_at INTEGER NOT NULL, deleted_at INTEGER
);

CREATE TABLE maintenance_items (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  maintenance_id TEXT NOT NULL REFERENCES maintenance(id),
  label TEXT NOT NULL, qty REAL, cost_sen INTEGER,
  updated_at INTEGER NOT NULL, deleted_at INTEGER
);

CREATE TABLE documents (             -- metadata only; bytes are in R2
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  vehicle_id TEXT, maintenance_id TEXT,
  category TEXT, expiry INTEGER,
  r2_key TEXT NOT NULL,
  size_bytes INTEGER, mime TEXT,
  updated_at INTEGER NOT NULL, deleted_at INTEGER
);

CREATE TABLE saved_routes (          -- "roadmap"
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  name TEXT,
  stops_json TEXT NOT NULL,
  profile TEXT,
  updated_at INTEGER NOT NULL, deleted_at INTEGER
);

CREATE TABLE provider_usage (        -- counts only, no payloads
  day TEXT NOT NULL, provider TEXT NOT NULL, endpoint TEXT NOT NULL,
  requests INTEGER NOT NULL DEFAULT 0,
  cache_hits INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (day, provider, endpoint)
);

CREATE TABLE audit_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  actor TEXT, action TEXT, entity TEXT, entity_id TEXT, at INTEGER NOT NULL
);

CREATE INDEX idx_vehicles_user     ON vehicles(user_id);
CREATE INDEX idx_roadtax_user_due  ON roadtax(user_id, valid_to);
CREATE INDEX idx_roadtax_due       ON roadtax(valid_to);
CREATE INDEX idx_insurance_due     ON insurance(valid_to);
CREATE INDEX idx_maintenance_veh   ON maintenance(vehicle_id, service_date);
```

Rules for agents:
- **Never** add a column for card numbers, NFC payloads or wallet balances.
- Every query must be scoped by `user_id`. Put this in the data-access layer; do not rely on handlers.
- Deletes are soft (`deleted_at`) so sync can propagate them; a purge job removes them later.
- Spatial needs: use lat/lon plus a geohash column with an index. SpatiaLite is not available on D1.

---

## 7. API overview

Base: `https://api.<domain>/v1`. JSON only. Errors use `{ "error": { "code": "...", "message": "..." } }` with the user's locale for `message`. Write a full OpenAPI file at `docs/backend_api.openapi.yaml` in phase 1 (see [indoor_api.openapi.yaml](indoor_api.openapi.yaml) for the existing style).

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/guest`, `POST /auth/upgrade`, `POST /auth/refresh`, `DELETE /account` |
| Sync | `GET /sync?since={ts}`, `POST /sync` (batch upserts with `updated_at`) |
| Vehicles and records | CRUD for `/vehicles`, `/roadtax`, `/insurance`, `/maintenance` |
| Documents | `POST /documents` (signed R2 upload), `GET /documents/{id}`, `DELETE` |
| Saved routes | CRUD `/routes` |
| Maps proxy | `/maps/search`, `/maps/geocode`, `/maps/reverse`, `/maps/route`, `/maps/traffic` |
| Config | `GET /config` (feature flags, provider defaults, minimum app version) |
| Admin (later) | `/admin/*`, behind Cloudflare Access plus role check |

Sync rule: last write wins per row by `updated_at`; deletes are soft and included in `GET /sync`. This is enough for single-user data. Team and owner sharing (wallet plan) needs a proper conflict design later; do not build it now.

Security baseline: short-lived access token plus refresh token; per-IP and per-user rate limits (KV or Durable Object); request size limits; validate all input with a schema (zod); CORS locked to the admin origin only; secrets only in Wrangler secrets, never in the repo.

---

## 8. Malay (ms-MY) support

- API responses and error messages are localized; default `ms-MY`, fallback `en`.
- Pass `language=ms-MY` to TomTom where supported (VERIFY coverage).
- Search normalization: Malay abbreviations and spelling variants (see 5.3).
- Text is UTF-8 everywhere. Admin portal is bilingual.
- Navigation voice depends on `flutter_tts` having a Malay voice on the device: test on target Android and iOS devices before promising it.
- Malaysian formats: plates (`WXY 1234`), addresses, currency (RM, stored as sen).

---

## 9. Cloudflare setup guide (for humans)

Do this once, in order. Agents may script steps 4–8 but a human owns the account and secrets.

1. **Account.** Create a Cloudflare account, add the domain, enable 2FA. Note the account ID.
2. **Tooling.** Install Node LTS, then `npm i -g wrangler` and run `wrangler login`.
3. **Repo layout.** Create `backend/` in this repo (separate from the Flutter app code):
   ```
   backend/
     wrangler.toml
     package.json
     src/
       index.ts            # Hono app
       routes/             # one file per area
       data/               # data-access layer (only place that touches D1/R2/KV)
       guards/             # QuotaGuard Durable Object, rate limiting
       providers/          # tomtom.ts, openfreemap.ts, valhalla.ts
     migrations/           # 0001_init.sql ...
     test/
   ```
4. **Create resources** (names are suggestions):
   ```
   wrangler d1 create moove-db
   wrangler r2 bucket create moove-docs
   wrangler r2 bucket create moove-packs        # offline packs, later
   wrangler kv namespace create moove-cache
   ```
5. **Bindings** in `wrangler.toml` (paste the IDs from step 4):
   ```toml
   name = "moove-api"
   main = "src/index.ts"
   compatibility_date = "2026-11-01"

   [[d1_databases]]
   binding = "DB"
   database_name = "moove-db"
   database_id = "<from step 4>"

   [[r2_buckets]]
   binding = "DOCS"
   bucket_name = "moove-docs"

   [[kv_namespaces]]
   binding = "CACHE"
   id = "<from step 4>"

   [[durable_objects.bindings]]
   name = "QUOTA"
   class_name = "QuotaGuard"

   [[migrations]]
   tag = "v1"
   new_sqlite_classes = ["QuotaGuard"]

   [triggers]
   crons = ["0 1 * * *"]   # daily: reminders, purge, usage rollup
   ```
6. **Secrets** (never in the repo):
   ```
   wrangler secret put TOMTOM_API_KEY        # server key, unrestricted, proxy only
   wrangler secret put JWT_SECRET
   ```
   The TomTom **tile key** for the app is a separate, restricted key (bundle ID / referrer limited) set in the app's build config, not as a Worker secret.
7. **Migrations.** `wrangler d1 migrations apply moove-db --local` while developing, then `--remote` for production.
8. **Environments.** Use `[env.staging]` and `[env.production]` with separate D1, R2 and KV resources. The app's build flavor points at the matching API domain. Never test against production data.
9. **Admin gate.** Put the admin portal (Pages) and `/admin/*` behind **Cloudflare Access** with an allow-list of staff emails.
10. **Alerts.** Enable Workers Logs and set notifications for error rate, and a daily usage check. The first real limit you hit will be requests per day on the free plan: expect to move to the **Workers Paid plan (about $5/month)** before or at public launch (VERIFY current price and limits).

Free-tier figures to re-check before launch (from memory): Workers requests per day and CPU time, D1 storage and daily row reads/writes, R2 storage and operation counts, KV daily operations.

---

## 10. Delivery phases (strict checkpoints)

Work **one phase at a time**. At the end of each phase: stop, summarize what was built, give verification steps, and wait for approval. Do not start the next phase early.

### Phase 0: Decisions and terms (no code)
- Read the TomTom terms and quota (5.1) and record the answers at the top of section 5.
- Create the TomTom keys (server and restricted tile key).
- **Deliverable:** updated section 5.1 with verified facts. **Stop.**

### Phase 1: Foundation
- `backend/` scaffold, Hono app, health endpoint, `/config`.
- D1 migration `0001_init.sql` from section 6, data-access layer, schema validation.
- Guest auth, token issue and refresh, rate limiting, audit log, account deletion.
- OpenAPI file, staging environment, CI that runs tests and `wrangler deploy` to staging.
- **Deliverable:** staging API with auth and tests passing. **Stop.**

### Phase 2: Vehicle data and sync
- CRUD and `/sync` for vehicles, roadtax, insurance, maintenance, saved routes.
- Document upload to R2 (signed URLs), metadata in D1.
- Flutter: a sync repository that uses Hive as the local source of truth and pushes/pulls through the API. NFC and wallet code stays untouched.
- Daily cron: renewal reminders list, purge of soft-deleted rows.
- **Deliverable:** two devices sync the same vehicle data. **Stop.**

### Phase 3: TomTom proxy and quota guard
- Provider adapters, cache layers, normalization, `QuotaGuard`, fallbacks (section 5).
- Flutter `MapProvider` adapter and TomTom provider behind a remote flag; Malay language passed through.
- **Deliverable:** search, reverse geocode and routing via TomTom with cache hit-rate and quota counters visible. Fallback proven by forcing the quota to zero. **Stop.**

### Phase 4: Admin portal v1
- Pages app behind Cloudflare Access. Bilingual.
- Screens: users (limited view), usage and quota dashboard, config flags, audit log.
- **Deliverable:** staff can see usage and change flags. **Stop.**

### Phase 5: Transit (Malaysia)
- Compute box (or free ARM instance) for GTFS ingestion; outputs bundled to R2.
- `GET /transit/nearby` using geohash-indexed stops. Respect `data.gov.my` rate limits (4 requests/min noted in [PRODUCT_AND_3D_PLAN.md](PRODUCT_AND_3D_PLAN.md)); cache centrally.
- **Deliverable:** nearby stops and arrivals for Malaysia. **Stop.**

### Phase 6: Routing engine, offline packs, contributions
- Valhalla on the box (and OTP only if the multimodal planner is approved), offline pack builds to R2.
- Road reports and place contributions with votes, trust score and moderation queue in the admin portal.
- **Deliverable:** per feature, as approved. **Stop.**

### Phase 7: Indoor navigation
- Follow [INDOOR_NAVIGATION_BACKEND.md](INDOOR_NAVIGATION_BACKEND.md): IMDF venue packages in R2, routing graph on the device.

---

## 11. Growth plan

| Trigger | Action |
|---|---|
| Free-plan request or D1 limits hit | Move to Workers Paid. |
| One D1 database nears its size cap or write rate | Shard by user (one D1 per cohort) or move append-only data (logs, usage) to R2 or Analytics Engine. This is why every row has `user_id` and UUID ids. |
| Admin analytics or spatial queries get slow or awkward | Add a managed Postgres (Supabase or Neon) for reporting and spatial tables. D1 stays for per-user data. |
| TomTom free quota runs out | Tighten caching within the terms, raise fallback share, then buy a paid TomTom plan. This is the first real cost to expect. |
| Routing box saturated | Add replicas behind a load balancer and cache common routes at the edge. |

---

## 12. Definition of done (every phase)

- Tests pass locally and in CI; new endpoints are in the OpenAPI file.
- Works on staging against staging resources only.
- No secrets in the repo; no card, NFC or wallet data anywhere on the server.
- Every query is `user_id`-scoped; the data-access layer is the only code touching bindings.
- Malay messages exist for every user-facing error.
- A short summary and verification steps are written for the human reviewer.

---

## 13. Open questions

1. TomTom free-tier size and period (per day or month) and which APIs it covers. (Phase 0)
2. What TomTom allows to be cached, and for how long. (Phase 0)
3. "Pintaz" and "krakenDev" from the original notes are not defined. If krakenDev provides open-source map data, record what it provides (tiles, geocoding, routing) and add it as another provider behind the same adapter.
4. Is "roadmap" in this plan correct as saved routes and trips (assumed here)?
5. Do reminders need server push, or are local notifications enough? (Affects Phase 2.)
6. Will the compute box be a paid VPS or a free ARM instance? (Affects Phase 5.)
