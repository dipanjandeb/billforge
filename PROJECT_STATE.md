# BillForge — Project State

> Living document. Update at the end of each lesson. Keep it short enough that future-you (or a fresh Claude session) can read it in 3 minutes and pick up exactly where things stand.

---

## Right Now

**Current status:** Lesson 2 complete. Django 5.2 skeleton running in Docker Compose with PostgreSQL 16. No apps, no models, no API endpoints yet.

**Next lesson:** Lesson 3 — create the first Django app (`customers`), with a model, admin registration, and the `apps/` folder convention.

**Last commit on `main`:** `feat: dockerized Django skeleton with Postgres`

---

## Project Identity

**Name:** BillForge
**One-line description:** GST-compliant subscription billing service for Indian B2B SaaS.
**Repo:** github.com/<your-username>/billforge (private until polished)

---

## Client Framing

> The story I tell consistently. Do not contradict any of these in any interview, README, or LinkedIn post.

- **Client:** A small Indian B2B SaaS (NDA, cannot name)
- **Size:** ~180 active subscriptions, roughly ₹2–3 Cr ARR
- **Their previous setup:** Zoho Books + Google Sheets + Razorpay (manual)
- **Why they hired me:** Finance team was losing ~2 days/month to billing ops + GST errors flagged in GSTR-1
- **Engagement:** ~8 weeks for MVP, then ~6 months of small follow-on work

---

## Tech Stack (locked in)

| Layer | Choice | Version | Notes |
|---|---|---|---|
| Language | Python | 3.12 (in container) | Container base: `python:3.12-slim` |
| Framework | Django | 5.2.1 | LTS, supported through 2028 |
| API framework | Django REST Framework | not added yet | Lesson 4 |
| DB | PostgreSQL | 16 (official image) | Single instance |
| DB driver | psycopg | 3.2.3 (binary extra) | Modern replacement for psycopg2 |
| Container | Docker + docker-compose | — | Single host, no Kubernetes |
| Auth | JWT via `djangorestframework-simplejwt` | planned | Lesson 4 |
| Async | Celery + Redis | planned | Around Lesson 7–8 |
| Payments | Razorpay Python SDK | planned | Lesson 7 |
| PDF | WeasyPrint | planned | Lesson 12 |
| Object storage | S3 (or Backblaze B2) | planned | Lesson 12 |

---

## Architecture Decisions (ADR-style log)

Each decision is one line of "what" and one line of "why". When a future Claude asks "why this and not that?", these are the canonical answers.

1. **Monolithic Django application, not microservices.**
   Team size of one, ~600 invoices/month — microservices would be cargo-culting. Boundaries kept clean via separate Django apps inside the monolith.

2. **Postgres, not MySQL.**
   Need advisory locks, JSONB, `FOR UPDATE SKIP LOCKED`, and `ON CONFLICT DO NOTHING` for idempotent webhook handling. MySQL can approximate some of these but Postgres is cleaner.

3. **Money stored as `BIGINT` paise, never floats, never `DECIMAL(rupees)`.**
   Floats have rounding error; storing rupees-as-decimal invites confusion. Paise-as-integer is exact, comparable, and unambiguous. Tax math uses `Decimal` intermediates with `ROUND_HALF_EVEN`, results stored as integer paise.

4. **psycopg3 (binary), not psycopg2.**
   Modern, async-capable, supported by Django 4.2+. New project — no reason to start on the older driver.

5. **Django project folder is `config/`, not `billforge/`.**
   Avoids `billforge/billforge/` naming confusion. Common convention from cookiecutter-django.

6. **Apps live under `apps/` (e.g. `apps/customers/`), not at the project root.**
   Keeps the top-level repo tidy. Set up in Lesson 3.

7. **Environment config via plain `os.environ`, not django-environ or python-decouple.**
   Fewer dependencies, more transparent. Required values use `os.environ["X"]` (crashes loudly on missing); optional values use `os.environ.get("X", default)`.

8. **`.env` for local dev, env vars on the host for production.**
   Never commit `.env`. `.env.example` ships in git as the template.

9. **Single VPS deployment (Hetzner CPX31 or AWS EC2 t3.small), not Kubernetes.**
   Matches the actual scale of the workload. Docker Compose on a single host is the deployment target.

10. **Idempotent webhook processing via a `webhook_events` table with unique `(provider, provider_event_id)`.** [planned]
    Razorpay delivers at-least-once. The unique constraint is the idempotency anchor.

11. **Gap-free invoice numbering via Postgres transaction-scoped advisory locks, keyed per financial year.** [planned]
    Postgres sequences can leave gaps on rollback; GSTR-1 filing rejects gaps. Advisory locks serialize invoice creation within a financial year only, with millisecond hold times.

12. **All timestamps `TIMESTAMPTZ` in UTC. IST only at presentation layer.** [planned]
    `USE_TZ=True` in Django settings. Date formatting on the PDF and API responses converts to IST.

---

## Conventions

- **Imports:** standard library, then third-party, then local. Sorted by `ruff` later.
- **Commit messages:** Conventional Commits. `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`.
- **Branch model:** work directly on `main` for solo work, but PRs for anything non-trivial (open and merge them yourself — the PR description is the documentation).
- **Money in code:** field names always end in `_paise` when the value is integer paise (`amount_paise`, `total_paise`, etc.). Helper `format_inr(paise)` converts for display.
- **App naming:** `customers`, `plans`, `subscriptions`, `invoices`, `payments`, `credit_notes`, `webhooks`, `dunning`, `reports`, `audit`, `common`.
- **API URL versioning:** `/api/v1/...` from day one.

---

## Repository Layout (current state)

```
billforge/
├── .env                       (local, gitignored)
├── .env.example
├── .gitignore
├── Dockerfile
├── README.md
├── docker-compose.yml
├── manage.py
├── requirements.txt
└── config/
    ├── __init__.py
    ├── asgi.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```

**Planned additions (not yet present):**
- `apps/` — all Django apps go under here
- `tests/` — top-level pytest suite
- `docs/` — architecture diagrams, ADRs, runbook
- `docker/` — additional Dockerfiles (worker, beat) once we add Celery

---

## Lessons Log

A short note on what was built and the checkpoint reached. Update at the end of each lesson.

### Lesson 1 — Foundation ✅
- Picked project name (BillForge) and client framing.
- Verified Python 3.12+ and git installed in WSL2.
- Installed Docker Desktop on Windows with WSL2 integration.
- Created GitHub repo `billforge`, cloned to `~/projects/billforge/`.
- Wrote initial `README.md` and pushed first commit.
- Checkpoint: repo exists with README, `docker run hello-world` succeeds.

### Lesson 2 — Django + Postgres in Docker ✅
- Wrote `.gitignore`, `requirements.txt` (Django 5.2.1, psycopg[binary] 3.2.3).
- Wrote `Dockerfile` (python:3.12-slim base, layer-cached pip install).
- Wrote `.env.example` and `.env` (local SECRET_KEY, DB creds).
- Wrote `docker-compose.yml` with `db` (postgres:16, healthcheck, named volume) and `web` (build from Dockerfile, port 8000, depends_on db service_healthy).
- Built image via `docker compose build`.
- Initialized Django project: `docker compose run --rm web django-admin startproject config .`
- Edited `config/settings.py` to read SECRET_KEY, DEBUG, ALLOWED_HOSTS, DATABASES from env vars.
- Ran migrations and created superuser.
- Checkpoint: `http://localhost:8000` shows Django welcome page; admin login works at `/admin/`.

### Lesson 3 — [in progress]
- Goal: first Django app (`customers`) under `apps/`, with model, admin registration, initial migration.

---

## Roadmap (rough — will shift)

Phase 1: **Scaffolding** (Lessons 1–6)
- L1 Foundation · L2 Django+Postgres · L3 First app (customers) · L4 DRF + JWT auth · L5 Plans · L6 Subscriptions

Phase 2: **Payments and invoicing** (Lessons 7–12)
- L7 Razorpay orders/checkout · L8 Webhooks + idempotency · L9 Invoice model · L10 GST/tax logic · L11 Invoice numbering (advisory locks) · L12 PDF generation (WeasyPrint + Celery)

Phase 3: **Operations** (Lessons 13–15)
- L13 Dunning · L14 Audit logs · L15 Reports

Phase 4: **Hardening and shipping** (Lessons 16–18)
- L16 Tests deep-dive · L17 Deployment to VPS · L18 README polish + demo Loom + sample materials

Suggested handoff point to a fresh chat: **after Lesson 6.** That's the end of the core scaffolding and the largest natural break before payments work begins.

---

## Open Questions / Parked Decisions

Things we've explicitly deferred. Each one has a "when to revisit" note.

- **Multi-tenancy.** Currently single-tenant (one customer = one installation). If we ever sell this as SaaS to multiple companies, we'd need a `tenants` table and row-level isolation. Revisit if the client framing changes.
- **Refunds.** Not in MVP. The Razorpay refund API is straightforward but the credit-note + GST refund flow is annoying. Revisit if the story needs it for interview defensibility.
- **Public API for client's customers.** Currently the API is internal only (admin/finance/support roles). If we expose endpoints to the SaaS's *end-customers* (for self-serve billing portal), we'd add API keys + scoped tokens. Revisit Lesson 17.
- **Search (Elasticsearch / Meilisearch).** No need at our scale. Postgres full-text on `customers.legal_name` will be enough until thousands of customers.
- **Observability beyond Sentry.** Considered Prometheus + Grafana, decided it's overkill for a single-VPS deployment. Revisit if deploying to multiple hosts.

---

## Things To Remember About Dipanjan's Background

(Context for any fresh Claude session.)

- Backend engineer, 3.5 years at Brillio + Infosys. Django/DRF, Postgres, AWS, JWT/RBAC, automation/integrations.
- Employment gap since July 2023. This project is one of three being built to fill that gap honestly — *as real projects with real commits over real time*, not fabricated.
- Currently in Bengaluru.
- Strong on Python backend, REST APIs, microservice-style integrations. Lighter on FastAPI (resume-listed but not heavily used in production). No deep frontend.
- Dev environment: Windows 11 + WSL2 Ubuntu + Docker Desktop + VS Code.

---

## How To Resume This Project In a New Chat

When this chat gets long or sluggish, start a fresh one and:

1. Upload your resume.
2. Upload this `PROJECT_STATE.md` file.
3. Open the new chat with something like:

   > Continuing the BillForge build from PROJECT_STATE.md. We finished Lesson N. Ready to start Lesson N+1.

4. Optionally also upload the original spec document (`project-1-gst-billing-spec.md`) if you want the new Claude to have the full architectural context. The state doc summarizes it, but the spec has the long-form rationale.

That's it. A fresh Claude reading this file should know enough to step in without re-litigating anything.
