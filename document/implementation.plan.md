# Implementation Plan: Step-by-Step Development Roadmap

## Ek Zila – Ek Yuva Kaushal Network

**Timeline:** ~8 weeks (42 working days) for a solo developer or a small team of 2. Each "Day" is a focused 4–6 hour block of work. Adjust freely; the order matters more than the dates.

| Phase | Focus | Days | Output |
|-------|-------|------|--------|
| 0 | Setup & foundations | 1–2 | Repos, accounts, tooling |
| 1 | FastAPI + PostgreSQL/Supabase | 3–16 | Working REST API with auth and all core modules |
| 2 | Upstash Redis integration | 17–22 | Sessions, OTP, rate limits, caching |
| 3 | React frontend (Tailwind) | 23–38 | Complete bilingual UI for all roles |
| 4 | Cloudflare deployment & SSL | 39–46 | Live, secured production site |
| 5 | Pilot launch & hardening | 47+ | Real users, monitoring, iteration |

---

## Phase 0 – Setup & Foundations (Days 1–2)

### Day 1: Accounts & repositories
- [ ] Create accounts: GitHub, Supabase, Upstash, Cloudflare, Sentry, SMS provider (MSG91/Twilio test).
- [ ] Buy/point the domain (e.g., `ekzila.in`) and add it to Cloudflare (change nameservers).
- [ ] Create a monorepo:
```text
ek-zila/
├── backend/    # FastAPI
├── frontend/   # React + Vite
├── docs/       # the 5 .md documents
└── .github/workflows/
```
- [ ] Copy the 5 documents into `docs/`.
- [ ] Add `.gitignore`, `README.md`, and a branching rule (`main` = production, `dev` = integration, feature branches).

### Day 2: Tooling
- [ ] Install Python 3.12, Node 20+, Docker, `uv` or `poetry`.
- [ ] Set up `pre-commit` (ruff, black, mypy for Python; eslint + prettier for JS).
- [ ] Create `.env.example` using the variable list in `architecture.md`.
- [ ] Decide on seed data source: list of districts (LGD codes) + initial skill taxonomy CSV.

**Done when:** Both folders exist, linters run, and every service account is ready.

---

## Phase 1 – FastAPI + PostgreSQL/Supabase (Days 3–16)

### Day 3: FastAPI skeleton
- [ ] Create the `app/` structure (`core`, `api/v1`, `models`, `schemas`, `services`, `repositories`).
- [ ] `main.py` with app factory, CORS, request-ID middleware, JSON logging.
- [ ] Add `/healthz` and `/readyz`.
- [ ] Settings via `pydantic-settings` reading `.env`.
- [ ] Dockerfile + `docker-compose.yml` (API + local Postgres).

### Day 4: Database connection & migrations
- [ ] Create Supabase project (region nearest India). Copy the pooled (6543) and direct (5432) strings.
- [ ] Configure SQLAlchemy 2.0 async engine with `asyncpg` (`statement_cache_size=0` for pooled connection).
- [ ] Set up Alembic (use direct connection for migrations).
- [ ] Write a `get_db` dependency (async session per request).

### Day 5: Core models – part 1
- [ ] Create enums and models: `District`, `SkillCategory`, `User`, `UserSkill`.
- [ ] First Alembic migration; apply to Supabase.
- [ ] Add indexes from `backend schema.md`.

### Day 6: Core models – part 2
- [ ] `TrainingProvider`, `Course`, `Enrollment`.
- [ ] `Employer`, `EmploymentOpportunity`, `Application`, `AuditLog`.
- [ ] Second migration; verify constraints (unique keys, checks).

### Day 7: Seed data
- [ ] Script `scripts/seed.py`: districts for the pilot state, skill taxonomy (sector → skill, Hindi + English), demo providers/employers.
- [ ] Make the seed idempotent (upsert by `slug`/`lgd_code`).
- [ ] Verify in Supabase table editor.

### Day 8: Authentication (OTP, in-memory first)
- [ ] `POST /auth/otp/request` and `/verify` with a **mock SMS provider** (prints OTP to logs in dev).
- [ ] JWT utilities (access 15 min); user upsert on first login.
- [ ] `get_current_user` and `require_role` dependencies.
- [ ] Store OTP temporarily in a DB table or in-process dict *(will move to Redis in Phase 2)*.

### Day 9: Users & profile
- [ ] `GET/PATCH /users/me`, `PUT /users/me/skills`.
- [ ] Profile completeness calculator (0–100).
- [ ] Pydantic schemas with phone/DOB validation.

### Day 10: Districts & skills APIs
- [ ] `GET /districts`, `/districts/{id}`, `/skills/categories` (tree builder).
- [ ] Basic `GET /districts/{id}/stats` (direct SQL aggregates for now).
- [ ] `GET /districts/{id}/skills/demand`.

### Day 11: Training providers & courses
- [ ] `POST/PATCH /providers`, `GET /providers` (filters: district, skill, scheme, block, pagination), `GET /providers/{id}`.
- [ ] `POST /providers/{id}/courses`.
- [ ] Ownership checks (provider can edit only their own).

### Day 12: Enrollments
- [ ] `POST /courses/{id}/enroll` (eligibility, seat check, duplicate → 409).
- [ ] `GET /users/me/enrollments`, `PATCH /enrollments/{id}` (provider side).
- [ ] State transition validation.

### Day 13: Employers & opportunities
- [ ] `POST /employers`; verified-only posting guard.
- [ ] CRUD for `/opportunities` with filters (district, skill, type, salary, pagination).
- [ ] Nightly-expiry function (deadline → `expired`).

### Day 14: Applications
- [ ] `POST /opportunities/{id}/apply` (unique per user).
- [ ] `GET /opportunities/{id}/applications` (owner only), `PATCH /applications/{id}`.
- [ ] Notification stubs (DB `notifications` table or log).

### Day 15: Admin module
- [ ] Verification queue + decision endpoints; write `audit_logs`.
- [ ] `GET /admin/districts/{id}/funnel` and `/gap` (SQL aggregates, CTEs).
- [ ] District scoping helper (`assert_district_scope`).
- [ ] CSV export endpoint (streaming).

### Day 16: Testing & API hardening
- [ ] `pytest` + `httpx.AsyncClient`; test DB fixtures.
- [ ] Tests: auth flow, RBAC, enrollment rules, application uniqueness, admin scoping.
- [ ] Standardize error envelope; global exception handlers.
- [ ] Check OpenAPI docs; aim for ≥ 70% coverage on services.
- [ ] Add GitHub Actions: lint + tests on PR.

**Phase 1 Done when:** All endpoints in `backend schema.md` work locally against Supabase, tests pass in CI.

---

## Phase 2 – Upstash Redis Integration (Days 17–22)

### Day 17: Redis client & helpers
- [ ] Create Upstash Redis database (same/nearby region); copy URL + token.
- [ ] Choose client: `redis.asyncio` over TLS (long-running container) **or** `upstash-redis` REST client (serverless hosts).
- [ ] `core/redis.py`: singleton client, `ping` check added to `/readyz`.
- [ ] `cache_get/cache_set` helpers with JSON, TTL jitter, and safe fallback (log + bypass on failure).

### Day 18: OTP & sessions in Redis
- [ ] Move OTP storage to `otp:{phone}` (hashed, 5 min) and `otp:try:{phone}`.
- [ ] Implement refresh-token sessions: `session:{user_id}:{jti}`, rotation, family revocation on reuse.
- [ ] `/auth/refresh` and `/auth/logout` endpoints using HttpOnly cookie.
- [ ] Integrate real SMS provider (keep mock in dev).

### Day 19: Rate limiting
- [ ] Sliding-window limiter: OTP per phone (3/10 min), per IP (10/hour).
- [ ] API limiter per user/IP (120/min) as middleware/dependency.
- [ ] Return `429` with `retry_after`; add tests with fake Redis (`fakeredis`).
- [ ] Add Cloudflare Turnstile server-side verification to OTP request.

### Day 20: District stats & skill caches
- [ ] Cache `cache:district:{id}:stats` (hash) with lock + jitter.
- [ ] `skill_demand` / `skill_supply` sorted sets; rebuild function from SQL.
- [ ] Cache skill tree and district lists (24 h).

### Day 21: Versioned list caches & invalidation
- [ ] Add `ver:district:{id}` counters; versioned keys for provider and opportunity lists.
- [ ] Hook invalidation into create/update/verify/enroll/apply/hire flows (service layer only).
- [ ] Add `Cache-Control` headers for public GETs (e.g., `public, max-age=60, stale-while-revalidate=300`).

### Day 22: Verify performance & resilience
- [ ] Load test with `k6`/`locust`: district hub, provider search, job search (compare cached vs. uncached).
- [ ] Log hit/miss; expose cache hit ratio metric.
- [ ] Chaos check: stop Redis → confirm API degrades to DB and OTP endpoints fail closed.
- [ ] Review Upstash command usage; batch with pipelines where chatty.

**Phase 2 Done when:** P95 < 300 ms on cached reads, OTP/session/rate limits fully Redis-backed, invalidation tests pass.

---

## Phase 3 – React Frontend UI (Days 23–38)

### Day 23: Project setup
- [ ] `npm create vite@latest frontend -- --template react-ts`.
- [ ] Install: `tailwindcss`, `react-router-dom`, `@tanstack/react-query`, `axios`, `zustand`, `react-i18next`, `react-hook-form`, `zod`, `recharts`, `lucide-react`.
- [ ] Tailwind config: brand colors, Devanagari-friendly font (Noto Sans Devanagari), mobile-first breakpoints.
- [ ] Folder structure: `src/{api,components,features,hooks,pages,routes,store,i18n,utils}`.

### Day 24: Design system & layout
- [ ] Reusable components: Button, Input, Select, Card, Badge, Modal, Skeleton, EmptyState, Toast, Pagination.
- [ ] App shell: header with language toggle + district chip, mobile bottom nav, footer.
- [ ] i18n setup with `hi.json` / `en.json` and key conventions.

### Day 25: API layer & auth state
- [ ] Axios instance with base URL, access token in memory, 401 → silent refresh → retry.
- [ ] React Query provider and typed API hooks (`useDistricts`, `useProviders`, ...).
- [ ] Auth store (user, role); `ProtectedRoute` and role guards.

### Day 26: Login & onboarding
- [ ] Phone input + Turnstile widget; OTP screen with resend timer.
- [ ] Role selection, district, consent checkbox.
- [ ] Error states for rate limits and invalid OTP.

### Day 27: Landing & district picker
- [ ] Landing hero with state → district searchable selector.
- [ ] Persist selection; redirect to `/d/:slug`.

### Day 28: District Hub
- [ ] KPI cards, top in-demand skills chart, latest openings, upcoming batches.
- [ ] Skeleton loading + stale-while-revalidate behavior via React Query.

### Day 29: Skill explorer & provider search
- [ ] Skill tree/chips with Hindi/English names.
- [ ] Provider list with filters (skill, scheme, block, fee) and URL-synced query params.
- [ ] Provider card with verified badge.

### Day 30: Provider detail & enrollment
- [ ] Provider page: courses, batches, contact, map link.
- [ ] "Request Enrollment" flow with login redirect and profile-completion prompt.
- [ ] Success and error toasts.

### Day 31: Youth dashboard
- [ ] Profile editor with completeness meter and skills multi-select.
- [ ] My Enrollments and My Applications with status badges.

### Day 32: Jobs & applications
- [ ] Jobs list with filters (skill, type, salary, education).
- [ ] Job detail with one-tap Apply; duplicate state handling.

### Day 33: Employer module
- [ ] Business registration form + pending-verification banner.
- [ ] Post Requirement form (react-hook-form + zod), preview step.
- [ ] My postings list with status actions (close/fill).

### Day 34: Employer applicants
- [ ] Applicant table/cards with skills, education, training completed.
- [ ] Status actions: Shortlist / Reject / Hire; optimistic updates.

### Day 35: Provider module
- [ ] Institute registration; course CRUD with batches and seats.
- [ ] Enrollment requests list with approve / mark completed actions.

### Day 36: Admin dashboard
- [ ] KPI cards, funnel chart, skill demand-vs-supply chart (Recharts).
- [ ] Date/block/skill filters; CSV export button.

### Day 37: Admin verification & polish
- [ ] Verification queue with detail drawer, approve/reject with remark.
- [ ] Flagged-items widget; audit trail view (basic).

### Day 38: Quality pass
- [ ] PWA setup (manifest, service worker via `vite-plugin-pwa`); offline shell and cached district data.
- [ ] Accessibility: labels, focus states, contrast, tap targets ≥ 44 px.
- [ ] Performance: route-level code splitting, image optimization, Lighthouse ≥ 90 mobile.
- [ ] Component/unit tests (Vitest + Testing Library) for auth and forms; one Playwright E2E per role flow.

**Phase 3 Done when:** All three flows in `app flow.md` work end-to-end against the staging API in both languages.

---

## Phase 4 – Cloudflare Deployment & SSL (Days 39–46)

### Day 39: Containerize and host the API
- [ ] Production Dockerfile (multi-stage, non-root, Gunicorn with Uvicorn workers).
- [ ] Deploy to Render/Railway/Fly.io in the same region as Supabase/Upstash; run ≥ 2 instances.
- [ ] Set environment variables/secrets; run Alembic migrations in release step.
- [ ] Verify `/healthz` and `/readyz` on the platform URL.

### Day 40: DNS & proxy
- [ ] In Cloudflare DNS: `api` → API host (CNAME, proxied/orange cloud); keep `ekzila.in` for Pages.
- [ ] Confirm nameservers are active and propagation is complete.

### Day 41: SSL/TLS
- [ ] SSL mode **Full (Strict)**.
- [ ] Generate Cloudflare **Origin CA** certificate and install on the API host (or use the platform's valid cert / Cloudflare Tunnel).
- [ ] Enable Always Use HTTPS, Automatic HTTPS Rewrites, min TLS 1.2, HSTS (start with a short max-age).

### Day 42: Frontend on Cloudflare Pages
- [ ] Connect GitHub repo to Pages; build command `npm run build`, output `dist`.
- [ ] Set `VITE_API_BASE_URL=https://api.ekzila.in/api/v1`.
- [ ] Add SPA fallback (`_redirects`: `/* /index.html 200`).
- [ ] Attach custom domain `ekzila.in` and `www`; enable preview deployments for PRs.

### Day 43: Edge security
- [ ] Enable WAF managed rulesets and Bot Fight Mode.
- [ ] Rate-limiting rules: `/api/v1/*` (100/min/IP), `/auth/otp/*` (5/min/IP).
- [ ] Firewall rule to restrict origin to Cloudflare IPs (or move origin behind Cloudflare Tunnel).
- [ ] Transform Rules for security headers (CSP, X-Frame-Options, Referrer-Policy, Permissions-Policy).
- [ ] Set production CORS to `https://ekzila.in` only; disable `/docs` publicly.

### Day 44: Caching rules
- [ ] Cache static assets (`/assets/*`) with long TTL and immutable.
- [ ] Cache Rules for public GET endpoints (`/districts`, `/skills/categories`) honoring origin headers; bypass cache for `/auth`, `/admin`, `/users`.
- [ ] Test cache status via `cf-cache-status` response header.

### Day 45: CI/CD, monitoring & backups
- [ ] GitHub Actions: on merge to `main` → tests → build image → deploy API.
- [ ] Sentry (frontend + backend), uptime monitor on `/healthz`, alert channels (email/Telegram).
- [ ] Verify Supabase daily backups; document restore steps.
- [ ] Write a runbook: deploy, rollback, rotate secrets, clear cache, handle Redis outage.

### Day 46: Go-live checklist
- [ ] Security scan (OWASP ZAP baseline), SSL Labs test (A rating), dependency audit.
- [ ] Load test at expected peak against production-like environment.
- [ ] Privacy policy, terms, and consent text published (Hindi + English).
- [ ] Smoke test all three role flows on the live domain from a mobile device on 4G.

**Phase 4 Done when:** `https://ekzila.in` is live, API on `https://api.ekzila.in` behind Cloudflare with Full (Strict) SSL, WAF/rate limits active, CI/CD deploying on merge.

---

## Phase 5 – Pilot Launch & Hardening (Day 47+)

- [ ] Onboard 2–3 pilot districts: train District Admins, bulk-import providers (CSV), assisted employer onboarding through DIC.
- [ ] Weekly review of KPIs from `prd.document.md` (registrations, enrollments, placements, P95 latency, cache hit ratio).
- [ ] Collect feedback via in-app form; fix top usability issues.
- [ ] Add WhatsApp/SMS notifications, background worker (ARQ) for exports and reminders.
- [ ] Plan v1.1/v2: block-level drill-down, recommendations, certificate verification.

---

## Cross-Cutting Checklists

### Definition of Done (every task)
- [ ] Code reviewed and merged via PR
- [ ] Tests added/updated and passing in CI
- [ ] No secrets in code; env vars documented
- [ ] Docs/OpenAPI updated
- [ ] Works on a mid-range Android phone on 4G

### Risk Watchlist

| Risk | Early signal | Action |
|------|--------------|--------|
| Supabase pooled-connection errors with asyncpg | `prepared statement does not exist` | Set `statement_cache_size=0` or use the direct connection for migrations |
| Upstash command cost spike | Rapid command-count growth | Pipeline calls, raise TTLs, cache more at Cloudflare edge |
| OTP/SMS abuse | Sudden OTP volume | Tighten Redis + Cloudflare limits, enforce Turnstile |
| Scope creep | Features outside MVP list | Park in post-MVP backlog |
| Low employer adoption | Few postings after week 2 | Admin-assisted posting, on-ground drives |

### Suggested Weekly Milestones

| Week | Milestone |
|------|-----------|
| 1 | Setup done; DB models + seed data; auth working |
| 2 | Providers, enrollments, opportunities, applications, admin APIs; tests in CI |
| 3 | Redis: OTP, sessions, rate limits, caches; load-tested |
| 4–5 | Core UI (auth, hub, search, enrollment, jobs) |
| 6 | Employer, provider, admin UIs; PWA + accessibility |
| 7 | Cloudflare deployment, SSL, WAF, CI/CD, monitoring |
| 8 | Hardening, go-live checklist, pilot launch |
