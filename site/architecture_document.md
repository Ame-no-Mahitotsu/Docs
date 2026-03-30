# SitoPresepi2 — Formal Architecture Document
**Author:** Patrick (Solution Architect)
**Date:** 2026-03-12
**Status:** APPROVED — 2026-03-13 (ARQ-001). Supersedes all previous drafts.
**Project:** site

---

## 1. Purpose and Scope

This document defines the technical architecture of SitoPresepi2 — a multilingual e-commerce and showcase website for handcrafted nativity houses (presepi) by Gabriele.

It is the authoritative technical reference for all team members. Any deviation from this document requires the owner's explicit approval and must be recorded in `constitution.md`.

---

## 2. Governing Constraints

These constraints were set before any technology choices and drive all decisions below:

| # | Constraint | Origin |
|---|---|---|
| C1 | No monolith — every layer must be independently replaceable | Owner |
| C2 | Localisation-ready from day one (IT / EN / DE) | BA requirement |
| C3 | Admin operated by the technical project owner for the lifetime of the project. No non-technical operator assumed. | Owner (revised 2026-03-13 — original BA assumption was incorrect; artisan is content subject, not operator) |
| C4 | No credentials in code — `.env` only | Constitution §8 |
| C5 | GDPR compliance before launch — IT, UK, DE jurisdictions | Constitution §9 |
| C6 | TDD — tests written before implementation, no deferrals | Constitution §6 |
| C7 | No cart/checkout UI in v1 — orders via phone/contact form (D2, approved 2026-03-14). Order/Payment models exist as database seams for v2. No provider lock-in. | PO decision + Owner (D2) |

---

## 3. Technology Stack

| Layer | Technology | Version target | Reason |
|---|---|---|---|
| API | Django + Django REST Framework | Django 5.x, DRF 3.x | Battle-tested, Python ecosystem, clean REST, pytest-django support |
| Frontend | Next.js | 14.x (App Router) | SSG with revalidation for SEO, React ecosystem, strong i18n support |
| Admin app | Custom Django app | — | Isolated from public API; narrow surface for artisan; no third-party admin framework dependency |
| Database | PostgreSQL | 16.x | Django-native, reliable, JSONB available if needed |
| Reverse proxy | Nginx | Latest stable | Single entry point, SSL termination, rate limiting, CORS enforcement |
| Containerisation | Docker + Docker Compose | Latest stable | Process isolation, reproducible environment, single-VPS deployment |
| Image storage | Local filesystem + django-storages | — | No object storage cost at this scale; abstraction layer means migration is one config change |
| Auth (admin) | Django session auth + django-otp (TOTP) | — | Two-factor, short session lifetime, isolated from customer domain |
| Auth (customer, future) | SimpleJWT | — | Stateless, separate from admin domain |
| Testing (Django) | pytest + pytest-django | — | TDD contract |
| Testing (Next.js) | Jest + React Testing Library | — | TDD contract |
| Hosting | Hetzner Cloud CPX21, Nuremberg/Falkenstein (DE) | 3 AMD vCPU / 4 GB RAM / 80 GB NVMe | German GmbH entity, ISO 27001, GDPR, ~€11.30/month all-in. Supersedes DigitalOcean Frankfurt (2026-03-12). See `docs/site/vps_provider_comparison.md`. |

**Rejected alternatives (not to be reopened without new evidence):**

- Django + Wagtail: rejected because integrated CMS creates a monolith that violates C1. Good tool, wrong constraint.
- Headless CMS (Strapi/Payload): two systems to maintain, higher operational complexity for a one-developer project.
- WordPress: plugin ecosystem dependency, hard to evolve cleanly.
- Laravel + Filament: valid alternative, ranked second — ruled out in favour of Python ecosystem consistency.

---

## 4. Deployment Topology

```
Internet
    │
  [ Nginx ]  ← Only public entry point
  port 80 → redirect to 443
  port 443 → SSL/TLS termination, rate limiting, CORS
    │
    ├──► [ Next.js ]         port 3000 (internal Docker network only)
    │       Public site — IT/EN/DE
    │       SSG with revalidation from Django API
    │
    └──► [ Django ]          port 8000 (internal Docker network only)
              ├── /api/v1/         ← public REST API (read-only, no auth)
              ├── /gestione/       ← admin app (session + TOTP auth)
              └── internal DB calls only
                      │
              [ PostgreSQL ]       port 5432 (internal Docker network only)
                                   never exposed outside container network
```

**Docker Compose services:**

```
docker-compose.yml
├── nginx          (ports 80, 443 → host)
├── django         (port 8000 → internal only, gunicorn)
├── nextjs         (port 3000 → internal only, node)
└── postgres       (port 5432 → internal only)
```

**Key topology rules:**
- Django and Next.js are never reachable directly from the internet — only via Nginx.
- PostgreSQL is never reachable outside the Docker internal network.
- Admin app (`/gestione`) is behind Nginx but served by the same Django container — it does **not** go through the public API.
- Secrets via `.env` file on server only — never in the repository.

---

## 5. Django Application Structure

```
django/
├── core/           ← shared: settings, base models, auth helpers, middleware, logging config
├── api/            ← public REST API views and serializers (read-only)
├── admin_app/      ← owner control panel (separate auth, TOTP, audit log, application logs)
├── catalogue/      ← Product, ProductImage, Size models [⚠ image upload: see §15]
├── commerce/       ← Order, OrderItem, Payment stub (v1: models only — no UI flow; cart deferred to v2)
├── customers/      ← Customer stub (v1 data only; Customer auth added here in v2)
├── gallery/        ← GalleryItem, TimeLapseItem models [⚠ image upload: see §15]
├── content/        ← Page, NavigationItem, localised text snippets
└── localisation/   ← Language, TranslatedField infrastructure
```

**Boundary rules:**
- `api/` may import from `catalogue`, `commerce`, `customers`, `gallery`, `content`. It may not import from `admin_app`.
- `admin_app/` may import from all domain apps via their service functions. It does not expose any URL under `/api/`.
- `customers/` owns all customer identity logic. `commerce/` imports from `customers/` — never the reverse.
- `customers/` may never import from `admin_app/`. `admin_app/` accesses customer data via `customers/` service functions only.
- No circular imports between domain apps.
- `core/` is the only place shared infrastructure lives — no sharing via other apps.

---

## 6. Data Model

### Localisation Pattern

Text fields that require translation use a **TranslatedField** approach: a related table holds one row per language per object. This means:

- Adding a language = adding rows, not changing the schema.
- No nullable `name_it`, `name_en`, `name_de` columns polluting every model.
- Consistent pattern across all translatable content.

```
Product (id, slug, price, created_at, updated_at)
    └── ProductTranslation (product FK, language FK, name, description)
```

This pattern applies to all content entities below.

---

### Entities

**Product**

| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| slug | SlugField | Language-neutral; unique; used in all API and frontend URLs |
| price | DecimalField | Euro cents stored as decimal |
| is_active | BooleanField | Soft visibility toggle |
| created_at | DateTimeField | Auto |
| updated_at | DateTimeField | Auto |

**ProductTranslation**

| Field | Type | Notes |
|---|---|---|
| product | FK → Product | |
| language | FK → Language | |
| name | CharField | |
| description | TextField | |

**ProductImage**

| Field | Type | Notes |
|---|---|---|
| product | FK → Product | |
| image | ImageField | Stored via django-storages |
| alt_text | CharField | Localised — see AltTextTranslation |
| display_order | PositiveIntegerField | For image carousel ordering |

**Size**

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| code | CharField | e.g. `S`, `M`, `L`, `XL` — pending ISS-001 |
| statue_height_min_mm | IntegerField | Compatible statue range |
| statue_height_max_mm | IntegerField | Compatible statue range |

**ProductSize** (join)

| Field | Type | Notes |
|---|---|---|
| product | FK → Product | |
| size | FK → Size | |
| stock_status | CharField | `available`, `made_to_order`, `unavailable` |

**GalleryItem**

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| image | ImageField | |
| display_order | PositiveIntegerField | |
| is_featured | BooleanField | For notable statuette subsection |
| created_at | DateTimeField | |

**GalleryItemTranslation**

| Field | Type | Notes |
|---|---|---|
| gallery_item | FK → GalleryItem | |
| language | FK → Language | |
| title | CharField | |
| description | TextField | |

**TimeLapseItem**

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| media_type | CharField | `video` or `image_sequence` |
| media_url | URLField | External (YouTube/Vimeo) or internal path |
| display_order | PositiveIntegerField | |
| is_active | BooleanField | |

**TimeLapseItemTranslation**

| Field | Type | Notes |
|---|---|---|
| time_lapse_item | FK → TimeLapseItem | |
| language | FK → Language | |
| title | CharField | |
| description | TextField | |

**Customer** *(stub — v1 nullable FK, accounts added in v2)*

| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| email | EmailField | Unique |
| first_name | CharField | |
| last_name | CharField | |
| is_active | BooleanField | |
| created_at | DateTimeField | |

*In v2: authentication credentials and address book added here. Identity domain boundary (Dominick) maintained — Customer auth is separate from Admin auth.*

**ContactEnquiry** *(v1 primary conversion record — D2, 2026-03-14)*

The contact form is the primary buyer intent record in v1. No online ordering exists.

| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| name | CharField | Submitted by visitor |
| email | EmailField | Submitted by visitor |
| message | TextField | Free text — escaped on output (C-04) |
| product_interest | CharField | nullable — pre-filled from product page CTA |
| locale | CharField | `it`, `en`, `de` — language of the form at submission |
| created_at | DateTimeField | auto_now_add — **GDPR retention field**: retention period defined in Privacy Policy; deletion mechanism required before launch (C-07, G-05) |
| is_archived | BooleanField | default False — set True when deleted per retention policy; hard delete scheduled after archival period |

*⚠ GDPR note: `created_at` is the trigger for retention enforcement. A management command or scheduled job must delete rows older than the defined retention period. This is a launch blocker (C-07).*

---

**Order** *(v1: model exists as v2 seam — no UI flow in v1; no cart, no checkout)*

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| customer | FK → Customer | **nullable** — null = guest order |
| guest_name | CharField | nullable — used when customer FK is null |
| guest_email | EmailField | nullable — used when customer FK is null |
| guest_address | TextField | nullable — used when customer FK is null |
| status | CharField | `pending`, `confirmed`, `shipped`, `cancelled` |
| language | FK → Language | Language of the browser at order time |
| created_at | DateTimeField | |
| updated_at | DateTimeField | |

**Payment** *(stub — seam for v2 payment processing; no provider integration in v1)*

| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| order | FK → Order | |
| provider | CharField | nullable — enum: `stripe`, `paypal`, `manual`; null in v1 |
| status | CharField | `pending`, `completed`, `failed`, `refunded` |
| amount | DecimalField | Amount at time of payment attempt |
| created_at | DateTimeField | |

*No payment processing logic in v1. The stub exists so v2 can add a provider without a schema redesign. Governed by architecture constraint C7.*

**OrderItem**

| Field | Type | Notes |
|---|---|---|
| order | FK → Order | |
| product | FK → Product | |
| size | FK → Size | |
| quantity | PositiveIntegerField | |
| unit_price_at_order | DecimalField | Price snapshot — never recalculated from live product |

**Language**

| Field | Type | Notes |
|---|---|---|
| code | CharField | `it`, `en`, `de` |
| name | CharField | `Italiano`, `English`, `Deutsch` |
| is_active | BooleanField | Toggle to enable/disable a language |
| is_default | BooleanField | Exactly one default (Italian) |

**Page**

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| slug | SlugField | Language-independent identifier |
| is_active | BooleanField | |

**PageTranslation**

| Field | Type | Notes |
|---|---|---|
| page | FK → Page | |
| language | FK → Language | |
| title | CharField | |
| body | JSONField | Structured JSON blocks — typed block schema; see §6 Block Types |
| meta_description | CharField | SEO |

---

## 6b. Infrastructure Models

### AuditLog
Lives in `admin_app/`. **Append-only** — `save()` and `delete()` raise exceptions if called; only `AuditLog.objects.create(...)` is permitted. DB-level: Django user has no `UPDATE` or `DELETE` privilege on this table.

| Field | Type | Notes |
|---|---|---|
| id | UUID | Primary key |
| actor | FK → Django admin user | Who performed the action |
| action | CharField | Enum: `login`, `logout`, `login_failed`, `content_created`, `content_updated`, `content_deleted`, `order_updated` |
| target_model | CharField | e.g. `Product`, `Order`, `Page` |
| target_id | UUIDField | nullable — null for auth events |
| ip_address | GenericIPAddressField | |
| timestamp | DateTimeField | auto_now_add — never editable |

### Application Logging

Separate from the security AuditLog. Built on Python's `logging` module — no additional library. Controlled by `LOG_LEVEL` environment variable (`ERROR` | `WARNING` | `INFO` | `DEBUG`). Set in `.env` — never hardcoded.

**Logger namespaces:**

| Logger | Covers |
|---|---|
| `sitopresepi.db` | Database connection events, query anomalies |
| `sitopresepi.catalogue` | Product and size data access |
| `sitopresepi.commerce` | Order and payment flow |
| `sitopresepi.customers` | Customer data access (v2: auth events) |
| `sitopresepi.admin` | Admin panel actions (operational — not the security audit log) |
| `sitopresepi.api` | Public API requests and errors |
| `sitopresepi.content` | Page and content serving |

Each logger writes to a rotating file handler. `LOG_LEVEL=ERROR` in production by default; set `DEBUG` locally or to diagnose a specific subsystem. New subsystems add a new namespace — no changes to existing loggers.

---

## 6c. Page Body — Block Types

`PageTranslation.body` is a `JSONField` containing an ordered list of typed blocks. Allowed block types are an enumerated set — the admin editor renders only these types; freeform HTML is not permitted.

**Defined block types (v1):**

```json
[
  { "type": "heading",   "level": 2, "text": "Chi siamo" },
  { "type": "paragraph", "text": "Gabriele realizza presepi..." },
  { "type": "image",     "src": "/media/laboratorio.jpg", "caption": "Il laboratorio", "alt": "..." },
  { "type": "divider" },
  { "type": "cta",       "text": "Scopri il catalogo", "href": "/it/catalogo/" }
]
```

New block types require a schema extension approved by the owner. Old content is unaffected when new types are added.

---

## 7. API Endpoint Map (v1)

All endpoints under `/api/v1/`. All are **read-only** (GET) unless noted. Authentication not required for public endpoints.

### Catalogue

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/products/` | List active products — language via `Accept-Language` header or `?lang=` |
| GET | `/api/v1/products/{slug}/` | Single product detail |
| GET | `/api/v1/sizes/` | All sizes (reference data) |

### Gallery

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/gallery/` | All gallery items |
| GET | `/api/v1/gallery/featured/` | Featured items (notable statuettes subsection) |

### Time-lapse

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/timelapse/` | All time-lapse items |

### Orders *(deferred — v2; no UI flow in v1)*

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/orders/` | ~~Submit an order (checkout)~~ — **deferred to v2 (D2)** |
| GET | `/api/v1/orders/{id}/` | ~~Order confirmation~~ — **deferred to v2 (D2)** |

### Contact *(primary conversion endpoint in v1)*

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/contact/` | Submit a contact enquiry — rate limited; accepts optional `product_interest` field pre-filled from product CTA |

### Pages / Content

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/pages/{slug}/` | Localised page content |

### Admin endpoints

Not public API — served under `/gestione/` directly by `admin_app`. Session + TOTP gated. Full CRUD for all content, orders, and site settings.

---

## 8. Localisation Architecture

- **Language selection:** `Accept-Language` header from browser, overridable via `?lang=it|en|de` query parameter or persistent session cookie.
- **URL strategy:** Language-prefixed paths on the Next.js side — `/it/catalogo/`, `/en/catalogue/`, `/de/katalog/`. Django API is language-agnostic; language is a parameter.
- **Fallback:** If a translation is missing for the requested language, fall back to Italian (default). Never return empty content — always surface the fallback.
- **New language addition:** Activate a Language row, add translation rows. Zero schema changes.
- **ISS-001 note:** Size codes and names are pending Gabriele's input. Do not build the size catalogue until ISS-001 is resolved.

---

## 9. Rendering Strategy (Next.js)

| Page type | Strategy | Reason |
|---|---|---|
| Product list | SSG + `revalidate: 3600` | Changes infrequently; SEO critical |
| Product detail | SSG + `revalidate: 3600` | Same as above |
| Gallery | SSG + `revalidate: 3600` | Static content |
| Time-lapse | SSG + `revalidate: 3600` | Static content |
| ~~Cart / Checkout~~ | ~~Client-side~~ | **Deferred — v2 (D2)** |
| ~~Order confirmation~~ | ~~Client-side~~ | **Deferred — v2 (D2)** |
| Contact | SSG shell + client form | Primary conversion endpoint; accepts pre-filled product interest |
| Chi sono / The Maker | SSG + `revalidate: 3600` | Artisan identity page — P-06b |
| Pages (Home etc.) | SSG + `revalidate: 3600` | Content-managed, SEO critical |

**Cart:** Deferred to v2. No cart state, no localStorage cart management in v1. B2C journey ends at Contact form.

---

## 10. Security Architecture

Full detail in Dominick's security baseline ([session_security_dominick_001_baseline.md](session_security_dominick_001_baseline.md)). Summary of non-negotiables:

| Layer | Requirement |
|---|---|
| Network | Nginx only entry point; HTTP → HTTPS redirect; fail2ban on VPS |
| Admin identity | Session + TOTP (all users including artisan); role-based inactivity: Owner = 1h (`ADMIN_SESSION_TIMEOUT_OWNER`), Gabriele = 2h (`ADMIN_SESSION_TIMEOUT_ARTISAN`); both env-configurable; 5 failures → 15min IP lock; non-obvious URL `/gestione` |
| Admin RBAC | Two roles: Django superuser (Owner — full access) + `artisan` Django Group (Gabriele — image upload, text edit, read-only orders/sales). No third role. |
| Customer identity | No customer accounts in v1; SimpleJWT seam ready for v2 |
| API | Public = read-only; write endpoints (orders, contact) rate-limited; CORS locked to Next.js origin |
| Data | PostgreSQL minimum-privilege user; no sensitive data in logs |
| Admin session cookie | HttpOnly, Secure, SameSite=Strict |
| Passwords | PBKDF2-SHA256 (Argon2 available via one-line config change) |
| Credentials | Never in code — `.env` on server only |
| GDPR | Privacy policy + data retention policy + cookie consent required before any production deployment |

---

## 11. Testing Contract

| Scope | Framework | Rule |
|---|---|---|
| Django models | pytest + pytest-django | Written before the model |
| Django API endpoints | pytest + pytest-django | Test defines contract before view exists |
| Django business logic | pytest + pytest-django | Test before implementation |
| Auth flows | pytest + pytest-django | Login, fail, lockout, TOTP, session expiry |
| Order flow | pytest + pytest-django | Each step in isolation + end-to-end |
| Localisation | pytest + pytest-django | Every content endpoint tested for all 3 languages |
| Next.js components | Jest + React Testing Library | Written before the component |
| Next.js user journeys | Jest + RTL | B2C browse → cart → checkout |
| API contract | Shared schema + integration test | Django and Next.js agree on response shapes |

**Minimum viable test:** The smallest test that proves the behaviour is correct. Not exhaustive edge cases upfront. No test deferrals.

---

## 12. Open Items — Blocking

| Ref | Blocker | Owner | Blocks |
|---|---|---|---|
| ISS-001 | Standard size names and statue height correspondences | Gabriele (via Owner) | Product catalogue model, size API |
| UX-PENDING | Gabriele's answers to Lota's 5 questions | Gabriele (via Owner) | Visual identity, design direction |
| GDPR | Privacy policy, data retention policy, cookie consent | Fran + Owner | Production launch |
| ~~TBD~~ | `Page.body` format — **resolved: structured JSON blocks** (see §6c) | ~~Owner + team~~ | Resolved ARQ-001 |
| ~~TBD~~ | Frontend CSS approach — **resolved: Tailwind CSS + tailwind.config.ts** | ~~Owner + Ash~~ | Resolved ARQ-001 |

---

## 13. Constraints for All Team Members

**Frontend (Ash):**
- No inline styles — all styling via Tailwind CSS. Design tokens in `tailwind.config.ts`. Resolved ARQ-001.
- Language switching must be instantaneous — no full page reload.
- No cart state in v1. Cart/checkout deferred (D2). Product CTA routes to Contact with pre-filled product name.
- No direct database access — Next.js reads from Django API only.

**Backend:**
- Every new endpoint gets a test written first — the test file is the specification.
- No business logic in views — views call service functions, service functions are tested independently.
- `admin_app` never calls public API endpoints — it calls service functions directly.
- `unit_price_at_order` on `OrderItem` is set at order creation and never updated. No live price recalculation after the fact.

**Admin app:**
- Content blocks are constrained to the defined block type enum (§6c). Freeform HTML is not permitted — this is a maintainability and consistency rule, not a user-protection rule. The operator is technical; the constraint exists to keep the rendering layer predictable.
- All admin actions must be logged via `AuditLog` (§6b). Security events via the security audit log; operational events via the `sitopresepi.admin` logger.

**Everyone:**
- No third-party service integrations without owner approval and a Registered Concern review.
- No new Python or npm package added without noting it and its purpose in the relevant session file.
- Any deviation from this document must be raised with the owner before implementation.
- When AI assistance is used to generate implementation code, a failing test must exist before the generated code is accepted. AI output is not exempt from the TDD requirement (Constitution §6).

---

## 14. What Comes Next

**Completed (2026-03-13 / 2026-03-14):**
- ~~**Dominick** — Nginx config reference + Docker Compose skeleton~~ — done: `nginx/nginx.conf`, `docker-compose.yml`, `.env.example`
- ~~**Patrick** — Formal architecture document~~ — done: this document
- ~~**Patrick / Ash** — BAR-001 C3 review of §2 and §13~~ — done
- ~~**Patrick + Alessandro + Dominick** — Dev environment strategy~~ — done: WRK-001; brief in `session_techlead_alessandro_003_scaffold.md`
- ~~**Alessandro** — Django project scaffold + test runner setup~~ — done: merged to `develop` 2026-03-13
- ~~**Ash** — Next.js project scaffold + i18n routing setup~~ — done: merged to `develop` 2026-03-13 (feature/nextjs-scaffold); 2 bugs fixed in PR review
- ~~**Dominick** — Security checklist (pre-launch and periodic)~~ — done: `shared/security_checklist.md`; §15 added to this document (2026-03-14)
- ~~**D2 scope change**~~ — cart/checkout deferred from v1; contact-first model approved (Owner, 2026-03-14)
- ~~**Lota** — IA pages list + slug map + navigation spec~~ — in progress: D1–D4 approved; mood boards and The Maker page spec delivered (session_ux_lota_001_ia.md)

**Remaining, in recommended order:**
1. **Patrick** — Final data model as Django migrations (after ISS-001 resolved for sizes)
2. **Lota** — Mood board direction owner approval → design tokens → component spec
3. **Gabriele (via Owner)** — ISS-001 size names; 5 visual questions for Lota; interview answers for The Maker page

**Blocked:**
- ISS-001: size names and statue height correspondences — needed before product catalogue models can be built (Gabriele via Owner)
- UX visual direction final approval — pending Gabriele's answers to Lota's 5 questions
- The Maker page content — pending Gabriele interview (questions ready in ISS-007)

---

---

## 15. Security Constraints (Architectural — Non-Negotiable)

*Source: Dominick's security checklist (`shared/security_checklist.md`), approved 2026-03-14. These constraints apply to all implementation work. They are not implementation details — they are architectural requirements.*

### 15.1 Image Upload (catalogue/ and gallery/ apps)

Any ticket that involves file upload handling — `ProductImage`, `GalleryItem`, or any future media model — must implement all four of the following before the feature is accepted:

| # | Constraint | Why |
|---|---|---|
| U-01 | Server-side MIME validation using `python-magic` (reads file headers, not just extension) | Extension spoofing is trivial; header validation is not |
| U-02 | SVG files rejected outright | SVG can embed JavaScript; stored XSS risk even from a trusted user |
| U-03 | Uploaded filename replaced with a UUID before storage | Predictable filenames enable targeted access and information disclosure |
| U-04 | Nginx serves `MEDIA_ROOT` as static files only — no CGI, no execution | Belt-and-suspenders: even if validation fails, files are not executed |

*Gate: Alessandro (Tech Lead) must confirm U-01 through U-04 are in the implementation brief before any image upload ticket is picked up.*

### 15.2 Contact Form (api/ app — ContactEnquiry model)

The contact form is the only publicly writable endpoint in v1. Security checklist items C-01 through C-07 are mandatory. The ticket brief for the contact form endpoint must explicitly reference `shared/security_checklist.md §Layer 4`.

Critical items (🔴):
- **C-01**: CSRF protection — Django REST Framework enforces; confirm not bypassed
- **C-04**: XSS via stored submission — escape on output in admin views
- **C-07 / G-05**: GDPR data retention — retention period in Privacy Policy + deletion mechanism before launch

### 15.3 GDPR — ContactEnquiry Retention

`ContactEnquiry.created_at` is the GDPR retention trigger. A management command or scheduled process must delete rows older than the defined retention period. This is a **launch blocker** — the site does not go live without this mechanism in place and documented in the Privacy Policy.

---

*This document reflects decisions approved by the Project Owner through 2026-03-14.*
