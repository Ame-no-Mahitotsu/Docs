# Security Checklist — SitoPresepi2
**Author:** Dominick (Security Engineer)
**Date:** 2026-03-13
**Status:** Approved deliverable — see ../site/session_security_dominick_001_baseline.md
**Project:** general

**Scope:** Single-VPS deployment (DigitalOcean Frankfurt). Contact-first v1 — no online ordering, no payment processing. Personal data in scope: contact form submissions only (name, email, message). Admin users: Owner (Django superuser) + Gabriele (artisan Django Group).

---

## Priority Legend
- 🔴 CRITICAL — must pass before launch. Site does not go live with any of these failing.
- 🟡 IMPORTANT — should pass; document any exception with mitigation.
- 🟢 GOOD PRACTICE — genuine value; not blocking.

---

## Part 1 — Pre-Launch Checklist

### Layer 1 — Network & Infrastructure

| # | Check | How to verify | Pass criterion | Priority |
|---|---|---|---|---|
| N-01 | HTTPS enforced — HTTP redirects to HTTPS | `curl -I http://yourdomain.com` | Response is `301` or `302` to `https://` | 🔴 CRITICAL |
| N-02 | Valid TLS certificate | `openssl s_client -connect yourdomain.com:443 2>/dev/null \| openssl x509 -noout -dates` | Not expired; CN matches domain | 🔴 CRITICAL |
| N-03 | HSTS header present and correct | `curl -I https://yourdomain.com \| grep -i strict` | `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` | 🔴 CRITICAL |
| N-04 | Only ports 80 and 443 open to internet | `nmap -sV yourdomain.com` from outside VPS | Only 80 (redirect) and 443 open. No 8000, 5432, 3000, 22 visible. | 🔴 CRITICAL |
| N-05 | SSH not on port 22 OR key-only auth | `ssh root@yourdomain.com` with password | Connection refused with password; key required. | 🔴 CRITICAL |
| N-06 | Django port 8000 not reachable from internet | `curl http://yourdomain.com:8000/api/health/` | Connection refused or timeout | 🔴 CRITICAL |
| N-07 | PostgreSQL port 5432 not reachable from internet | `nc -zv yourdomain.com 5432` | Connection refused | 🔴 CRITICAL |
| N-08 | Next.js port 3000 not reachable from internet | `curl http://yourdomain.com:3000/` | Connection refused | 🔴 CRITICAL |
| N-09 | fail2ban active | `sudo fail2ban-client status sshd` | Status shows active, ban list exists | 🟡 IMPORTANT |
| N-10 | Nginx rate limiting on contact form endpoint | Review `nginx.conf` for `limit_req_zone` and `limit_req` on contact API path | Rate limit zone defined; `limit_req` applied | 🟡 IMPORTANT |
| N-11 | Nginx rate limiting on admin login endpoint | Review `nginx.conf` — `limit_req` on `/gestione/login/` | Rate limit zone applied | 🟡 IMPORTANT |

---

### Layer 2 — Django & Application Config

| # | Check | How to verify | Pass criterion | Priority |
|---|---|---|---|---|
| A-01 | `DEBUG=False` in production | `grep DEBUG .env` on server; or request a deliberate 404 URL | No Django debug page — standard 404 only | 🔴 CRITICAL |
| A-02 | `SECRET_KEY` is production-grade | `grep SECRET_KEY .env` on server | Not the test value (`test-only-insecure-...`); ≥50 random characters | 🔴 CRITICAL |
| A-03 | `.env` not in git repository | `git log --all --full-history -- .env` | No output | 🔴 CRITICAL |
| A-04 | `.env` file permissions | `ls -la .env` on server | `600` — readable only by owner | 🟡 IMPORTANT |
| A-05 | `SESSION_COOKIE_SECURE=True` | `grep SESSION_COOKIE_SECURE .env` | `True` | 🔴 CRITICAL |
| A-06 | `CSRF_COOKIE_SECURE=True` | `grep CSRF_COOKIE_SECURE .env` | `True` | 🔴 CRITICAL |
| A-07 | `ALLOWED_HOSTS` set to production domain only | `grep ALLOWED_HOSTS .env` or `settings/production.py` | Only the real domain — no `*`, no `localhost` | 🔴 CRITICAL |
| A-08 | Admin URL is non-obvious | Check `urls.py` | Admin at `/gestione/` or similar — **not** `/admin/` | 🟡 IMPORTANT |
| A-09 | Default `/admin/` returns 404 | `curl -I https://yourdomain.com/admin/` | 404 — not a Django admin login page | 🟡 IMPORTANT |
| A-10 | CORS locked to Next.js production domain | `grep CORS .env` or `settings/production.py` | `CORS_ALLOWED_ORIGINS` contains only production origin; not `*` | 🔴 CRITICAL |
| A-11 | PostgreSQL Django user is not superuser | `psql -U django_user -c "\du"` | No `Superuser` attribute | 🟡 IMPORTANT |
| A-12 | No UPDATE/DELETE privilege on AuditLog table for Django DB user | `psql -U django_user -c "\dp admin_app_auditlog"` | Only `SELECT` and `INSERT` granted | 🟡 IMPORTANT |
| A-13 | Uploaded images served as static files only (not executable) | Review Nginx `location /media/` config | No CGI/WSGI handler on media path; files served as-is | 🟡 IMPORTANT |

---

### Layer 3 — Authentication & Admin

| # | Check | How to verify | Pass criterion | Priority |
|---|---|---|---|---|
| B-01 | TOTP enrolled — Owner account | Check Django-otp device list in admin | ≥1 confirmed TOTPDevice for Owner | 🔴 CRITICAL |
| B-02 | TOTP enrolled — Gabriele account | Check Django-otp device list | ≥1 confirmed TOTPDevice for Gabriele | 🔴 CRITICAL |
| B-03 | Login throttling active | Make 6 failed login attempts at `/gestione/login/` | 6th attempt blocked; IP locked for 15 minutes | 🔴 CRITICAL |
| B-04 | Session cookie flags | Browser DevTools → Application → Cookies after admin login | `HttpOnly`, `Secure`, `SameSite=Strict` flags all set | 🔴 CRITICAL |
| B-05 | Session timeout — Owner | Remain inactive >1 hour | Session expires; redirected to login | 🟡 IMPORTANT |
| B-06 | Audit log records login | Log in; check AuditLog | `login` entry with correct actor, IP, timestamp | 🟡 IMPORTANT |
| B-07 | Audit log records failed login | Make one deliberate failed login; check AuditLog | `login_failed` entry present | 🟡 IMPORTANT |
| B-08 | Gabriele cannot access product management | Log in as Gabriele; attempt product admin | 403 Forbidden | 🔴 CRITICAL |

---

### Layer 4 — Contact Form

The contact form is the only publicly writable endpoint in v1. Primary attack surface.

| # | Check | How to verify | Pass criterion | Priority |
|---|---|---|---|---|
| C-01 | CSRF protection | Submit form without valid CSRF token (e.g. via `curl` without token) | 403 Forbidden | 🔴 CRITICAL |
| C-02 | Rate limiting | Submit 20 requests in 10 seconds | Requests beyond limit return 429 | 🟡 IMPORTANT |
| C-03 | Email field validation | Submit `not-an-email` as email | 400 Bad Request | 🟡 IMPORTANT |
| C-04 | XSS via stored submission | Submit `<script>alert(1)</script>` as message; view in Django admin | Content escaped — not executed | 🔴 CRITICAL |
| C-05 | Admin notification does not render raw HTML | Submit HTML tags; check notification email | Plain/escaped text in email | 🟡 IMPORTANT |
| C-06 | No internal error disclosure | Submit malformed content types | 400/422 — no Django stack trace, no path disclosure | 🟡 IMPORTANT |
| C-07 | Data retention defined and documented | Check Privacy Policy and deletion mechanism | Retention period stated; deletion procedure exists | 🔴 CRITICAL (GDPR) |

---

### Layer 5 — Frontend / Next.js

| # | Check | How to verify | Pass criterion | Priority |
|---|---|---|---|---|
| F-01 | No secrets in client-side bundle | `npm run build`; inspect `.next/static/` | No API keys, tokens, credentials in any JS file | 🔴 CRITICAL |
| F-02 | `X-Frame-Options` header | `curl -I https://yourdomain.com \| grep -i x-frame` | `X-Frame-Options: SAMEORIGIN` | 🟡 IMPORTANT |
| F-03 | `X-Content-Type-Options` header | `curl -I https://yourdomain.com \| grep -i x-content` | `X-Content-Type-Options: nosniff` | 🟡 IMPORTANT |
| F-04 | `Referrer-Policy` header | `curl -I https://yourdomain.com \| grep -i referrer` | `Referrer-Policy: strict-origin-when-cross-origin` | 🟢 GOOD PRACTICE |
| F-05 | Content Security Policy header | `curl -I https://yourdomain.com \| grep -i content-security` | CSP header present; at minimum `default-src 'self'` | 🟢 GOOD PRACTICE |
| F-06 | npm audit — no high/critical CVEs | `npm audit --audit-level=high` in `nextjs/` | Zero high or critical vulnerabilities | 🟡 IMPORTANT |
| F-07 | pip audit — no high/critical CVEs | `pip-audit` or `safety check` in Django environment | Zero high or critical vulnerabilities | 🟡 IMPORTANT |

---

### Layer 6 — GDPR & Legal

| # | Check | How to verify | Pass criterion | Priority |
|---|---|---|---|---|
| G-01 | Privacy Policy live in all three locales | Visit `/it/privacy/`, `/en/privacy/`, `/de/datenschutz/` | Real content — not placeholder | 🔴 CRITICAL |
| G-02 | Privacy Policy names the data controller | Read the policy | Full name and contact details of data controller present | 🔴 CRITICAL |
| G-03 | Contact form has consent language | View the contact form | Statement below submit button: "I dati saranno trattati per rispondere alla tua richiesta. [Privacy Policy link]" (localised in all three languages) | 🔴 CRITICAL |
| G-04 | Cookie consent mechanism in place | Visit in incognito window | If site uses only strictly necessary cookies (session, CSRF): a simplified notice is sufficient — document this decision. If any analytics: full consent banner required before any tracking fires. | 🔴 CRITICAL |
| G-05 | Contact form data retention defined | Check Privacy Policy and deletion mechanism | Retention period stated (e.g. 12 months); deletion procedure documented and implemented | 🔴 CRITICAL |
| G-06 | No third-party tracking before consent | Inspect network requests on first visit (before consent) | No calls to Google Analytics, Meta Pixel, or similar | 🔴 CRITICAL |
| G-07 | Gabriele's data access covered | Read Privacy Policy | If Gabriele sees enquiries: his access to data is disclosed, or Owner is stated as sole controller | 🟡 IMPORTANT |

**Note on cookies for v1:** If v1 uses only session cookies (admin auth) and CSRF cookies — both strictly necessary — no consent banner is required under ePrivacy. A simple notice ("questo sito usa solo cookie tecnici necessari") is sufficient. Confirm no analytics are added before launch without revisiting this.

---

## Part 2 — Periodic Checklist

### Monthly

| # | Check | How | Why |
|---|---|---|---|
| P-M-01 | SSL certificate expiry | `openssl s_client -connect yourdomain.com:443 2>/dev/null \| openssl x509 -noout -dates` | Let's Encrypt auto-renews at 30 days; verify it actually happened |
| P-M-02 | Disk usage — image storage | `df -h` on VPS; check `MEDIA_ROOT` size | Gabriele's uploads accumulate; full disk kills the site silently |
| P-M-03 | fail2ban ban log | `sudo fail2ban-client status sshd` and `sudo zgrep 'Ban' /var/log/fail2ban.log*` | Spot unusual IP patterns; elevated bans indicate active probing |
| P-M-04 | Admin audit log spot check | Query last 30 days of AuditLog | Verify only expected actors; flag unexpected logins or content changes |
| P-M-05 | Contact form submissions review | Check stored submissions in admin | Confirms form works; catch abuse patterns early |

---

### Quarterly

| # | Check | How | Why |
|---|---|---|---|
| P-Q-01 | Django dependency audit | `pip-audit` in production environment | CVEs are discovered continuously |
| P-Q-02 | npm dependency audit | `npm audit --audit-level=high` | Same reason |
| P-Q-03 | Admin user account review | List all Django admin users | Confirm only Owner and Gabriele; no unexpected superusers |
| P-Q-04 | TOTP device review | Check `django_otp_totp_totpdevice` table | Confirm active devices match known devices; revoke orphans |
| P-Q-05 | Nginx access log anomaly check | `grep -E " (4[0-9]{2}\|5[0-9]{2}) " /var/log/nginx/access.log \| awk '{print $7}' \| sort \| uniq -c \| sort -rn \| head -20` | Unusual 4xx/5xx spikes on unexpected paths indicate scanning |
| P-Q-06 | Contact form spam check | Review volume and content of submissions | If volume is high: revisit rate limiting or add honeypot field |

---

### Annually

| # | Check | How | Why |
|---|---|---|---|
| P-A-01 | Django version end-of-life check | Check djangoproject.com/download for LTS support dates | Plan upgrades before support ends |
| P-A-02 | Node.js / Next.js version end-of-life check | Check Node.js release schedule | Node 20 LTS ends April 2026; plan migration to Node 22 |
| P-A-03 | **Backup restoration test** | Restore database from latest backup to a test environment; verify data | Untested backups are not backups. This is the one that bites people. |
| P-A-04 | Privacy Policy review | Read current policy; check for regulatory changes in IT/UK/DE | GDPR guidance evolves; new features may have added data flows |
| P-A-05 | Data retention enforcement | Query contact form submissions older than retention period | Delete per policy — do not let it accumulate indefinitely |
| P-A-06 | DigitalOcean firewall audit | Review cloud firewall ruleset | Confirm no ports opened during debugging and forgotten |
| P-A-07 | Security headers review | Run through securityheaders.com | CSP often needs tightening as the site evolves |
| P-A-08 | Self-conducted penetration check | `nikto -h https://yourdomain.com` + manual OWASP Top 10 check | Annual sanity check proportionate to this scale |

---

## Standing Flag — Image Upload Attack Surface

**Applies when:** Alessandro implements image upload for Gabriele's admin access.

- **Risk:** Malicious file uploaded disguised as an image (PHP shell, SVG with embedded JS)
- **Likelihood:** Low — Gabriele is a known trusted user, not a public endpoint
- **Impact:** Medium — if served directly, script execution or stored XSS is possible
- **Required mitigations before Gabriele's uploads go live:**
  1. Validate file type server-side using `python-magic` (reads file headers, not just extension)
  2. Nginx serves `MEDIA_ROOT` as static files only — no CGI, no execution
  3. Uploaded files stored with a randomised filename (no predictable URLs)
  4. SVG files either rejected outright or sanitised before storage

Flag this to the Tech Lead (Patrick) when the image upload ticket is created.
