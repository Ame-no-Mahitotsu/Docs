# Testing Strategy — Site Track
> Created: 2026-03-17 | Author: Lauren (SDET)
> Project: site
> Status: draft — G5 approved, awaiting G6 review (Rich)
> Ticket: ISS-063

---

## Scope

This document covers all testing for the **site track** (`site-N` sprints): the Django REST Framework API, the Next.js frontend, the PostgreSQL database layer, and all user-facing pages. It is the gate-blocking prerequisite for site-1 opening.

The tooling track testing strategy is a separate document (`strategy_testing_tooling_001.md`) and is not repeated here.

---

## Stack Under Test

| Layer | Technology | Test tooling |
|---|---|---|
| API | Django 5.2.x (>=5.2.8) + DRF 3.x, Python 3.14.3 | pytest + pytest-django |
| Frontend | Next.js 14.x App Router, TypeScript strict | Jest + React Testing Library |
| E2E | Full stack (browser → API → DB) | Playwright |
| Accessibility | WCAG 2.1 AA | Playwright + axe-core (via @axe-core/playwright) |
| Database | PostgreSQL 16.x | pytest-django (test DB, not production) |
| Containers | Docker + Docker Compose | All tests run inside containers |

---

## Test Responsibility Matrix

| Test type | Who writes | Who owns | Runs in |
|---|---|---|---|
| Unit — Django models/services | Developer (Claire/backend) | Developer | Docker `test` service |
| Unit — Next.js components | Developer (Ash/frontend) | Developer | Docker `test` service |
| Integration — API endpoints | Developer + Lauren shared | Lauren owns gate | Docker `test` service |
| E2E — user journeys | Lauren | Lauren | Playwright container |
| Accessibility audits | Lauren | Lauren | Playwright + axe run |
| Exploratory / acceptance | Rich (QA) | Rich | Staging environment |

**Boundary rule:** Unit tests prove the developer's intention. Integration and E2E tests prove the system does what users and stakeholders need. When a defect is found in E2E that should have been caught at unit level, Lauren flags it to the developer as a testability coaching point — not just a bug report.

---

## Python / Django Testing

### Environment

All Django tests run inside the dedicated `test` Docker service (profile-gated):

```bash
docker compose --profile test run --rm test
```

This is the canonical invocation. Never run Django tests against the dev database.

### Framework

- **pytest** + **pytest-django** — approved 2026-03-12 (Constitution decisions log)
- **pytest-cov** — approved 2026-03-14, security conditions apply (see tooling strategy)

### `conftest.py` structure

Two levels, scaffolded by Alessandro at site-1 day one:

```
django/conftest.py                    # project-level: DB setup, shared fixtures
<app_name>/tests/conftest.py          # app-level: app-specific fixtures only
```

No project-root `conftest.py` for the site track.

### Test location

```
<app_name>/
└── tests/
    ├── conftest.py
    ├── test_models.py
    ├── test_services.py
    └── test_views.py
```

### Naming convention

```
test_<subject>_<condition>_<expected_result>
```

Examples:
- `test_product_with_invalid_slug_returns_404`
- `test_contact_form_missing_email_rejects_with_400`
- `test_audit_log_on_image_upload_appends_entry`

### Coverage gate

**80% line coverage minimum** across all Django apps (`--cov-fail-under=80`).

Coverage below 80% → PR cannot merge. No deferrals.

### What NOT to test

- Django ORM internals
- Admin UI rendering (Django's own test suite covers that)
- Serializer field labels (test that the correct data is returned, not the field name string)

---

## Next.js / Frontend Testing

### Framework

- **Jest** + **React Testing Library** — approved 2026-03-12
- **TypeScript strict mode** — all test files in `.tsx` / `.ts`

### Test location

Co-located with the component:

```
app/
└── components/
    ├── ProductCard.tsx
    └── ProductCard.test.tsx
```

### What to test per component type

| Component type | Test what |
|---|---|
| Display component | Renders without crash; correct content from props; i18n strings present |
| Interactive component | User events trigger correct handlers; state changes correctly |
| Page component | Correct layout rendered; no missing required data fields; locale routing resolves |
| API data consumer | Mocked API returns correct shape; loading and error states handled |

### What NOT to test

- CSS classes or visual appearance
- Pixel layout
- Internal implementation details (hooks internal state)
- `lib/types.ts` type correctness (TypeScript compiler covers this)

### Coverage gate

No numeric coverage gate for frontend. **Behaviour coverage** is the bar: every user-visible state must have a test. Lauren reviews in G5/G6 — if a component has no test, the PR does not merge.

---

## E2E Testing — Playwright

### Framework decision

**Playwright** — preferred for this stack (Next.js App Router + Django REST). Reasons:
- First-class TypeScript support, consistent with frontend codebase
- Built-in accessibility testing via `@axe-core/playwright`
- Handles App Router navigation and server component hydration correctly
- Parallel test execution, trace viewer, screenshot on failure

**Playwright version:** To be locked at site-1 day one. Lauren proposes latest stable at that time. **Owner approval required before installation.**

### Test location

```
SitoPresepi2-src/
└── e2e/
    ├── playwright.config.ts
    ├── fixtures/
    │   └── index.ts          # shared fixtures: page objects, test data
    ├── journeys/             # user journey tests (one file per journey)
    │   ├── catalogue.spec.ts
    │   ├── contact.spec.ts
    │   └── i18n.spec.ts
    └── accessibility/        # accessibility audit tests (one file per page)
        ├── home.a11y.spec.ts
        ├── catalogue.a11y.spec.ts
        └── contact.a11y.spec.ts
```

### User journeys — site-1 scope

The following journeys must have E2E coverage before any feature is considered done:

| Journey | Priority | File |
|---|---|---|
| Visitor browses catalogue (IT/EN/DE) | Must | `catalogue.spec.ts` |
| Visitor views product detail | Must | `catalogue.spec.ts` |
| Visitor submits contact form (valid) | Must | `contact.spec.ts` |
| Visitor submits contact form (invalid — validation shown) | Must | `contact.spec.ts` |
| Locale switch IT → EN → DE persists across pages | Must | `i18n.spec.ts` |
| Visitor views "The Maker" page | Should | `maker.spec.ts` |
| Visitor views "The Making" page | Should | `making.spec.ts` |

Additional journeys raised as tickets when site-1 planning is complete.

### Page Object Model

All Playwright tests use the Page Object Model pattern — no raw `page.locator()` calls in test bodies. Locator logic lives in `fixtures/`. This is enforced in G5 review.

### Playwright configuration

```typescript
// playwright.config.ts — minimum required settings
{
  testDir: './e2e',
  fullyParallel: true,
  reporter: [['html', { open: 'never' }], ['list']],
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'mobile',   use: { ...devices['iPhone 13'] } },
  ],
}
```

Mobile viewport is mandatory — EU Accessibility Act requires no device exclusion.

---

## Accessibility Standard

### Legal basis

The **EU Web Accessibility Directive** and **European Accessibility Act (EAA)** apply. Target markets are Italy, UK, and Germany. WCAG 2.1 Level AA is the minimum conformance level. This is a legal obligation — not a best practice.

**Non-negotiable:** No page ships without passing the automated accessibility gate. Manual review by Rich supplements automated checks but does not replace them.

### Standard

**WCAG 2.1 Level AA** — all pages, all locales (IT/EN/DE).

### Automated gate — axe-core

Every page in the `accessibility/` suite runs `@axe-core/playwright` and asserts zero violations at impact level `critical` or `serious`:

```typescript
import AxeBuilder from '@axe-core/playwright';

test('home page has no critical accessibility violations', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();
  const blocking = results.violations.filter(v =>
    ['critical', 'serious'].includes(v.impact)
  );
  expect(blocking).toHaveLength(0);
});
```

`moderate` and `minor` violations are logged as defect tickets (via `ticket.py`) but do not block the gate in site-1. This threshold is reviewed at the site-1 retrospective.

### Manual accessibility checklist (Rich — per sprint release)

- [ ] Keyboard navigation: all interactive elements reachable and operable via keyboard
- [ ] Focus indicators: visible on all interactive elements
- [ ] Screen reader: headings hierarchy logical; landmark regions present; images have `alt` text
- [ ] Colour contrast: minimum 4.5:1 for normal text, 3:1 for large text
- [ ] Language attribute: `<html lang="it">` / `lang="en"` / `lang="de"` correct per locale
- [ ] Form labels: every input has a visible or `aria-label` label
- [ ] Error messages: identified in accessible way (not colour alone)

**Note on German copy:** `„du"` informal register is approved sitewide (Constitution decisions log 2026-03-16). All accessibility text (error messages, ARIA labels) must follow this register consistently.

### Accessibility defect severity mapping

| axe impact | Ticket severity |
|---|---|
| critical | major |
| serious | major |
| moderate | minor |
| minor | trivial |

---

## CI/CD Quality Gates

### Current status

Manual deployment is approved for v1 (Constitution decisions log 2026-03-12). No CI/CD pipeline exists yet. This section defines the **gate requirements** — the specific checks that must pass before any PR merges or any release deploys. When CI/CD tooling is selected (Owner decision), these gates are implemented as pipeline steps.

**CI/CD tooling decision is not made.** Owner must approve before implementation. GitHub Actions is the most natural choice given the planned GitHub migration, but this is flagged as an open question — not decided here.

### Pre-merge gates (every PR)

These must all pass before any feature branch merges to `develop`:

| Gate | Check | Fail action |
|---|---|---|
| Django unit tests | `pytest --cov-fail-under=80` | Block merge |
| Next.js unit tests | `jest --ci` | Block merge |
| Linting — Python | `ruff check .` | Block merge |
| Linting — TypeScript | `eslint .` | Block merge |
| Formatting — Python | `black --check .` + `isort --check .` | Block merge |
| Formatting — TypeScript | `prettier --check .` | Block merge |
| Type check | `tsc --noEmit` | Block merge |
| No secrets | Secret scanner (e.g. `detect-secrets`) | Block merge |

### Pre-deploy gates (every release to production)

These run after the pre-merge gates and before any production deployment:

| Gate | Check | Fail action |
|---|---|---|
| E2E journeys (Must priority) | Playwright — all `journeys/` specs | Block deploy |
| Accessibility audit | Playwright + axe — all `accessibility/` specs, zero critical/serious violations | Block deploy |
| Django migration check | `python manage.py migrate --check` | Block deploy |
| Security headers check | Response headers audit (Content-Security-Policy, HSTS, X-Frame-Options) | Block deploy |

### Staging environment requirement

E2E and accessibility gates must run against a **staging environment** — not the local dev container. Staging environment spec is a Salvatore (DevOps) deliverable, to be raised as a ticket when site-1 opens.

---

## Definition of Done — Site Track

### All story types (universal)

- [ ] Ticket exists in backlog before any work starts
- [ ] TDD: failing test written and committed before implementation code (Constitution §6)
- [ ] All pre-existing tests still pass — zero regression
- [ ] Pre-commit hooks pass (Black, isort, Ruff, ESLint, Prettier)
- [ ] PR description complete (what, tests written, architecture alignment, anything unusual)
- [ ] G5 Tech Lead review approved (Alessandro)
- [ ] G6 Domain specialist review approved (Claire → backend, Ash → frontend)
- [ ] G7 PO acceptance confirmed (Sofia)

### API endpoint (Django/DRF)

- [ ] Serializer test: correct fields returned for valid object
- [ ] View test: correct status codes for valid / invalid / unauthenticated requests
- [ ] Service test: business logic exercised independently of the view
- [ ] `lib/types.ts` updated to match serializer output exactly
- [ ] GDPR: no PII returned that isn't required for the endpoint's purpose

### UI Component (Next.js)

- [ ] Renders without crash
- [ ] Correct content rendered from props
- [ ] All three locales (IT/EN/DE) render without missing string keys
- [ ] User interactions trigger correct handlers
- [ ] No hardcoded strings — all text via i18n keys

### Feature story (end-to-end)

All of the above, plus:
- [ ] Playwright E2E journey test covers the happy path
- [ ] Playwright E2E test covers at least one failure/edge case
- [ ] Accessibility audit passes for all affected pages (zero critical/serious violations)

### Bug fix

- [ ] Regression test added that reproduces the bug **before** the fix is written
- [ ] Test is red before fix, green after fix — output pasted at G2
- [ ] Root cause documented in ticket notes

---

## Open Decisions — Owner Approval Required

The following decisions are flagged and not made. Each will be raised as a ticket when site-1 opens.

| # | Decision | Options | Lauren's recommendation |
|---|---|---|---|
| OD-1 | CI/CD tooling | GitHub Actions, GitLab CI, Bitbucket Pipelines, CircleCI | GitHub Actions — natural fit once site repo moves to GitHub |
| OD-2 | Playwright version to pin | Latest stable at site-1 day one | Agree with this approach — no pre-pinning |
| OD-3 | axe-core `moderate` violation threshold for site-1 gate | Block now vs. log-only in site-1 | Log-only in site-1, reviewed at retro — site is new, some `moderate` violations expected during scaffold |
| OD-4 | Staging environment spec | Separate DigitalOcean droplet vs. same droplet, different port | Separate droplet — Salvatore to specify |
| OD-5 | Jest unit tests run location | Docker `test` service vs. host (Next.js `next dev` runs with volume mount in dev container; no Jest container spec approved yet) | To be confirmed with Salvatore before site-1 scaffold |

---

## Relationship to Tooling Track Strategy

The tooling track strategy (`strategy_testing_tooling_001.md`) is a separate, approved document. Nothing in this document supersedes or amends it. The two tracks share:

- Python version: 3.14.3
- pytest + pytest-cov
- Coverage gate: 80% minimum
- TDD mandate (Constitution §6)
- Naming convention: `test_<subject>_<condition>_<expected_result>`
- Arrange / Act / Assert structure

They differ in:
- Site track adds pytest-django, Jest, RTL, Playwright, axe-core
- Site track tests run in Docker containers (site track is containerised; tooling is host-venv)
- Site track has E2E and accessibility gates; tooling has none

---

## Document History

| Date | Change | Author |
|---|---|---|
| 2026-03-17 | Initial draft — G3 delivery | Lauren |
| 2026-03-17 | G5 corrections: axe-core import fixed to AxeBuilder/@axe-core/playwright; OD-5 added for Jest container question | Lauren |
