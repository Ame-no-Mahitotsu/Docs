# Pending: Architecture Review Meeting
**Raised:** 2026-03-12
**Status:** CLOSED — resolved in ARQ-001 (2026-03-13). See meeting_arq_001.md.
**Project:** site

---

## Items to Discuss

### 1. Customer Table — Stub Added
Owner spotted missing `Customer` table in Patrick's data model. Patrick acknowledged the omission — it was deferred because customer accounts are out of scope for v1, but the lack of a stub would cause a painful migration in v2.

**Decision taken:** Add a nullable `Customer FK` stub to `Order` from day one. Guest orders store details inline. Registered customer orders link to `Customer`. Patrick and Dominick need to align on the identity implications.

Revised model:
- `Customer` — email, first_name, last_name, created_at, is_active
- `Order` — customer FK (nullable), guest_email, guest_name, guest_address, items, status, created_at

**Needs:** Dominick to confirm identity domain boundary still holds with this model.

---

### 2. Open Question — What Else Did We Miss?
The Customer table was caught by the owner reviewing the data model. Before the architecture document is finalised, the full team should review Patrick's data model for similar gaps — entities that are out of scope for v1 but whose absence now would cause migration pain later.

Candidates to check:
- **ShopOwner / B2B contact** — do we need a stub for B2B accounts?
- **Payment** — stub for payment transaction record even if processing is deferred?
- **Loyalty / points** — too early, but worth a quick flag
- **Audit log** — Dominick requires this; is it in the data model?

---

### 3. Participants Needed
- Patrick (lead)
- Dominick (identity + security implications)
- Alessandro (data model + migration strategy)
- Owner

Fran (BA) optionally — to validate against requirements.
