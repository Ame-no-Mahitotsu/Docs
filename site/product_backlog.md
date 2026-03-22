# Product Backlog — SitoPresepi2
> Written by PO role. Status: approved by Project Owner.
> Date: 2026-03-12
> Project: site

---

## Vision
A localised, extensible website for handcrafted nativity houses. Serves two audiences — B2B shop owners (catalogue + contact) and B2C end consumers (browse + order). Built to grow: payment, customer archive, and loyalty features slot in without rework.

---

## MoSCoW Priority — v1

### MUST HAVE

#### PB-01 — Product Catalogue
As a shop owner, I want to browse a catalogue of nativity houses with sizes, photos, and prices, so that I can evaluate products for resale.
As a consumer, I want to see all available products clearly described, so that I can choose what to buy.

**Acceptance criteria:**
- Products display name, description, size information, photos, price
- Size is expressed in relation to compatible statue proportions
- Catalogue is manageable via the Admin back office
- All text is localisation-ready (Italian, English, German)
- *Blocked by ISS-001 for size data — structure must be ready*

---

#### PB-02 — Contact (Form, Email, Phone)
As a shop owner, I want multiple ways to contact the artisan, so that I can enquire about products and arrange an order.
As a consumer, I want to contact the artisan for any questions not handled by the order flow.

**Acceptance criteria:**
- Contact form submits successfully and notifies the admin
- Email address is displayed and clickable
- Phone number is displayed
- All three are present and functional in all three languages

---

#### PB-03 — Admin Back Office
As an administrator, I want a protected back office, so that I can manage all site content and data without developer help.

**Acceptance criteria:**
- Login protected with password (basic security for v1)
- Admin can manage: product listings, gallery, time-lapse content, site pages, orders
- Usable by a non-technical operator
- Architecture supports security hardening in future

---

#### PB-04 — Localisation Foundation
As any user, I want the site in my language (Italian, English, or German), so that I can use it comfortably.

**Acceptance criteria:**
- Site is fully localisation-ready from day one — not a post-build translation
- Italian is the source language
- English and German are v1 targets
- All UI labels, content fields, and system messages support localisation
- Language switcher accessible from all pages

---

### SHOULD HAVE

#### PB-05 — Showcase Gallery
As a shop owner or consumer, I want to see a gallery of houses already built, so that I can appreciate the quality and craftsmanship.

**Acceptance criteria:**
- Dedicated gallery section with high-quality images
- Optional subsection: houses built for specific notable statuettes
- Gallery manageable via Admin back office
- Images performant across devices

---

#### ~~PB-06 — B2C Cart and Order Flow~~ — **DEFERRED (2026-03-13)**

> Deferred post-v1. Reason: artisan product where personal conversation is the sale. B2C order intent is now expressed via Contact form pre-filled with product name. Gabriele follows up personally. Cart/checkout to be reconsidered once live traffic data shows demand for self-serve ordering. Approved: Owner + Lota + Orchestrator.

~~As a consumer, I want to add products to a cart and place an order, so that I can buy a house without needing to contact the artisan directly.~~

---

### COULD HAVE

#### PB-07 — Time-lapse Section
As any visitor, I want to see a time-lapse of a house being built, so that I can understand the craftsmanship and effort behind each piece.

**Acceptance criteria:**
- Dedicated section for time-lapse media (video and/or image sequences)
- Content manageable via Admin back office
- Works across devices and connection speeds

---

## Out of Scope — v1
| Feature | Reason |
|---------|--------|
| Custom (su misura) order flow | Too time-intensive; handled off-site by design |
| Payment processing | Plugged in post-v1; stub in place |
| Customer accounts / archive | Future phase |
| Loyalty programme | Future phase |
| Wholesale / B2B ordering flow | B2B channel uses catalogue + contact |

---

## Open Issues
| Ref | Impact |
|-----|--------|
| ISS-001 | Blocks final product data for PB-01 — size names and statue height correspondences needed |

---

## Handoff Notes
- **For Architect / Tech Lead:** PB-04 (localisation) and NFR-01 (extensibility) are structural — must be addressed in stack selection, not retrofitted.
- **For UX:** B2B and B2C journeys are distinct. Navigation and flows must serve both without conflating them.
- **For Scrum Master:** PB-01 through PB-04 are the v1 core. PB-05 and PB-06 complete v1. PB-07 is a bonus if capacity allows.
