# System Architecture & Data Flow

## Ek Zila – Ek Yuva Kaushal Network

---

## 1. Architecture at a Glance

**Style:** Modular monolith API (FastAPI) + SPA frontend (React), with Redis as a performance/state layer and Cloudflare as the edge. This is the right MVP trade-off: simple to deploy and debug, and easy to split into services later.

| Layer | Technology | Role |
|-------|-----------|------|
| Edge | **Cloudflare** (DNS, CDN, WAF, SSL, Turnstile, Pages) | Routing, TLS, DDoS/bot protection, static asset delivery |
| Frontend | **React + Vite + Tailwind CSS** | SPA/PWA, bilingual UI |
| API | **FastAPI** (async, Uvicorn/Gunicorn) | REST APIs, auth, business logic |
| Primary DB | **PostgreSQL (Supabase)** | System of record |
| Cache/State | **Upstash Redis** | Sessions, OTP, rate limits, district stats/skill cache |
| Background | FastAPI BackgroundTasks → later ARQ/Celery | Stats refresh, notifications |
| Observability | Sentry, structured logs, UptimeRobot/Better Stack | Errors, uptime |

---

## 2. High-Level Diagram

```text
                          ┌──────────────────────────────┐
                          │   Users (Mobile / Desktop)   │
                          │ Youth • Provider • MSME • Admin
                          └──────────────┬───────────────┘
                                         │ HTTPS (TLS 1.2+)
                                         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE (Edge Layer)                        │
│  DNS • Proxy (orange cloud) • SSL/TLS Full(Strict) • WAF • Bot Mgmt    │
│  Rate-limit rules • Turnstile • Cache Rules • HSTS                     │
│                                                                        │
│   ekzila.in / www ─────────► Cloudflare Pages (React static build)     │
│   api.ekzila.in  ──────────► Origin: FastAPI (via Proxy / Tunnel)      │
└───────────────┬───────────────────────────────┬────────────────────────┘
                │ static assets (CDN cache)     │ /api/v1/* (dynamic)
                ▼                               ▼
     ┌─────────────────────┐        ┌──────────────────────────────────┐
     │  React SPA (Vite)   │        │      FastAPI Backend (Async)     │
     │  Tailwind • i18n    │  JSON  │ ┌──────────────────────────────┐ │
     │  React Query • PWA  │◄──────►│ │ Middleware: CORS, RateLimit, │ │
     └─────────────────────┘ (JWT)  │ │ AuthN/Z, Request-ID, Logging │ │
                                    │ ├──────────────────────────────┤ │
                                    │ │ Routers → Services → Repos   │ │
                                    │ └───────┬──────────────┬───────┘ │
                                    └─────────┼──────────────┼─────────┘
                       cache/session/limits   │              │  SQL (asyncpg)
                                              ▼              ▼
                                  ┌──────────────────┐  ┌─────────────────────┐
                                  │  Upstash Redis   │  │ PostgreSQL/Supabase │
                                  │ sessions • OTP   │  │ (PgBouncer pooled)  │
                                  │ rate-limit • stats│ │ districts, users,   │
                                  │ skill caches     │  │ providers, jobs ... │
                                  └──────────────────┘  └─────────────────────┘
                                              ▲
                                  ┌───────────┴───────────┐
                                  │ External: SMS/OTP API │
                                  │ Sentry • Email         │
                                  └───────────────────────┘
```

### Mermaid version (renders in GitHub/Notion/Obsidian)

```mermaid
flowchart LR
  U[Users] -->|HTTPS| CF[Cloudflare: DNS, WAF, CDN, SSL]
  CF -->|static| PG[Cloudflare Pages - React SPA]
  CF -->|/api/v1| API[FastAPI Backend]
  PG -. XHR/fetch JWT .-> CF
  API <--> R[(Upstash Redis)]
  API <--> DB[(PostgreSQL / Supabase)]
  API --> SMS[SMS / OTP Provider]
  API --> SEN[Sentry / Logs]
```

---

## 3. Component Responsibilities

### 3.1 Cloudflare (Edge)
| Feature | Configuration |
|---------|---------------|
| **DNS** | `ekzila.in` → Pages; `api.ekzila.in` → backend origin (proxied). |
| **SSL/TLS** | Mode **Full (Strict)** + Origin CA certificate on API host; Always-HTTPS; min TLS 1.2; HSTS enabled. |
| **CDN** | Cache static assets (`/assets/*`) with long TTL + hashed filenames. Cache public GET endpoints selectively via Cache Rules (e.g., `/api/v1/districts`, 1 h) respecting `Cache-Control` from API. |
| **WAF** | Managed rules (OWASP), block known bad bots, geo-restrict admin routes if desired. |
| **Rate limiting** | Edge rule: e.g., 100 req/min/IP on `/api/v1/*`; stricter (5/min) on `/auth/otp/*`. (Redis does fine-grained per-user/phone limits.) |
| **Turnstile** | Required on OTP request and registration forms. |
| **Origin protection** | Allow only Cloudflare IPs at origin, or use **Cloudflare Tunnel** (no public origin IP). |
| **Headers** | CSP, X-Frame-Options, Referrer-Policy via Transform Rules. |

### 3.2 React Frontend
- **Build:** Vite → static files on Cloudflare Pages (global CDN).
- **State/data:** TanStack Query (server cache, retries, stale-while-revalidate), Zustand (UI/auth state).
- **Routing:** React Router with role-guarded routes.
- **i18n:** `react-i18next` (hi/en).
- **PWA:** Service worker caches shell + district data for flaky networks.
- **Auth:** Access token held in memory; refresh token in `HttpOnly; Secure; SameSite=Lax` cookie.

### 3.3 FastAPI Backend
```text
backend/
├── app/
│   ├── main.py                # app factory, middleware, routers
│   ├── core/                  # config, security, logging, redis client
│   ├── api/v1/                # routers: auth, districts, skills, providers,
│   │                          #   opportunities, applications, admin
│   ├── models/                # SQLAlchemy 2.0 models
│   ├── schemas/               # Pydantic v2 schemas
│   ├── services/              # business logic (cache-aware)
│   ├── repositories/          # DB queries
│   ├── deps.py                # dependencies (db session, current_user, RBAC)
│   └── tasks/                 # background jobs
├── alembic/                   # migrations
├── tests/
└── Dockerfile
```
- **Async stack:** SQLAlchemy 2.0 async + `asyncpg`, Pydantic v2, `httpx` for outbound calls.
- **Layering:** Router (HTTP) → Service (rules + cache) → Repository (SQL). Keeps cache logic out of routers.
- **Auth:** JWT access token (15 min) + rotating refresh token (7 days) tracked in Redis.
- **Docs:** OpenAPI at `/docs` (disable or protect in production).

### 3.4 Upstash Redis
| Use case | Structure | TTL |
|----------|-----------|-----|
| Sessions / refresh tokens | String/Hash `session:{user_id}:{jti}` | 7 d |
| OTP | String `otp:{phone}` + counter `otp:try:{phone}` | 5 min |
| Rate limiting | Sliding window / token bucket `rl:{scope}:{id}` | window |
| District stats | Hash `cache:district:{id}:stats` | 10 min |
| Skill demand ranking | Sorted set `cache:district:{id}:skill_demand` | 15 min |
| Directory/search pages | String (JSON) `cache:providers:{district}:{skill}:{page}` | 15 min |
| Static-ish lists | String `cache:districts:{state}` | 24 h |
| Cache versioning | Integer `ver:district:{id}` | none |

> Connection note: Upstash offers an HTTP/REST client (`upstash-redis`) and standard TLS Redis protocol. Use the **REST client** if you deploy the API on serverless/edge-like hosts; use `redis-py` (async) over TLS on a long-running container for lower per-command latency. Upstash bills per command, so batch with pipelines and avoid chatty patterns.

### 3.5 PostgreSQL (Supabase)
- Primary store; use the **pooled connection string (port 6543, PgBouncer transaction mode)** for the API. With asyncpg set `statement_cache_size=0`.
- Use direct connection (5432) only for migrations.
- Indexes on `(district_id, skill_category_id, status)`, trigram/FTS on titles.
- **Row-level scoping is enforced in the API** (the API connects with a service role; do not expose the DB to the browser).

---

## 4. Key Data Flows

### 4.1 Cached Read – Student opens a District Hub
```text
React ──GET /api/v1/districts/{id}/stats──► Cloudflare (WAF, rate limit)
   ──► FastAPI ──► Redis HGETALL cache:district:{id}:stats
        ├─ HIT  → return JSON (≈ 5–20 ms) + Cache-Control: public, max-age=60
        └─ MISS → acquire lock `lock:district:{id}:stats` (SET NX EX 10)
                  → run aggregate SQL on Postgres
                  → HSET + EXPIRE (600 s) → return JSON
```
*Cache-stampede protection:* short lock + stale-while-revalidate (serve stale value while one worker refreshes).

### 4.2 Login with OTP
```text
1. Client → POST /auth/otp/request {phone, turnstile_token}
2. Edge: Turnstile verified, IP rate limit
3. API: Redis rate limit (phone: 3/10 min, IP: 10/hour) → generate 6-digit OTP
        → store HASH(otp) in Redis (5 min) → call SMS provider
4. Client → POST /auth/otp/verify {phone, otp}
5. API: compare hash, max 5 tries → upsert user → issue access JWT
        + refresh token (cookie) → store session in Redis
6. Client stores access token in memory; refresh via POST /auth/refresh
```

### 4.3 Write with Cache Invalidation – Employer posts an opportunity
```text
POST /opportunities (JWT, role=employer, verified)
 → validate → INSERT (Postgres, transaction)
 → INCR ver:district:{id}            (invalidates versioned list caches)
 → DEL cache:district:{id}:stats
 → ZINCRBY cache:district:{id}:skill_demand 1 skill:{sid}   (optional)
 → 201 Created
```

### 4.4 Admin Analytics
```text
GET /admin/districts/{id}/funnel
 → RBAC check (admin of that district) → Redis cache (5 min)
 → MISS: SQL aggregates (registered → trained → applied → placed)
```

---

## 5. Caching Strategy

| Pattern | Applied to |
|---------|-----------|
| **Cache-aside** | Directory lists, district stats |
| **Write-through counters** | Live counters (page views, applications today) via `HINCRBY`, flushed to Postgres by a periodic job |
| **Versioned keys** | Paginated/filtered lists: key includes `ver:district:{id}`; bump version on any write instead of scanning/deleting keys |
| **TTL jitter** | Add ±10% random TTL to avoid synchronized expiry |
| **Negative caching** | Cache "not found" for 30 s to protect DB |
| **Graceful degradation** | If Redis is unavailable, log + bypass to DB; rate limiting fails *open* for reads but *closed* for OTP endpoints |

**Never cache:** per-user private data in shared keys, OTP plaintext, anything role-scoped without the role/user in the key.

---

## 6. Security Architecture

| Area | Controls |
|------|----------|
| **Transport** | TLS everywhere, HSTS, Full (Strict) SSL, Origin CA. |
| **AuthN** | OTP login, short-lived JWT (15 min), refresh rotation with reuse detection (revoke family in Redis). |
| **AuthZ** | RBAC dependency (`require_role`) + district scoping (`admin.district_id == resource.district_id`). |
| **Input** | Pydantic validation, parameterized SQL (ORM), size limits, file-type checks for uploads. |
| **Abuse** | Turnstile, Cloudflare rate limit, Redis per-identity limits, OTP attempt caps. |
| **CORS** | Allow only `https://ekzila.in` (and preview domain in staging). |
| **Secrets** | Environment variables / platform secret store; never in repo; rotate keys. |
| **Data** | Encryption at rest (Supabase), minimal PII, DPDP-style consent, soft delete + purge job. |
| **Audit** | `audit_logs` table for verification, role change, and status updates. |
| **Headers** | CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`. |

---

## 7. Deployment Topology

| Environment | Frontend | Backend | DB | Redis |
|-------------|----------|---------|----|-------|
| **Local** | Vite dev (5173) | Uvicorn `--reload` (8000) | Docker Postgres / Supabase dev | Upstash dev DB |
| **Staging** | `staging.ekzila.in` (Pages preview) | Container on Render/Railway/Fly.io | Supabase staging | Upstash staging |
| **Production** | `ekzila.in` (Pages) | Container (≥2 instances), health-checked | Supabase prod (daily backups) | Upstash prod (TLS, regional close to API) |

- **CI/CD:** GitHub Actions → lint + tests → build Docker → deploy API; Pages auto-builds from `main`.
- **Region placement:** Choose Supabase, API host and Upstash in the **same region closest to India** (e.g., Mumbai/Singapore) to minimize latency.
- **Health endpoints:** `/healthz` (liveness), `/readyz` (checks DB + Redis).

> FastAPI should run as a normal container/ASGI service (Gunicorn + Uvicorn workers), not on Cloudflare Workers. Cloudflare fronts it as proxy/CDN/WAF.

---

## 8. Scalability & Performance Plan

1. **Stateless API** → scale horizontally behind a load balancer.
2. **DB pooling** via PgBouncer; keep async pool small (e.g., 10–20 per instance).
3. **Read-heavy optimization:** Redis for district hub, directories; Cloudflare edge cache for public endpoints.
4. **Pagination:** keyset (cursor) pagination for large lists.
5. **Indexes & materialized views** for admin funnel when data grows.
6. **Background workers** for notifications and stat refresh (ARQ with Redis).
7. **Future split:** Notification service, analytics service, and search (Meilisearch/pg_trgm → OpenSearch) if needed.

---

## 9. Observability

| Concern | Tooling |
|---------|---------|
| Errors | Sentry (frontend + backend) |
| Logs | JSON logs with `request_id`, `user_id`, `district_id` |
| Metrics | Prometheus-compatible `/metrics` or platform metrics: latency, error rate, cache hit ratio |
| Uptime | UptimeRobot / Better Stack on `/healthz` |
| Alerts | P95 latency > 800 ms, 5xx > 2%, Redis errors, DB pool exhaustion |

---

## 10. Environment Variables (Reference)

```bash
# Backend
APP_ENV=production
DATABASE_URL=postgresql+asyncpg://user:pass@host:6543/postgres
UPSTASH_REDIS_URL=rediss://default:***@host:6379     # or REST url/token
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
JWT_SECRET=...
JWT_ACCESS_MINUTES=15
JWT_REFRESH_DAYS=7
SMS_API_KEY=...
TURNSTILE_SECRET=...
CORS_ORIGINS=https://ekzila.in
SENTRY_DSN=...

# Frontend
VITE_API_BASE_URL=https://api.ekzila.in/api/v1
VITE_TURNSTILE_SITE_KEY=...
```

---

## 11. Architecture Decision Records (Summary)

| ADR | Decision | Reason |
|-----|----------|--------|
| 1 | Modular monolith over microservices | Small team, faster delivery, simpler ops |
| 2 | Postgres as source of truth | Relational data, aggregations, strong constraints |
| 3 | Redis only for ephemeral data | Safe to lose; DB remains authoritative |
| 4 | Refresh token in HttpOnly cookie | Mitigates XSS token theft |
| 5 | Versioned cache keys | Simple, scan-free invalidation |
| 6 | Cloudflare Pages for SPA | Free global CDN, preview deployments |
