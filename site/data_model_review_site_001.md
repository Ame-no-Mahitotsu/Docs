---
ID: data_model_review_site_001
Type: session
Role: tech-lead
Topic: Architecture review — data model before implementation (ISS-101)
Date: 2026-03-23
Participants: Alessandro (Tech Lead) — Owner to sign off
Ticket: ISS-101
Sprint: site-1
Status: open
Project: site
Outputs: Corrected app list for ISS-102, corrected model list for ISS-103, AltText gap resolution, ISS-001 impact on Size model
---

# Data Model Review — Pre-Implementation
**Ticket:** ISS-101
**Reviewer:** Alessandro (Tech Lead)
**Source:** architecture_document.md §5, §6, §6b, §6c

This review must be signed off by the Owner before Claire starts ISS-102 or ISS-103. Its purpose is to catch naming discrepancies, missing models, and scope ambiguities before they land in migrations.

---

## Finding 1 — ISS-102 app list has wrong names and missing apps

ISS-102 lists: `catalogue, gallery, contact, pages, admin_app, customers`

Architecture §5 defines: `core, api, admin_app, catalogue, commerce, customers, gallery, content, localisation`

**Corrections required:**

| ISS-102 (wrong) | Architecture §5 (correct) | Action |
|---|---|---|
| `pages` | `content` | Rename — Page and PageTranslation live here per §6 |
| `contact` | *(not named in arch)* | Approve — logical home for ContactEnquiry; clean boundary |
| *(missing)* | `commerce` | Add — Order, OrderItem, Payment stubs live here |
| *(missing)* | `localisation` | Add — Language model lives here; all translation models depend on it |
| *(missing)* | `api` | Add — public REST views and serializers |
| *(missing)* | `core` | Already scaffolded (ISS-004) — confirm it exists |

**Corrected app list for ISS-102:**
```
catalogue/    — Product, ProductTranslation, ProductImage, ProductSize, Size
gallery/      — GalleryItem, GalleryItemTranslation, TimeLapseItem, TimeLapseItemTranslation
contact/      — ContactEnquiry  [new app — not in architecture, approved here]
content/      — Page, PageTranslation, NavigationItem
commerce/     — Order, OrderItem, Payment  (stubs — no UI flow in v1)
customers/    — Customer stub
localisation/ — Language
admin_app/    — AuditLog, admin views (already partially scaffolded)
api/          — public REST views and serializers (no models)
core/         — settings, base models, middleware, logging config (already scaffolded ISS-004)
```

**Decision needed from Owner:**
- Confirm `contact/` as the app name for ContactEnquiry (alternative: put it in `content/`)
- My recommendation: separate `contact/` app — it has its own rate-limiting, GDPR retention, and email dispatch logic. Mixing it into `content/` violates the single-responsibility boundary.

---

## Finding 2 — ISS-103 model names have three errors

ISS-103 title: `Product, ProductImage, Gallery, GalleryImage, ContactMessage, Page`

| ISS-103 (wrong) | Architecture §6 (correct) |
|---|---|
| `Gallery` | `GalleryItem` |
| `GalleryImage` | `GalleryItemTranslation` |
| `ContactMessage` | `ContactEnquiry` |

These are not cosmetic — wrong names become wrong migration files, wrong serializer class names, wrong test names. They must be corrected before Claire starts.

---

## Finding 3 — ProductImage.alt_text is undefined in the architecture

Architecture §6 states: `ProductImage.alt_text — Localised, see AltTextTranslation`

There is no `AltTextTranslation` model defined anywhere in the architecture document. This is a gap.

**Options:**
- **Option A** — `ProductImageAltText(product_image FK, language FK, alt_text CharField)` — full localised model, consistent with the translation pattern used everywhere else.
- **Option B** — Single `alt_text = CharField` on `ProductImage` in v1. No translation. Add `ProductImageAltText` in v2.

**My recommendation: Option B for v1.**

Rationale: alt text is an accessibility field, not a primary content field. In v1 the content is authored by the Owner in Italian; the site publishes English and German too, but alt text translation is not a launch blocker. Option A adds a join for every image load with no v1 benefit. The field is easy to replace with a FK in v2.

**Decision needed from Owner.**

---

## Finding 4 — ISS-001 impact on the Size model

ISS-001 resolved (2026-03-20): dimensions confirmed as max 25h × 30w × 15d cm. Size display names deferred to content phase.

Architecture §6 defines `Size` with a `code` field (`S`, `M`, `L`, `XL`).

With size names deferred, the `code` field is a machine key only in v1 — it is not displayed to users. Claire should implement it as a `CharField(max_length=10, unique=True)` with no translation model. Display names will be added as a `SizeTranslation` model in a future ticket when Gabriele confirms them.

**No decision needed — this is a clarification for Claire.**

---

## Finding 5 — commerce/ and localisation/ are in scope for ISS-102 scaffold but NOT ISS-103 models

The architecture requires `Order`, `OrderItem`, `Payment`, and `Customer` to exist as v2 seams from day one (C7). However, building full models for these in site-1 when no UI exists yet would be premature. The correct approach:

- **ISS-102** (scaffold): create all app directories including `commerce/`, `localisation/`, `customers/`. Each gets `__init__.py`, empty `models.py`, empty `tests/` directory.
- **ISS-103** (core models): implement `Language` (required as FK by everything else), then `catalogue/`, `gallery/`, `contact/`, `content/` models only.
- `commerce/`, `customers/` stubs (Order, OrderItem, Payment, Customer): **separate ticket, site-2 or site-3** — they are not needed until the contact API (ISS-113) and they add migration complexity with no v1 payoff.

**Exception:** `Customer` is referenced as a nullable FK on `Order`. Since Order is deferred, Customer stub can also be deferred. The seam exists at the app level (directory created in ISS-102); the model comes later.

**Decision needed from Owner.**

---

## Summary — Decisions Required from Owner

| # | Decision | My recommendation |
|---|---|---|
| D1 | `contact/` as standalone app vs inside `content/` | Standalone `contact/` app |
| D2 | ProductImage alt_text: full translation model now vs single CharField v1 | Single CharField, translate in v2 |
| D3 | commerce/customers stubs in ISS-103 or deferred | Defer to site-2; app directories in ISS-102 only |

**Clarifications (no decision needed):**
- ISS-102 app list: use corrected list above
- ISS-103 model names: use `GalleryItem`, `GalleryItemTranslation`, `ContactEnquiry`
- Size.code: machine key only in v1, no translation model yet

---

## Corrected Implementation Brief for Claire (pending Owner sign-off)

### ISS-102 — Revised scope
Create the following Django app directories (each with `models.py`, `tests/` dir, `apps.py`, `__init__.py`):
```
catalogue/
gallery/
contact/
content/
commerce/
customers/
localisation/
api/
```
`core/` and `admin_app/` already exist from ISS-004 — verify, do not recreate.

### ISS-103 — Revised model list (site-1 only)

**`localisation/` — implement first, everything else depends on it**
```python
Language(code, name, is_active, is_default)
```

**`catalogue/`**
```python
Product(id UUID, price Decimal, is_active bool, created_at, updated_at)
ProductTranslation(product FK, language FK, name, description, slug unique-per-lang)
Size(id UUID, code CharField unique, statue_height_min_mm int, statue_height_max_mm int)
ProductSize(product FK, size FK, stock_status choices[available/made_to_order/unavailable])
ProductImage(product FK, image ImageField, alt_text CharField, display_order int)
```
Note on `ProductImage.alt_text`: single CharField per D2 decision above.
Note on `Size.code`: machine key only — no display name in v1.

**`gallery/`**
```python
GalleryItem(id UUID, image ImageField, display_order int, is_featured bool, created_at)
GalleryItemTranslation(gallery_item FK, language FK, title, description)
TimeLapseItem(id UUID, media_type choices[video/image_sequence], media_url URLField, display_order int, is_active bool)
TimeLapseItemTranslation(time_lapse_item FK, language FK, title, description)
```

**`contact/`**
```python
ContactEnquiry(
    id UUID,
    name CharField,
    email EmailField,
    message TextField,
    product_interest CharField nullable,
    locale CharField choices[it/en/de],
    created_at DateTimeField auto_now_add,  # GDPR retention trigger — §15.3
    is_archived BooleanField default False
)
```
⚠ GDPR: `created_at` is the retention trigger. Deletion mechanism required before launch (C-07).

**`content/`**
```python
Page(id UUID, slug SlugField, is_active bool)
PageTranslation(page FK, language FK, title CharField, body JSONField, meta_description CharField)
```

**`admin_app/` — AuditLog (already partially scaffolded, verify exists)**
```python
AuditLog(id UUID, actor FK→User, action CharField, target_model CharField,
         target_id UUIDField nullable, ip_address, timestamp auto_now_add)
```
Append-only enforcement: override `save()` and `delete()` to raise `NotImplementedError`.

---

## Owner Sign-Off

Please confirm or redirect on D1, D2, and D3 above. Once signed off, ISS-101 resolves and Claire starts ISS-102.
