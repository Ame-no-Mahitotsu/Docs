# Security Checklist Sign-Off — Site-6 Pre-Launch
**Author:** Dominick (Security Engineer)
**Date:** 2026-03-29
**Sprint:** site-6
**Ticket:** ISS-125
**Status:** COMPLETE — 2 blocking GAPs identified (see summary)

---

## How to Read This Document

Each checklist item from `security_checklist.md` is assessed against one of three statuses:

| Status | Meaning |
|---|---|
| ✅ PASS | Verified by static code review — criterion met in committed code/config |
| ⏳ PENDING | Cannot be verified without a live deployment — must be re-run at launch |
| 🔴 GAP | Known failure — criterion is NOT met; ticket required before launch |

Items marked ⏳ PENDING must be re-verified using the commands in `security_checklist.md` immediately before production deployment. A separate deployment sign-off run is required — this document covers the code-review phase only.

---

## Blocking GAPs Summary

Two blocking issues were identified. Both must be resolved before launch.

| # | Item | Description | Ticket |
|---|---|---|---|
| GAP-1 | X-Forwarded-For trust not configured | Django settings have no `TRUSTED_PROXIES`/proxy trust config. nginx.conf explicitly flags this as CRITICAL — without it, audit log IP fields can be spoofed via crafted `X-Forwarded-For` headers. Any Django code reading `HTTP_X_FORWARDED_FOR` from `request.META` directly will see attacker-controlled data. | ISS-193 (new) |
| GAP-2 | `/de/datenschutz/` returns 404 | ISS-192 is open. The German-locale privacy page URL (`/de/datenschutz/`) has no rewrite in `next.config.ts`. The footer link in `layout.tsx` generates this URL; it 404s. G-01 is GDPR-CRITICAL and requires all three locale privacy pages to be accessible. | ISS-192 |

---

## Part 1 — Pre-Launch Assessment

### Layer 1 — Network & Infrastructure

| # | Status | Notes |
|---|---|---|
| N-01 | ⏳ PENDING | `nginx.conf` Server Block 1 issues `return 301 https://$host$request_uri` — config correct. Verify with `curl -I` against live server. |
| N-02 | ⏳ PENDING | Let's Encrypt cert provisioning — deployment step. No code-level verification possible. |
| N-03 | ⏳ PENDING | `nginx.conf` has `add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always`. Note: `preload` directive is absent from the Nginx header but is set via Django (`SECURE_HSTS_PRELOAD = True` in production.py). Django SecurityMiddleware will add `preload` to its own HSTS header. Two sources sending HSTS headers is redundant — recommend removing `SECURE_HSTS_PRELOAD` from `production.py` and adding `preload` to the Nginx header as the single authoritative source. Non-blocking at launch; flag for tidy-up. Verify full header with `curl` post-deploy. |
| N-04 | ⏳ PENDING | Docker Compose config binds Django (8000), Next.js (3000), and PostgreSQL (5432) to internal Docker network only. Nginx binds 80/443. Cloud firewall rules must be verified from outside the VPS at deploy time. |
| N-05 | ⏳ PENDING | SSH key-only auth is a VPS OS-level configuration — not in repo. Verify on server: `ssh root@domain` with password must fail. |
| N-06 | ⏳ PENDING | Port 8000 is not published to host in `docker-compose.yml` (internal network only). Verify with `curl http://domain:8000/api/health/` from outside. |
| N-07 | ⏳ PENDING | Same as N-06 — PostgreSQL 5432 on internal network. Verify with `nc -zv domain 5432`. |
| N-08 | ⏳ PENDING | Same as N-06 — Next.js 3000 on internal network. Verify with `curl http://domain:3000/`. |
| N-09 | ⏳ PENDING | fail2ban is a VPS OS-level package. Not in repo. Configure and verify at deploy time. |
| N-10 | ✅ PASS | `nginx.conf` defines `limit_req_zone $binary_remote_addr zone=api_write:10m rate=10r/m` and applies `limit_req zone=api_write burst=5 nodelay` to `location ~ ^/api/v1/(orders|contact)/`. Rate limit status 429. |
| N-11 | ✅ PASS | `nginx.conf` defines `limit_req_zone $binary_remote_addr zone=admin_login:10m rate=5r/m` and applies `limit_req zone=admin_login burst=3 nodelay` to `location /gestione/login/`. |

---

### Layer 2 — Django & Application Config

| # | Status | Notes |
|---|---|---|
| A-01 | ✅ PASS | `production.py` line 8: `DEBUG = False`. `base.py` also defaults to `False`. No path to `True` in production settings. |
| A-02 | ✅ PASS | `base.py` line 16: `SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]` — uses direct key access, not `os.environ.get()`. Django raises `KeyError` on startup if the variable is absent; the application does not start with a default or empty key. `.env.example` documents the placeholder with generation instructions. |
| A-03 | ✅ PASS | `.env` and `.env.local` are in `.gitignore`. `git log --all --full-history -- .env` produces no output. `.env.example` (safe to commit — placeholders only) is present. |
| A-04 | ⏳ PENDING | `.env` file permissions (`chmod 600`) must be verified on the server. `.env.example` documents this requirement. |
| A-05 | ✅ PASS | `production.py` line 11: `SESSION_COOKIE_SECURE = True`. |
| A-06 | ✅ PASS | `production.py` line 12: `CSRF_COOKIE_SECURE = True`. |
| A-07 | ✅ PASS | `base.py` line 18: `ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "").split(",")`. No hardcoded `*`. Defaults to empty list (Django rejects all hosts) if variable not set. Production `.env` must set this to the real domain only. |
| A-08 | ✅ PASS | `sitopresepi/urls.py` line 8: `path("gestione/", admin.site.urls)`. Not `/admin/`. |
| A-09 | ✅ PASS | `nginx.conf` line 687: `location /admin/ { return 404; }` — Django admin URL does not exist from the internet. |
| A-10 | ✅ PASS | `base.py` line 140–141: `CORS_ALLOWED_ORIGINS = os.environ.get("DJANGO_CORS_ALLOWED_ORIGINS", "").split(",")` and `CORS_ALLOW_ALL_ORIGINS = False`. CORS origin must be set in production `.env` to Next.js production URL only. |
| A-11 | ⏳ PENDING | PostgreSQL Django user privilege verification requires live DB access. Verify with `psql -U django_user -c "\du"` at deploy time. |
| A-12 | ⏳ PENDING | `UPDATE`/`DELETE` restriction on `admin_app_auditlog` table must be verified against the live DB. Verify at deploy time. |
| A-13 | ✅ PASS | `nginx.conf` `/media/` location: `alias /var/www/media/` with no CGI, no WSGI handler, no `fastcgi_pass`. Files are served as static. `X-Content-Type-Options: nosniff` header is also applied globally. |
| **A-new** | 🔴 GAP | **X-Forwarded-For proxy trust not configured.** `base.py` and `production.py` have no `TRUSTED_PROXY_IPS`, `USE_X_FORWARDED_HOST`, or equivalent. nginx.conf comment (line 575–579) flags this as CRITICAL: without proxy trust configuration, any code reading `request.META['HTTP_X_FORWARDED_FOR']` directly receives attacker-controlled data. Impact: AuditLog IP fields are spoofable; any future Django-level rate limiting by IP would be bypassable. **Ticket ISS-193 raised.** |

---

### Layer 3 — Authentication & Admin

| # | Status | Notes |
|---|---|---|
| B-01 | ⏳ PENDING | TOTP enrollment for Owner — verified only against live admin (`django_otp_totp_totpdevice` table). `django_otp` is in `INSTALLED_APPS` and `OTPMiddleware` is in `MIDDLEWARE` — framework is in place. |
| B-02 | ⏳ PENDING | TOTP enrollment for Gabriele — same as B-01. |
| B-03 | ⏳ PENDING | Login throttling functional test requires running app. Nginx rate limiting (N-11) is code-verified: 5r/m with burst 3 on `/gestione/login/`. Django-level IP lockout mechanism (referenced in session_dominick_001.md as "5 failed attempts → 15 min IP lockout") should be verified to confirm it is implemented in `admin_app` — run 6 failed login attempts against live server to confirm. |
| B-04 | ✅ PASS | `base.py`: `SESSION_COOKIE_HTTPONLY = True`, `SESSION_COOKIE_SAMESITE = "Strict"`. `production.py`: `SESSION_COOKIE_SECURE = True`. All three flags set. `nginx.conf`: `X-Frame-Options: DENY` as belt-and-suspenders against clickjacking. |
| B-05 | ✅ PASS | `base.py` line 105: `SESSION_COOKIE_AGE = 3600` (1 hour). Owner timeout matches the approved 1h value from BAR-001. Note: `ADMIN_SESSION_TIMEOUT_ARTISAN = 7200` is also configured (env-configurable). Verify inactivity-based reset behaviour at launch. |
| B-06 | ⏳ PENDING | AuditLog `login` entry recording — requires running admin to verify. `admin_app` module is in `INSTALLED_APPS`. |
| B-07 | ⏳ PENDING | AuditLog `login_failed` entry recording — same as B-06. |
| B-08 | ⏳ PENDING | Gabriele RBAC restrictions require live admin session to verify. `artisan` Django Group permissions configuration must be tested against product management, user management, and site configuration paths. |

---

### Layer 4 — Contact Form

| # | Status | Notes |
|---|---|---|
| C-01 | ✅ PASS | `CsrfViewMiddleware` is in `MIDDLEWARE` in `base.py` (position 4 of 7). Django enforces CSRF token validation on all non-safe methods (POST, PUT, PATCH, DELETE) for session-authenticated requests. API endpoints using DRF SessionAuthentication are also protected. |
| C-02 | ⏳ PENDING | Rate limit functional test (20 requests in 10s → 429) requires running server. Nginx `api_write` zone (10r/m, burst 5) is code-verified and will produce 429 responses. |
| C-03 | ⏳ PENDING | Email field validation — requires running API. Django REST Framework field validators should enforce this; verify with `curl` POST at deploy time. |
| C-04 | ✅ PASS | Django templates auto-escape HTML by default. Django admin renders stored content through its template engine with auto-escaping. DRF serializers do not render HTML. Stored XSS via admin view requires deliberate `mark_safe()` usage — none is present in the codebase. |
| C-05 | ⏳ PENDING | Admin notification email content — requires live email flow to verify. |
| C-06 | ⏳ PENDING | Error message content — requires running API to verify no stack traces leak. `DEBUG = False` in production removes Django debug pages; DRF `DEFAULT_RENDERER_CLASSES` should produce clean JSON errors. Verify with malformed POST at deploy time. |
| C-07 | ✅ PASS | ISS-122 resolved: `delete_expired_contacts` management command implemented with 365-day default retention. Privacy Policy content (IT/EN/DE) states the retention period. Data retention is defined and documented. |

---

### Layer 5 — Frontend / Next.js

| # | Status | Notes |
|---|---|---|
| F-01 | ✅ PASS | No secrets in codebase. All sensitive values are environment variables (`.env`, `.env.example` placeholders only). `NEXT_PUBLIC_SITE_URL` is the only `NEXT_PUBLIC_` variable — it is a public URL, not a secret. `npm build` does not embed secrets. |
| F-02 | ✅ PASS | `nginx.conf`: `add_header X-Frame-Options "DENY" always`. |
| F-03 | ✅ PASS | `nginx.conf`: `add_header X-Content-Type-Options "nosniff" always`. |
| F-04 | ✅ PASS | `nginx.conf`: `add_header Referrer-Policy "strict-origin-when-cross-origin" always`. |
| F-05 | ✅ PASS | `nginx.conf`: full CSP header present. `default-src 'self'`, `script-src 'self'`, `frame-ancestors 'none'`, `form-action 'self'`. Note: `style-src 'unsafe-inline'` is present (required for Next.js inline CSS injection) — this is a documented trade-off in the nginx.conf comments; a nonce-based approach is the future improvement. |
| F-06 | ✅ PASS | `npm audit --audit-level=high` in `nextjs/` → **0 vulnerabilities** (run 2026-03-29). Re-run immediately before deployment. |
| F-07 | ⏳ PENDING | `pip-audit` or `safety check` requires the Django Docker container environment. Run inside the container before deployment. |

---

### Layer 6 — GDPR & Legal

| # | Status | Notes |
|---|---|---|
| G-01 | 🔴 GAP | **DE privacy page 404s.** ISS-122 resolved — `/it/privacy/` and `/en/privacy/` are live (code-verified). `/de/datenschutz/` is NOT reachable: ISS-192 is open (missing rewrite in `next.config.ts`). This item cannot pass until ISS-192 is resolved. GDPR-CRITICAL — the DE privacy page is legally required before launch for DE market. |
| G-02 | ✅ PASS | ISS-122 resolved. Privacy policy content includes data controller name. Placeholder email (`privacy@[domain]`) is marked in the content — must be replaced with the real contact email before deployment. |
| G-03 | ⏳ PENDING | Contact form consent language — requires visual inspection of the rendered contact page. Verify that localised consent text is visible below the submit button in IT/EN/DE. |
| G-04 | ✅ PASS | ISS-122 resolved. Cookie notice implemented in `layout.tsx` footer. v1 uses only strictly necessary cookies (Django session, CSRF) — no analytics. `nginx.conf` Permissions-Policy header disables unused browser APIs. No third-party tracking scripts in the codebase. A simplified notice is legally sufficient under ePrivacy for strictly necessary cookies. |
| G-05 | ✅ PASS | Same as C-07. Retention period is stated in the Privacy Policy and enforced by the management command. |
| G-06 | ✅ PASS | No analytics or third-party tracking scripts in the Next.js codebase. No Google Analytics, Meta Pixel, or equivalent. No third-party JS loaded from external origins (CSP `script-src 'self'` would block it even if added accidentally). |
| G-07 | ⏳ PENDING | Gabriele's data access disclosure in Privacy Policy — requires content review of live policy. Check whether the policy states who handles enquiries. |

---

## Launch Gate Verdict

| Gate | Result | Action required |
|---|---|---|
| Static code review | **CONDITIONAL PASS** | Resolve GAP-1 (ISS-193) and GAP-2 (ISS-192) before proceeding |
| Deployment verification | **NOT YET RUN** | All ⏳ PENDING items must be re-verified at deploy time |
| Overall launch readiness | **BLOCKED** | Two blocking GAPs must be resolved |

**This document must be re-issued as a deployment sign-off once the site is live on the production VPS.** At that point, all PENDING items must be converted to PASS or documented exceptions before launch is approved.

---

## Tickets Raised From This Review

| Ticket | Title | Priority |
|---|---|---|
| ISS-193 | Configure Django TRUSTED_PROXIES — X-Forwarded-For spoofing risk | High / Major |
| ISS-192 | (existing) `/de/datenschutz/` rewrite missing from `next.config.ts` | Low / Minor |
