# Product Requirements Document (PRD)

## Ek Zila – Ek Yuva Kaushal Network

**Version:** 1.0 (MVP scope) | **Status:** Draft for build | **Owner:** Product

---

## 1. Vision & Summary

**Vision:** Every district becomes a self-aware skill-and-employment ecosystem, where each young person can see which skills are in demand *in their own district*, where to train for them, and which local employers are hiring.

**One-line pitch:** A district-wise network that connects Youth ↔ Training Providers ↔ Local Employers (MSMEs) ↔ District Administration on a single data-driven platform.

**Principle:** *Hyper-local first.* Every screen, search and statistic is scoped by district by default.

---

## 2. Problem Statement

| # | Problem | Impact |
|---|---------|--------|
| 1 | **Skill gap** – Training programs are not aligned with what local industries actually need. | Youth are trained but unemployable; industries cannot find skilled workers. |
| 2 | **No localized employment pathway** – Job portals are national and urban-heavy; small-town MSME openings are rarely listed. | Youth migrate for work or stay underemployed. |
| 3 | **Fragmented information** – Training centres, schemes and apprenticeships are scattered across offline notices, WhatsApp, and multiple portals. | Low awareness and enrollment. |
| 4 | **No feedback loop for administrators** – Districts lack real-time data on trained vs. placed youth and which skills are in demand. | Poor planning, wasted training budgets. |
| 5 | **Access barriers** – Low bandwidth, language (Hindi/regional), mobile-first users. | Existing portals are hard to use. |

---

## 3. Goals & Non-Goals

### Goals (MVP)
1. District-scoped discovery of skill training and local jobs/apprenticeships.
2. Simple onboarding for youth via mobile number + OTP.
3. Easy posting of requirements by MSMEs.
4. A verified training-provider directory with courses.
5. District admin dashboard showing the skill-to-employment funnel.
6. Fast (P95 < 300 ms cached reads), mobile-first, bilingual (Hindi + English) UI.

### Non-Goals (MVP)
- Payments / paid courses.
- In-platform video learning (LMS).
- Native mobile apps (PWA only).
- AI resume builder / AI matching (post-MVP).

---

## 4. Target Audience & Personas

### 4.1 Youth / Students (Primary)
- **Age:** 16–35; Class 10 pass to graduate; ITI/diploma holders; school/college dropouts.
- **Needs:** "What can I learn near me? Who will give me a job after?"
- **Constraints:** Mostly mobile, 4G/low bandwidth, Hindi preferred.
- **Jobs-to-be-done:** Find training, track enrollment, apply to local openings.

### 4.2 District Administrators (Governance)
- **Who:** District Skill Development Officer, District Magistrate's office, DIC (District Industries Centre), NIC staff.
- **Needs:** Real-time view of registered youth, trained, placed, and skill demand-supply gaps.
- **Jobs-to-be-done:** Approve providers/employers, monitor KPIs, export reports.

### 4.3 Local Industries / MSMEs
- **Who:** Factories, shops, workshops, agro-processing units, service businesses, contractors.
- **Needs:** Find locally-trained, reliable workers fast; low-friction posting.
- **Jobs-to-be-done:** Post job/apprenticeship, review applicants, mark hires.

### 4.4 Training Providers
- **Who:** ITIs, PMKVY/Skill India training partners, polytechnics, NGOs, private institutes.
- **Needs:** Visibility, student leads, proof of placement outcomes.
- **Jobs-to-be-done:** List courses, manage batches, report enrollments/completions/placements.

---

## 5. User Roles & Permissions (RBAC)

| Capability | Youth | Provider | Employer | District Admin | Super Admin |
|-----------|:----:|:--------:|:--------:|:--------------:|:-----------:|
| Browse districts/skills/providers/jobs | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own profile & skills | ✅ | ✅ | ✅ | ✅ | ✅ |
| Enroll in course / apply to job | ✅ | – | – | – | – |
| Manage courses & batches | – | ✅ (own) | – | ✅ | ✅ |
| Post/manage opportunities | – | – | ✅ (own) | ✅ | ✅ |
| Approve/verify provider & employer | – | – | – | ✅ (own district) | ✅ |
| View district analytics | – | limited | limited | ✅ (own district) | ✅ (all) |
| Manage districts/skill taxonomy | – | – | – | – | ✅ |

---

## 6. Core Features

### 6.1 MVP (Must Have)

**F1 – District Selector & Hub**
- Select state → district (auto-suggest, remembered via cookie/profile).
- District Hub page: top in-demand skills, upcoming training, latest openings, key stats.

**F2 – Authentication & Profile**
- Mobile OTP login (no password). Role chosen at signup.
- Youth profile: name, age, education, district/block, skills, preferred sectors, language.
- Profile completeness meter.

**F3 – Skill Taxonomy**
- Hierarchical skill categories (Sector → Skill), aligned to NSQF/Sector Skill Council naming where possible.
- Hindi + English names.

**F4 – Training Provider Directory**
- Search/filter by district, block, skill, scheme (PMKVY, DDU-GKY, ITI, etc.), mode.
- Provider detail page with courses, batches, contact, location (map link), verified badge.
- "Express interest / Enroll" request.

**F5 – Local Employment Opportunities**
- MSME posting: title, type (job / apprenticeship / internship / gig), skill required, openings, pay range, location, deadline.
- Youth browse with filters; one-tap apply with profile.
- Employer applicant list with status pipeline (Applied → Shortlisted → Hired/Rejected).

**F6 – District Admin Dashboard**
- KPIs: registered youth, profile completion, enrollments, completions, placements, open vacancies.
- Skill demand vs. supply chart.
- Provider and employer verification queue.
- CSV export.

**F7 – Notifications**
- SMS/WhatsApp-ready hooks (OTP, application updates). In-app notification list.

### 6.2 Post-MVP (Should/Could Have)
- Skill-to-job recommendation engine; block/panchayat-level drill-down.
- Certificates verification; employer ratings; WhatsApp bot; offline-first PWA; district comparison leaderboard; Aadhaar/DigiLocker/Skill India Digital (SID) integrations; multi-language (Bhojpuri, Maithili, Magahi, etc.).

---

## 7. Functional Requirements (Selected, Traceable)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | User can register/login via mobile OTP; OTP expires in 5 min, max 5 attempts. | P0 |
| FR-02 | Every user is linked to exactly one primary district. | P0 |
| FR-03 | Youth can search providers by district + skill category. | P0 |
| FR-04 | Employer can create/edit/close opportunities tied to a district & skill. | P0 |
| FR-05 | Youth can apply to an opportunity once; duplicates rejected. | P0 |
| FR-06 | New providers/employers are `pending` until verified by a District Admin. | P0 |
| FR-07 | District Admin sees only their district's data (row-level scoping). | P0 |
| FR-08 | District stats are served from cache and refreshed ≤ 10 min. | P0 |
| FR-09 | UI available in Hindi and English with runtime toggle. | P1 |
| FR-10 | Admin can export funnel report as CSV. | P1 |
| FR-11 | Audit log for verification and status changes. | P1 |

---

## 8. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Performance** | P95 API latency < 300 ms (cached), < 800 ms (uncached); LCP < 2.5 s on 4G. |
| **Availability** | 99.5% monthly uptime (MVP); graceful degradation if Redis is down (fallback to DB). |
| **Scalability** | Handle 100k registered users, 1k concurrent users per launch district, horizontally scalable API. |
| **Security** | HTTPS everywhere, JWT + refresh rotation, RBAC, rate limiting, WAF, input validation, OWASP Top 10. |
| **Privacy** | Collect minimum PII; no Aadhaar numbers stored; consent captured; compliant with India's DPDP Act, 2023 principles; data deletion on request. |
| **Accessibility** | WCAG 2.1 AA basics; large touch targets; low-bandwidth mode. |
| **Localization** | Hindi + English; Devanagari font support; date/number formats. |
| **Observability** | Structured logs, error tracking, uptime checks, cache hit-ratio metric. |

---

## 9. Success Metrics (KPIs)

### North Star
**Number of youth placed (job/apprenticeship) through the platform per district per month.**

### Metric Tree

| Area | Metric | MVP Target (first 6 months, pilot districts) |
|------|--------|----------------------------------------------|
| Acquisition | Registered youth | 25,000 across pilot districts |
| Activation | Profile completion ≥ 80% | 60% of registrants |
| Engagement | Weekly active youth | 30% of registered |
| Training | Course enrollments requested | 8,000 |
| Training | Course completion rate | ≥ 65% |
| Supply | Verified training providers | 150 |
| Demand | Verified employers/MSMEs | 400 |
| Demand | Open opportunities posted | 1,500 |
| Outcome | Application → Hire conversion | ≥ 8% |
| Outcome | Time-to-first-application (new user) | < 3 days |
| Admin | Weekly admin dashboard usage | ≥ 1 session/week per district officer |
| Tech | P95 latency / Cache hit ratio / Error rate | < 300 ms / > 85% / < 1% |

---

## 10. Assumptions, Dependencies & Risks

**Assumptions:** Pilot starts with 2–3 districts; district admins will onboard providers/employers; SMS gateway available.

**Dependencies:** SMS/OTP provider (e.g., MSG91/Twilio), Supabase Postgres, Upstash Redis, Cloudflare account, district official data (LGD codes).

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| Low employer participation | High | Ground-level onboarding drives with DIC; WhatsApp-based quick posting; admin-assisted posting. |
| Fake providers/jobs | Medium | Mandatory verification workflow, report button, audit logs. |
| Low digital literacy | High | Hindi-first UI, minimal steps, voice/icon cues, assisted kiosks at CSC/ITI. |
| SMS cost/abuse | Medium | OTP rate limits (Redis), Cloudflare Turnstile, per-number caps. |
| Data privacy issues | Medium | Minimal PII, consent, role-scoped access, encryption at rest. |

---

## 11. Release Plan

| Release | Scope |
|---------|-------|
| **MVP (v1.0)** | F1–F7 for 2–3 pilot districts. |
| **v1.1** | Notifications via WhatsApp/SMS, CSV exports, block-level filters. |
| **v2.0** | Recommendations, certificates verification, state-wide rollout, integrations. |

---

## 12. Open Questions

1. Which pilot districts and which state's official skill taxonomy to adopt first?
2. Will District Admin accounts be provisioned by the Super Admin only, or via government email domain verification?
3. Which SMS/WhatsApp provider will be funded?
4. Is integration with the NCS portal / Skill India Digital in scope for v2?
