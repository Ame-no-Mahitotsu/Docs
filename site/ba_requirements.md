# Business Requirements — SitoPresepi2
> Written by BA role. Status: draft — pending PO prioritisation.
> Date: 2026-03-11
> Project: site

---

## Business Context

A website for handcrafted nativity houses (presepi). Products are artisanal, available in standard catalogue sizes. Custom (su misura) orders exist but are handled entirely off-site — the website does not support them.

---

## User Types

### U1 — Shop Owner (B2B)
A retail shop owner who has been visited in person by the artisan. Comes to the site to browse the catalogue and evaluate products for resale. Places orders by contacting the artisan directly (not through the site).

### U2 — End Consumer (B2C)
An individual buyer who discovers the site via word of mouth or internet search. Browses the catalogue, selects products, and places an order through the site. Payment is not processed in v1.

### U3 — Administrator (Owner)
The project owner. A technical professional with development and infrastructure experience. Sole operator of the site for its lifetime. Has full access to all admin functions: product management, content, gallery, orders, user management, site configuration.

### U4 — Artisan (Gabriele)
The artisan whose work the site showcases. Has limited, content-scoped access to the admin panel. Can upload images (gallery, time-lapse, product images), edit localised text content, and view orders and sales figures. Cannot manage products, users, or site configuration. Acts independently within his permitted scope — no owner approval required for content actions.

*Note: U3 and U4 are distinct admin roles. U4 was not defined in the original requirements due to an incorrect assumption (C3) that the admin would be operated by a non-technical user. Corrected 2026-03-13 — see meeting_bar_001.md.*

---

## Functional Requirements

### FR-01 — Product Catalogue
- The site must display a catalogue of nativity houses in standard sizes
- Each product must show: name, description, size information, photos, price
- Size is expressed in relation to compatible statue proportions
- *(Size names and statue height correspondences: see ISS-001 — pending)*

### FR-02 — Showcase Gallery
- A dedicated gallery section displaying houses already built
- Must support high-quality images
- May include a subsection featuring houses built for specific notable statuettes

### FR-03 — Time-lapse Section
- A dedicated section showing the artisan's building process via time-lapse media
- Supports video and/or image sequences

### FR-04 — B2C Cart and Order Flow
- U2 can add catalogue items to a cart
- U2 can proceed to checkout and submit an order
- Checkout collects: name, delivery address, email, product and size selection
- Payment step shows card and PayPal as options (stubbed — no processing in v1)
- U2 receives an order confirmation after submission

### FR-05 — Contact
- The site must provide three contact methods: contact form, email address, phone number
- Contact is the primary action for U1 (B2B enquiries and orders)
- Contact is also available to U2 for general enquiries

### FR-06 — Admin Back Office
- Password-protected administration area
- U3 can manage: product listings, gallery content, time-lapse content, site pages, orders
- Basic security for v1; architecture must support hardening in future

### FR-07 — Localisation
- The site must be built localisation-ready from day one — not translated, but properly localised
- Source language: Italian
- v1 target languages: Italian, English, German
- All content, UI labels, and system messages must be localisation-aware

---

## Non-Functional Requirements

### NFR-01 — Extensibility
The architecture must not block future additions including: payment processing, customer archive, loyalty programme, additional languages.

### NFR-02 — Admin Usability
*(Revised 2026-03-13 — original requirement was based on a false assumption. See BAR-001.)*

The back office must support two distinct user roles with different access scopes:
- The Owner (technical operator) requires full administrative control with no artificial simplifications.
- Gabriele (artisan) requires a focused, limited interface for content upload and sales visibility — sufficient usability for a non-developer performing routine content tasks, but no consumer-grade UX required.

---

## Out of Scope — v1
- Custom (su misura) order flow on the site
- Payment processing
- Customer accounts / archive
- Loyalty programme
- Wholesale pricing or B2B-specific ordering

---

## Open Issues
| Ref | Description |
|-----|-------------|
| ISS-001 | Standard size names and statue height correspondences — needed before product catalogue can be built |

---

## Handoff Notes
- **For Architect / Tech Lead:** Localisation (FR-07) and extensibility (NFR-01) must be considered from day one in stack and data model decisions.
- **For PO:** All features above are requirements — prioritisation and phasing is your call.
- **For UX:** Two distinct user journeys (B2B browse+contact vs B2C browse+order) need separate consideration in the navigation and flow design.
