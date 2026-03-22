# Sprint 0 — Technical Tickets (Retrospective)
> Created retrospectively 2026-03-13. These tasks were executed before the sprint process was formalised. Captured here for audit trail and DoD completeness.
> Linked from `issue_backlog.md` (ISS-002, ISS-003, ISS-004).
> Project: site

---

## ISS-002 — DevOps Skeleton: Docker Compose, Nginx, .env.example

| Field | Value |
|---|---|
| **ID** | ISS-002 |
| **Type** | task |
| **Component** | devops |
| **Status** | resolved |
| **Owner** | Dominick |
| **Urgency** | high |
| **Sprint** | 0 |
| **Closed** | 2026-03-13 |

### Opening

Before any development can start, the team needs a runnable local stack and a documented environment configuration. Without it, every developer starts from scratch and there is no single source of truth for what ports, services, and variables the application expects.

This ticket covers the production-facing infrastructure skeleton — the base `docker-compose.yml`, the Nginx reverse-proxy config, and the `.env.example` template. The dev override lives in ISS-003.

### Requirements

- `docker-compose.yml` defines all production services: `nginx`, `django`, `nextjs`, `postgres`
- Nginx service is gated behind `profiles: ["production"]` so it does not start in dev
- All service definitions are present with correct image/build references and dependency order
- `.env.example` documents every required environment variable with safe placeholder values (no real secrets)
- `nginx/nginx.conf` provides a reference production-grade config: upstream blocks for Django and Next.js, HTTPS-ready, correct headers

### Definition of Done

- [ ] `docker-compose.yml` committed to `develop` (or `main`)
- [ ] `nginx/nginx.conf` committed
- [ ] `.env.example` committed with all known variables at time of creation
- [ ] All services can be referenced by other developers building on top of this skeleton
- [ ] No real secrets committed

### Notes

```
2026-03-13 Dominick: Delivered as part of initial devops deliverable. Nginx excluded from dev
via profile. Ports not yet bound to 127.0.0.1 in base file — that is handled in the dev
override (ISS-003). .env.example includes DATABASE_URL, SECRET_KEY, and CORS origins.
```

---

## ISS-003 — Dev Environment: docker-compose.override.yml, Test Runner, Port Binding

| Field | Value |
|---|---|
| **ID** | ISS-003 |
| **Type** | task |
| **Component** | devops |
| **Status** | resolved |
| **Owner** | Alessandro |
| **Urgency** | high |
| **Sprint** | 0 |
| **Closed** | 2026-03-13 |

### Opening

The production Docker Compose skeleton (ISS-002) is not suitable for day-to-day development: it has no source volume mounts, no hot-reload, and no test isolation. A separate `docker-compose.override.yml` (gitignored, auto-merged by Docker Compose) provides the dev-only layer without polluting the production config.

This was designed in WRK-001 (working session 2026-03-13) with Patrick (Architect), Dominick (Security), and Alessandro (Backend).

### Requirements

- `docker-compose.override.yml` must exist in the repo root and be listed in `.gitignore` (not committed — each developer creates their own from `.env.example`)
- Host-side ports bound to `127.0.0.1` only: `127.0.0.1:8000:8000` for Django, `127.0.0.1:5432:5432` for Postgres, `127.0.0.1:3000:3000` for Next.js — LAN exposure prevented (WRK-D2)
- Django container uses source volume mount and `runserver` for hot-reload
- Next.js container uses source volume mount and `next dev` for hot-reload
- A dedicated `test` service (gated via `profiles: ["test"]`) runs pytest against a separate `${POSTGRES_DB}_test` database to prevent dev data contamination
- `SESSION_COOKIE_SECURE=False` and `CSRF_COOKIE_SECURE=False` present in the dev `.env` section of `.env.example` (WRK-D7) — without these, Django admin login silently fails on HTTP

### Definition of Done

- [ ] `docker-compose.override.yml` template documented in `.env.example` or `README`
- [ ] `.gitignore` includes `docker-compose.override.yml` and `.env`
- [ ] Test runner confirmed working: `docker compose --profile test run --rm test pytest`
- [ ] Dev stack confirmed: `docker compose up` starts Django + Next.js + Postgres without Nginx
- [ ] `SESSION_COOKIE_SECURE` and `CSRF_COOKIE_SECURE` documented as `False` for dev in `.env.example`
- [ ] Security decision documented: WRK-D2 (LAN isolation), WRK-D7 (cookie security)

### Notes

```
2026-03-13 Patrick (WRK-001): Approved Option A — fully containerised dev. Nginx excluded via
profile. LAN isolation via 127.0.0.1 port binding.

2026-03-13 Dominick (WRK-001): Confirmed port binding to 127.0.0.1 solves local laptop exposure
risk. Docker container-side binding remains 0.0.0.0 — required for Docker port forwarding to work.

2026-03-13 Alessandro: Delivered as part of Django scaffold brief. Known gap at close: the
docker-compose.override.yml file itself was not committed (correct — it is gitignored). Owner to
create their own from the documented template. Orchestrator offered to assist.
```

---

## ISS-004 — Django Project Scaffold + Health Endpoint

| Field | Value |
|---|---|
| **ID** | ISS-004 |
| **Type** | task |
| **Component** | core |
| **Status** | resolved |
| **Owner** | Alessandro |
| **Urgency** | high |
| **Sprint** | 0 |
| **Closed** | 2026-03-13 |

### Opening

The Django project needed a scaffold that follows the agreed architecture: split settings (base/development/production), a working `api` app, a passing smoke test, and all tooling (Black, isort, Ruff, pre-commit) in place. This is the foundation every subsequent backend feature is built on.

Brief: `session_techlead_alessandro_003_scaffold.md`. Executed under TDD (Constitution §6): failing test committed first (RED), implementation second (GREEN).

### Requirements

- Django project `sitopresepi` created under `django/`
- Split settings: `settings/base.py`, `settings/development.py`, `settings/production.py`
  - `base.py`: all shared config, `dj-database-url` for `DATABASE_URL`, `SECRET_KEY` from env (no default in production path)
  - `development.py`: `DEBUG=True`, `ALLOWED_HOSTS` for localhost, `SESSION_COOKIE_SECURE=False`, `CSRF_COOKIE_SECURE=False`
  - `production.py`: `DEBUG=False`, HSTS, `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE=True`
- `api` Django app with:
  - `GET /api/health/` endpoint returning `{"status": "ok", "db": "ok"}` (db check via `SELECT 1`)
  - URL namespace `api:`
- Two-level `conftest.py`:
  - Project-level `django/conftest.py`: sets env var defaults before Django loads (`DJANGO_SECRET_KEY`, `DATABASE_URL=sqlite:///:memory:`)
  - App-level `django/api/tests/conftest.py`: app-specific fixtures (empty at scaffold)
- `requirements.txt` includes: `django`, `djangorestframework`, `django-cors-headers`, `dj-database-url==2.*`
- Pre-commit hooks: Black, isort, Ruff
- All tests pass: `pytest django/`

### Definition of Done

- [ ] `pytest` passes with zero failures
- [ ] `GET /api/health/` returns 200 with `{"status": "ok", "db": "ok"}`
- [ ] Split settings work: `DJANGO_SETTINGS_MODULE=sitopresepi.settings.development` starts the server
- [ ] Pre-commit hooks installed and passing
- [ ] TDD enforced: RED commit precedes GREEN commit in git history
- [ ] PR reviewed: Tech Lead (Patrick) + Backend domain (Alessandro) — two-reviewer model satisfied
- [ ] Merged to `develop` via `--no-ff` merge

### Notes

```
2026-03-13 Alessandro: Scaffold delivered. Three bugs caught and fixed in PR review before merge:
  Bug 1: test_health_endpoint_returns_200 asserted == {"status": "ok"} missing "db" field.
         Fix: updated assertion to {"status": "ok", "db": "ok"}.
  Bug 2: development.py SESSION_COOKIE_SECURE used != "true" (inverted logic).
         Default "False" evaluated to True — would have silently broken admin login on HTTP.
         Fix: changed to == "true".
  Bug 3: base.py SECRET_KEY raises KeyError in test env (no .env present).
         Fix: conftest.py sets os.environ.setdefault() before Django initialises.
  Missing: dj-database-url not in original requirements.txt — added during review.

2026-03-13 Patrick (Tech Lead): All bugs caught in review, none escaped to develop. TDD RED/GREEN
commits confirmed in git history.
```

---

## ISS-005 — Next.js Scaffold + i18n Routing

| Field | Value |
|---|---|
| **ID** | ISS-005 |
| **Type** | task |
| **Component** | frontend |
| **Status** | resolved |
| **Owner** | Ash |
| **Urgency** | high |
| **Sprint** | 0 |
| **Closed** | 2026-03-13 |

### Opening

The Next.js frontend needed a scaffold to match the Django scaffold: App Router structure, three-locale i18n routing, Tailwind CSS wired up, TypeScript strict mode, `lib/types.ts` as the API contract stub, one passing smoke test, and a production-ready Dockerfile. This is the foundation every subsequent frontend feature is built on.

Brief: `session_frontend_ash_001_adminux.md`. Executed under TDD (Constitution §6): failing test committed first (RED), implementation second (GREEN).

### Requirements

- Next.js 14.x project under `nextjs/`
- App Router with `[locale]` dynamic segment routing (`it`, `en`, `de`)
- `middleware.ts`: redirects prefix-less paths to `/it` (default locale); matcher excludes static assets
- `lib/i18n.ts`: `LOCALES`, `Locale` type, `DEFAULT_LOCALE`, `getSupportedLocales()`, `isValidLocale()`
- `lib/types.ts`: `HealthResponse` interface stub; catalogue/commerce types marked TBD
- `app/layout.tsx`: root layout owns `<html><body>` (Next.js App Router requirement)
- `app/[locale]/layout.tsx`: validates locale via `isValidLocale()`, calls `notFound()` on invalid; returns fragment (no html/body duplication)
- `app/[locale]/page.tsx`: minimal placeholder page per locale
- Tailwind CSS wired via `tailwind.config.ts` (design token source; brand colours/fonts as placeholders)
- `next.config.ts`: `output: "standalone"` for Docker multi-stage build
- `Dockerfile`: three-stage (deps → builder → runner), `node:20-alpine`, non-root user `nextjs` in runner
- TypeScript strict mode, ESLint + Prettier configured
- One passing unit test (`lib/i18n.test.ts`) covering `getSupportedLocales`, `isValidLocale`, `DEFAULT_LOCALE`

### Definition of Done

- [ ] `npm test` passes with zero failures (`lib/i18n.test.ts` green)
- [ ] `middleware.ts` redirects `/` → `/it`, passes through `/it/`, `/en/`, `/de/`
- [ ] `isValidLocale("fr")` returns false; `notFound()` triggered for unknown locales
- [ ] `next build` succeeds (no TypeScript errors, no missing html/body in root layout)
- [ ] TDD enforced: RED commit precedes GREEN commit in git history
- [ ] PR reviewed: Tech Lead (Patrick) + Frontend domain (Ash) — two-reviewer model satisfied
- [ ] Merged to `develop` via `--no-ff` merge

### Notes

```
2026-03-13 Ash: Scaffold delivered. Two bugs caught and fixed in PR review before merge:
  Bug 1: app/layout.tsx returned bare {children} without <html><body>.
         Next.js App Router requires root layout to own <html> and <body>; returning children
         directly causes a build error. Fix: added <html lang="it"><body> shell.
         Known limitation: root layout cannot dynamically set lang={locale} because it has no
         access to the [locale] route segment. Static "it" is acceptable for v1; to be revisited.
  Bug 2: jest.config.ts had two wrong key names:
         setupFilesAfterFramework → correct key is setupFilesAfterEnv
         testPathPattern → correct key is testMatch
         These would have caused jest to silently ignore the setup file and test pattern.

2026-03-13 Patrick (Tech Lead): Both bugs caught in review. Confirmed output: "standalone" present
for Docker build. TDD RED/GREEN commits confirmed.

2026-03-13 Ash: app/[locale]/layout.tsx: <html><body> removed, returns React fragment. ESLint
suppress comment added for react/jsx-no-useless-fragment on the fragment return.
```
