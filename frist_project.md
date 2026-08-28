# AI-Powered Government Scheme Eligibility & Citizen Service Portal
## Complete System Architecture & Project Design Document

---

## 1. Project Overview & Scope Clarification

Before diving into architecture, one framing point matters for how you build this: this project should be positioned as a **decision-support and information-aggregation tool**, not an official government system. That distinction shapes almost every design decision below — especially around data sourcing, disclaimers, and what the "AI" actually does (matching/ranking against a curated database, not inventing eligibility rules).

For a final-year project, this is an excellent choice because it touches nearly every skill a software engineering evaluator wants to see: full-stack development, database design, an ML component that's genuinely justifiable (not bolted on), security/auth, multilingual UX, and a real social-impact narrative. The sections below give you everything needed to build, defend, and demo it.

---

## 2. High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│  React.js SPA (Web) — Responsive, PWA-capable                    │
│  Marathi / Hindi / English (i18next)                              │
└───────────────────────────┬───────────────────────────────────────┘
                             │ HTTPS / REST (JSON)
┌───────────────────────────▼───────────────────────────────────────┐
│                       API GATEWAY / BACKEND                       │
│  Django REST Framework or FastAPI                                 │
│  ┌───────────────┬────────────────┬───────────────────────────┐  │
│  │ Auth Service   │ Scheme Service │ Recommendation Service     │  │
│  │ (JWT/OAuth)    │ (CRUD, search) │ (ML model, rule engine)    │  │
│  ├───────────────┼────────────────┼───────────────────────────┤  │
│  │ Profile Service│ Tracking       │ Notification Service       │  │
│  │                │ Service        │ (email/SMS/in-app)         │  │
│  └───────────────┴────────────────┴───────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────────┘
                             │
┌───────────────────────────▼───────────────────────────────────────┐
│                        DATA LAYER                                 │
│  PostgreSQL (primary relational store)                            │
│  Redis (session cache, rate limiting, OTP store) — optional       │
│  Object storage (documents/uploads) — S3-compatible / local       │
└─────────────────────────────────────────────────────────────────┘
                             │
┌───────────────────────────▼───────────────────────────────────────┐
│              ADMIN / DEPARTMENT PORTAL (separate route)           │
│  Scheme data management, analytics, application status updates    │
└─────────────────────────────────────────────────────────────────┘
```

**Why this layered/service-oriented (not microservices) approach:** For a college-scale project, a **modular monolith** (one Django/FastAPI codebase organized into clean service modules) is the right call. True microservices would add deployment complexity disproportionate to the project's scale, while a modular monolith still lets you *talk about* service boundaries in your architecture defense and refactor into microservices later if needed.

---

## 3. User Roles & Permissions

| Role | Description | Key Permissions |
|---|---|---|
| **Citizen (Guest)** | Unauthenticated visitor | Browse public scheme info, use eligibility checker without saving profile |
| **Citizen (Registered)** | Logged-in user | Manage profile, get personalized recommendations, save schemes, track applications, receive notifications |
| **Department Officer** | Represents a government department | View/update application statuses for their department's schemes, respond to queries, view department-level analytics |
| **Content Moderator / Data Curator** | Maintains scheme database | Add/edit/verify scheme data, mark source + last-updated date, retire outdated schemes |
| **Super Admin** | Platform administrator | Full access: user management, role assignment, system-wide analytics, audit logs, content approval workflow |

**Role-based access control (RBAC) design principle:** Permissions should be attached to *roles*, not individual users, and enforced at the API layer (via decorators/middleware) — never trust frontend-only restriction. Each endpoint should declare required role(s) explicitly.

---

## 4. Database Schema & Relationships

### 4.1 Core Entities (PostgreSQL)

**users**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| full_name | VARCHAR(150) | |
| email | VARCHAR(150) UNIQUE | |
| phone | VARCHAR(15) UNIQUE | |
| password_hash | VARCHAR(255) | bcrypt/argon2 |
| role | ENUM | citizen, officer, curator, admin |
| preferred_language | ENUM | en, hi, mr |
| is_verified | BOOLEAN | |
| created_at, updated_at | TIMESTAMP | |

**citizen_profiles**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| date_of_birth | DATE | |
| gender | VARCHAR(20) | |
| occupation | VARCHAR(100) | |
| annual_income | DECIMAL | |
| education_level | VARCHAR(100) | |
| category | ENUM | General, SC, ST, OBC, EWS, etc. |
| state | VARCHAR(100) | |
| district | VARCHAR(100) | |
| disability_status | BOOLEAN | |
| marital_status | VARCHAR(30) | |
| updated_at | TIMESTAMP | |

**schemes**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| title | VARCHAR(255) | |
| description | TEXT | |
| department | VARCHAR(150) | |
| category | VARCHAR(100) | education, health, agriculture, employment, housing, etc. |
| eligibility_criteria | JSONB | structured rules (see §7) |
| benefits | TEXT | |
| required_documents | JSONB | array of document types |
| application_procedure | TEXT | |
| official_link | VARCHAR(500) | |
| start_date, end_date | DATE | nullable if evergreen |
| source_reference | VARCHAR(500) | **mandatory** citation of official source |
| last_verified_date | DATE | **mandatory** |
| is_active | BOOLEAN | |
| created_by, updated_by | UUID (FK → users.id) | |
| created_at, updated_at | TIMESTAMP | |

**applications** (tracking)
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK) | |
| scheme_id | UUID (FK) | |
| status | ENUM | saved, applied_externally, under_review, approved, rejected |
| status_note | TEXT | |
| applied_on | DATE | |
| last_updated_by | UUID (FK → users.id) | officer who updated |
| updated_at | TIMESTAMP | |

**notifications**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK) | |
| type | ENUM | new_scheme, deadline_reminder, status_change |
| message | TEXT | |
| is_read | BOOLEAN | |
| created_at | TIMESTAMP | |

**recommendation_logs** (for ML feedback loop + analytics)
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK) | |
| scheme_id | UUID (FK) | |
| match_score | DECIMAL | |
| shown_at | TIMESTAMP | |
| user_action | ENUM | clicked, saved, dismissed, applied |

**audit_logs**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| actor_id | UUID (FK → users.id) | |
| action | VARCHAR(100) | |
| entity_type, entity_id | VARCHAR | |
| timestamp | TIMESTAMP | |

### 4.2 Relationships (ER Description)

- `users` (1) ──── (1) `citizen_profiles` — one profile per citizen user
- `users` (1) ──── (∞) `applications` — a citizen can track many applications
- `schemes` (1) ──── (∞) `applications` — a scheme can be applied to by many citizens
- `users` (1) ──── (∞) `notifications`
- `users` (1) ──── (∞) `recommendation_logs` — (∞) ──── (1) `schemes`
- `users` (1) ──── (∞) `schemes` via `created_by`/`updated_by` (curator relationship)
- `users` (1) ──── (∞) `audit_logs`

For your ER diagram deliverable, draw this as a standard crow's-foot diagram: `users` as the central entity, with `citizen_profiles` as a strict 1:1 extension table, and `schemes`, `applications`, `notifications` radiating out as 1:many. `applications` is effectively a many-to-many junction between `users` and `schemes` with extra attributes (status, dates) — model it as its own table rather than a pure join table since it carries state.

---

## 5. AI/ML Recommendation Methodology

This is the part evaluators will scrutinize most, so be precise about what kind of "AI" this actually is — overclaiming here will hurt you in a viva/defense.

### 5.1 Recommended approach: Hybrid rule-based + ML ranking

**Stage 1 — Rule-based eligibility filtering (deterministic, explainable):**
Each scheme's `eligibility_criteria` is stored as structured JSON, e.g.:
```json
{
  "min_age": 18,
  "max_age": 35,
  "max_income": 250000,
  "category": ["SC", "ST", "OBC", "EWS"],
  "occupation": ["student", "unemployed"],
  "state": ["Maharashtra"],
  "gender": ["any"]
}
```
A rules engine compares the citizen's profile against these fields and produces a **binary or partial eligibility score** (e.g., 8/10 criteria matched). This stage should do the heavy lifting — it's transparent, auditable, and doesn't risk hallucinating eligibility, which matters a lot given your own design principle about not inventing rules.

**Stage 2 — ML-based ranking/personalization (where the "AI" genuinely adds value):**
Once you have a filtered candidate set of eligible/near-eligible schemes, use ML to **rank and personalize**, not to determine eligibility:
- **Content-based filtering**: represent each scheme as a feature vector (category, department, keywords via TF-IDF) and each citizen's profile similarly; rank schemes by cosine similarity to surface the most relevant ones first.
- **Collaborative signals** (once you have usage data): use `recommendation_logs` (clicks, saves, applications) to boost schemes that similar citizens engaged with — classic collaborative filtering with scikit-learn or a simple matrix factorization.
- **NLP for search/matching**: use scikit-learn's `TfidfVectorizer` + cosine similarity, or a lightweight embedding model, so citizens can type free-text queries ("I am a farmer's daughter looking for scholarship") and get matched against scheme descriptions — this is a strong, demoable NLP component.

### 5.2 Why this hybrid design (not pure ML)

- **Explainability**: Citizens and evaluators need to see *why* a scheme was recommended ("You matched 9/10 criteria: age ✓, income ✓, category ✓..."). A black-box model alone can't do this well.
- **Cold start**: You won't have enough real usage data as a college project to train a robust collaborative model from scratch — rule-based filtering works from day one.
- **Safety**: Given your own requirement not to invent eligibility rules, having ML *override* deterministic criteria would be risky. ML should refine ranking, not redefine eligibility.

### 5.3 Suggested tech implementation
- `scikit-learn` for TF-IDF, cosine similarity, and optionally a simple classifier (Logistic Regression / Random Forest) trained on synthetic labeled data (does-citizen-profile-X-match-scheme-Y) to predict match probability as a secondary ranking signal.
- Keep the model retrainable via a scheduled script as scheme data and user interaction logs grow.

---

## 6. API Design (REST Endpoints)

**Auth**
- `POST /api/auth/register`
- `POST /api/auth/login` → returns JWT access + refresh token
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `POST /api/auth/verify-otp` (phone/email verification)

**Citizen Profile**
- `GET /api/profile/me`
- `PUT /api/profile/me`

**Schemes**
- `GET /api/schemes` — filters: category, department, state, keyword, pagination
- `GET /api/schemes/{id}`
- `POST /api/schemes` (curator/admin only)
- `PUT /api/schemes/{id}` (curator/admin only)
- `DELETE /api/schemes/{id}` (soft delete, admin only)

**Recommendations**
- `POST /api/recommendations` — body: citizen profile snapshot or profile_id → returns ranked scheme list with match scores and matched/unmatched criteria breakdown
- `POST /api/recommendations/search` — free-text NLP search

**Applications / Tracking**
- `GET /api/applications` — citizen's own tracked applications
- `POST /api/applications` — save/track a scheme
- `PUT /api/applications/{id}` — update status (officer/admin)
- `GET /api/applications/{id}`

**Notifications**
- `GET /api/notifications`
- `PUT /api/notifications/{id}/read`

**Admin / Analytics**
- `GET /api/admin/analytics/overview` — total users, top schemes, category-wise distribution
- `GET /api/admin/analytics/scheme/{id}` — views, saves, applications for one scheme
- `GET /api/admin/audit-logs`

All endpoints should return consistent JSON envelopes (`{ "success": bool, "data": ..., "error": ... }`) and use proper HTTP status codes (401 for auth failures, 403 for permission failures, 422 for validation errors).

---

## 7. Authentication & Security Architecture

- **JWT-based auth**: short-lived access tokens (~15 min) + longer-lived refresh tokens stored in httpOnly, secure cookies (not localStorage, to reduce XSS token theft risk).
- **Password security**: bcrypt or Argon2 hashing, never reversible encryption.
- **RBAC middleware**: every protected route checks role via a decorator (Django: custom permission classes; FastAPI: dependency injection) — enforced server-side always.
- **Input validation**: use Pydantic (FastAPI) or DRF serializers (Django) for strict schema validation on every input; sanitize free-text search inputs to prevent injection.
- **Rate limiting**: on auth endpoints especially (login, OTP) to prevent brute-force — `django-ratelimit` or a Redis-backed limiter.
- **HTTPS everywhere**: enforce via reverse proxy (Nginx) with TLS termination; HSTS headers.
- **Data protection**: encrypt sensitive PII at rest where feasible (income, category data), and log access to sensitive fields in `audit_logs`.
- **CORS**: restrict allowed origins to your known frontend domain(s).
- **OWASP Top 10 awareness**: parameterized queries (ORM handles this by default), CSRF protection for cookie-based sessions, output encoding to prevent XSS in React (React escapes by default — avoid `dangerouslySetInnerHTML`).

---

## 8. Backend Folder Structure (Django example)

```
backend/
├── config/                  # project settings, urls, wsgi/asgi
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
├── apps/
│   ├── users/                # auth, roles, registration
│   ├── profiles/              # citizen_profiles
│   ├── schemes/                # scheme CRUD, search
│   ├── recommendations/        # rule engine + ML ranking
│   │   ├── rules_engine.py
│   │   ├── ml_ranker.py
│   │   └── nlp_search.py
│   ├── applications/           # tracking
│   ├── notifications/
│   └── analytics/
├── common/
│   ├── permissions.py          # RBAC decorators/classes
│   ├── pagination.py
│   └── exceptions.py
├── ml_models/
│   ├── train_model.py
│   ├── saved_models/
│   └── data/                   # synthetic training data
├── tests/
├── requirements.txt
├── Dockerfile
└── manage.py
```

*(If you go FastAPI instead, mirror this with `routers/`, `schemas/` (Pydantic), `services/`, `models/` (SQLAlchemy) — the module boundaries stay the same.)*

---

## 9. Frontend Page Structure (React)

```
src/
├── pages/
│   ├── Home/
│   ├── Auth/ (Login, Register, OTP Verify)
│   ├── Onboarding/ (profile setup wizard)
│   ├── Dashboard/ (citizen — recommended schemes, saved, tracked)
│   ├── SchemeDetail/
│   ├── SchemeSearch/ (filters + NLP search bar)
│   ├── ApplicationTracker/
│   ├── Notifications/
│   ├── Profile/
│   └── Admin/
│       ├── SchemeManagement/
│       ├── UserManagement/
│       ├── AnalyticsDashboard/
│       └── AuditLogs/
├── components/
│   ├── common/ (Navbar, Footer, LanguageSwitcher, Loader)
│   ├── scheme/ (SchemeCard, EligibilityBadge, MatchScoreBar)
│   └── forms/
├── i18n/
│   ├── en.json
│   ├── hi.json
│   └── mr.json
├── services/ (API client wrappers, one per backend module)
├── context/ (AuthContext, LanguageContext)
├── hooks/
└── utils/
```

---

## 10. Multilingual Implementation (Marathi, Hindi, English)

- **Library**: `react-i18next` on frontend for UI strings (buttons, labels, static content).
- **Dynamic content (scheme data)**: store `title_en`, `title_hi`, `title_mr` (and similarly for description, eligibility text) as separate columns or a JSONB `translations` field on the `schemes` table — don't rely on machine translation for legal/eligibility text, since accuracy matters. Curators should input verified translations, or at minimum clearly flag machine-translated content as unverified.
- **Language detection/switch**: store `preferred_language` on the user profile; default to browser locale for guests; persist selection via a language switcher in the navbar.
- **Font/rendering**: ensure Devanagari script rendering is tested (Noto Sans Devanagari or similar web font) since default system fonts sometimes render Marathi/Hindi poorly.

---

## 11. Admin & Department Dashboard Design

**Admin Dashboard sections:**
1. **Overview** — total registered citizens, active schemes, applications this month, top 5 most-recommended schemes (charts via Chart.js/Recharts)
2. **Scheme Management** — table view with search/filter, add/edit form with mandatory `source_reference` and `last_verified_date` fields, approval workflow (curator submits → admin approves before going live, for data quality control)
3. **User Management** — role assignment, account status (active/suspended)
4. **Application Oversight** — department officers see only applications routed to their department; can update status and add notes
5. **Analytics & Reports** — category-wise scheme distribution, geographic demand heatmap (which districts show most eligible citizens for which categories), exportable CSV/PDF reports
6. **Audit Logs** — searchable log of all admin/curator actions

## 12. Citizen Dashboard Design

1. **Profile completeness meter** — nudges citizens to complete profile for better recommendations
2. **Recommended for You** — ranked scheme cards with match-score badges and a short "why this matched" explanation
3. **Saved Schemes**
4. **Tracked Applications** — status timeline view
5. **Notifications feed**
6. **Search & Browse** — full catalog with filters, independent of personalized recommendations
7. **Disclaimer banner** (persistent, not just on first login): recommendations are informational; final eligibility is determined by the relevant government authority — this should appear on every scheme detail page, not buried in a terms page.

---

## 13. Sample Demo Data (clearly fictional)

> ⚠️ All data below is fictional/demo data for development and testing only. Do not treat as real scheme information.

```json
{
  "id": "demo-scheme-001",
  "title": "Demo Scholarship for Rural Students (FICTIONAL)",
  "department": "Demo State Education Department",
  "category": "education",
  "eligibility_criteria": {
    "min_age": 15, "max_age": 25,
    "max_income": 200000,
    "category": ["SC", "ST", "OBC", "EWS"],
    "occupation": ["student"],
    "state": ["Demo State"]
  },
  "benefits": "Placeholder: annual stipend of ₹X (demo value)",
  "required_documents": ["Income certificate (demo)", "Caste certificate (demo)", "Bonafide student certificate (demo)"],
  "official_link": "https://example.gov.in (placeholder — replace with verified source)",
  "source_reference": "PLACEHOLDER — must be replaced with actual official gazette/portal citation before production use",
  "last_verified_date": "2026-01-01",
  "is_active": true
}
```

When you populate the real database, every scheme record must trace to an actual, citable government source (state portal, gazette notification, myscheme.gov.in, or the relevant department site) — this is both good engineering practice and important for the project's credibility.

---

## 14. Development Roadmap (Beginner → Completed Project)

**Phase 1 — Foundations (Weeks 1–2)**
- Set up Git repo, Docker Compose (Postgres + backend + frontend)
- Django/FastAPI skeleton, React skeleton with routing
- Basic user model + JWT auth (register/login)

**Phase 2 — Core Data Layer (Weeks 3–4)**
- Design and migrate full database schema
- Build Scheme CRUD APIs + admin scheme management UI
- Seed with 15–20 real, well-researched government schemes (start small, verified)

**Phase 3 — Citizen Experience (Weeks 5–6)**
- Citizen profile creation/onboarding flow
- Scheme browsing, search, and filtering UI
- Rule-based eligibility engine (Stage 1 from §5)

**Phase 4 — AI/ML Layer (Weeks 7–8)**
- TF-IDF/cosine-similarity content-based ranking
- NLP free-text search
- Match-score explanation UI ("why recommended")

**Phase 5 — Tracking & Notifications (Week 9)**
- Application tracking CRUD + status timeline
- In-app notifications; optional email via SMTP (e.g., SendGrid free tier)

**Phase 6 — Multilingual + Accessibility (Week 10)**
- i18next integration, translated static UI
- Accessibility pass (ARIA labels, keyboard nav, contrast)

**Phase 7 — Admin Analytics + Polish (Week 11)**
- Analytics dashboard with charts
- Audit logging, role management UI

**Phase 8 — Testing, Deployment, Documentation (Week 12)**
- Full test suite, deployment, README, demo video, project report

*(Compress into 6–8 weeks if timeline is shorter; the phase order is more important than exact durations — get auth + data layer solid before layering ML on top.)*

---

## 15. Testing Strategy

| Layer | Approach | Tools |
|---|---|---|
| Backend unit tests | Model validation, rules engine logic, ML ranking function outputs | pytest / Django TestCase |
| API integration tests | Endpoint behavior, auth/permission enforcement | pytest + DRF test client, or FastAPI TestClient |
| Frontend unit tests | Component rendering, form validation | Jest + React Testing Library |
| E2E tests | Full user flows: register → profile → get recommendations → track application | Playwright or Cypress |
| Security testing | Auth bypass attempts, injection testing, rate-limit verification | OWASP ZAP (basic scan), manual checklist |
| ML evaluation | Precision/recall of rule-engine matches against a hand-labeled test set of citizen profiles × schemes | scikit-learn metrics |

---

## 16. Deployment Strategy

- **Containerization**: Dockerfiles for backend and frontend, `docker-compose.yml` for local dev (Postgres + backend + frontend + optional Redis).
- **CI/CD**: GitHub Actions — run tests on PR, build Docker images on merge to main.
- **Hosting options (free/low-cost, suitable for a college project)**:
  - Backend: Render, Railway, or a free-tier VM
  - Frontend: Vercel or Netlify
  - Database: Render/Railway managed Postgres, or Supabase
- **Environment separation**: `.env` files for dev/staging/prod secrets (never commit secrets — use `.env.example` in repo).
- **Domain/HTTPS**: use the hosting platform's free TLS (Let's Encrypt via Render/Vercel) if you want a live demo link.

---

## 17. Future Improvements (good talking points for your report's "future scope" section)

- Voice-based query support for low-literacy users (regional language speech-to-text)
- WhatsApp/SMS-based scheme alerts for citizens without reliable internet
- Integration with DigiLocker for auto-fetching documents
- Aadhaar-based verification (would require significant compliance work — good to mention as a "real deployment" consideration rather than implement)
- Chatbot assistant using an LLM for conversational scheme discovery
- Offline-first PWA mode for low-connectivity areas
- Grievance redressal module

---

## 18. Making This a Strong Final-Year Project

- **Scope it realistically**: don't try to integrate real government APIs (most don't have public APIs) — instead build a well-curated, clearly-sourced database of real schemes (aim for 20–30 well-researched real schemes rather than a shallow 100+ scraped list).
- **Emphasize the hybrid rule-engine + ML design** in your report — evaluators respond well to a well-justified, explainable architecture over an unexplainable "we used AI" claim.
- **Include a comparison/evaluation section**: measure recommendation precision on a test set of synthetic citizen profiles against manually-determined correct matches.
- **Have a live demo with realistic-looking (but clearly labeled demo) accounts** for citizen, officer, and admin roles.
- **Document limitations honestly** in your report: this is a discovery tool, not an authoritative source — that intellectual honesty is a strength in an academic evaluation.

---

## 19. Technology Justification Summary

| Technology | Why it fits this project |
|---|---|
| **React.js** | Component-based UI suits a dashboard-heavy app with reusable scheme cards, forms, and role-specific views; huge ecosystem for i18n and accessibility libraries |
| **Django (or FastAPI)** | Django gives you auth, admin panel, and ORM out of the box — fast for a solo/small-team timeline. FastAPI is a good alternative if you want async performance and prefer Pydantic-based validation with less built-in scaffolding |
| **PostgreSQL** | Strong relational integrity for interlinked entities (users, schemes, applications) plus native JSONB support for flexible fields like `eligibility_criteria` and multilingual `translations` |
| **scikit-learn** | Lightweight, well-documented, sufficient for TF-IDF/cosine-similarity ranking and simple classifiers — no need for heavy deep-learning infra at this scale |
| **JWT/OAuth** | Stateless auth scales well and is standard for SPA + REST API architectures |
| **Docker** | Ensures consistent dev/deployment environments and is an expected skill signal for evaluators |
| **Git/GitHub** | Version control and portfolio visibility — also lets you demonstrate a commit history showing iterative development, which examiners often check |

---

## Closing Note

Build this in the phase order above — get the rule-based eligibility engine rock-solid and well-tested *before* layering ML ranking on top of it. A working, explainable, well-sourced scheme-matching tool with modest ML will score and demo better than an ambitious system with unverified data or an unexplainable "black box" recommender.
