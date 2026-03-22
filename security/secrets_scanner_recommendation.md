# Secrets Scanner Recommendation — SitoPresepi2

**Author:** Dominick (Security Engineer)
**Date:** 2026-03-19
**Ticket:** ISS-093
**Implements:** ISS-090 prerequisite
**Status:** Approved deliverable — implement via ISS-090

---

## Decision: `detect-secrets==1.5.0`

### Rationale

| Criterion | detect-secrets | gitleaks |
|---|---|---|
| Language / dependency | Pure Python — no external binary | Go binary — must be installed separately or downloaded |
| Pre-commit integration | Native: official pre-commit hook repo | Works via `local` hook or binary download; not a native pip package |
| False positive handling | `.secrets.baseline` — declarative, committed, auditable | `--baseline` flag; less standardised baseline format |
| History scanning | No (staged changes only) | Yes (full git history) |
| Proportionality | ✅ Correct scale for this project | Better suited to large repos needing history audits |
| Pinned version available via pip | ✅ `detect-secrets==1.5.0` | ❌ Not pip-installable |

**Verdict:** `detect-secrets` is the right tool for pre-commit secrets prevention in this project. It is Python-native, integrates cleanly with the existing venv and pyproject.toml, and its baseline file is a committed, auditable artefact.

**Note:** If CI/CD is later added and full git-history scanning is desired, gitleaks is the appropriate tool *for that context*. This recommendation covers pre-commit only.

---

## What Salvatore must implement (ISS-090)

### Step 1 — Add dependencies to `pyproject.toml`

Add to the `dev` extras in `pyproject.toml`:

```toml
dev = [
    "pytest==9.0.2",
    "pytest-cov==7.0.0",
    "pre-commit==4.5.1",
    "detect-secrets==1.5.0",
]
```

### Step 2 — Install into venv

```bash
source .venv/Scripts/activate
pip install pre-commit==4.5.1 detect-secrets==1.5.0
```

### Step 3 — Create `.pre-commit-config.yaml`

Create at repo root:

```yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

### Step 4 — Generate the baseline

Run from repo root (venv active):

```bash
detect-secrets scan > .secrets.baseline
```

**Review the baseline before committing.** Open `.secrets.baseline` and verify every entry is a genuine false positive. The initial scan on this repo will likely flag:

| File | Likely trigger | Is it a real secret? |
|---|---|---|
| `docker-compose.yml` | Comments containing "private key", "password", "POSTGRES_PASSWORD" | No — documentation strings; values come from `.env` on server |
| `tools/tests/*.py` | Test data containing "password", "key" in field/variable names | No — schema field names and test strings |

All confirmed false positives are acknowledged by the baseline and will not re-trigger on future commits.

### Step 5 — Install the git hook

```bash
pre-commit install
```

This writes the hook to `.git/hooks/pre-commit`. From this point forward, every `git commit` runs detect-secrets against staged files.

### Step 6 — Verify

```bash
pre-commit run --all-files
```

All files should pass (baseline is acknowledged). If any new secret is found: it is a real finding — do not add it to the baseline, fix the source instead.

### Step 7 — Commit

```bash
git add .pre-commit-config.yaml .secrets.baseline pyproject.toml
git commit -m "chore(devops): add detect-secrets pre-commit hook — ISS-090"
```

---

## Detectors

The default detector set is appropriate for this project. No custom detector configuration is needed. Default detectors cover:

- High entropy strings
- AWS credentials
- Private keys (PEM headers)
- Basic auth in URLs
- Generic API key patterns

Do not disable any default detector without a documented reason.

---

## Exclusion patterns

No permanent file exclusions are recommended. The `.secrets.baseline` is the correct mechanism for false positives — it is transparent, committed, and auditable. Permanent exclusions in `.pre-commit-config.yaml` hide the exclusion from code review; baseline entries are visible.

`memory/pm.db` and other binary files are already gitignored and will never be staged — no explicit exclude needed.

---

## Security note

The `.secrets.baseline` file **must be committed to the repository**. It is a team-visible record of acknowledged false positives. If a new committer sees it flagging an entry they don't recognise, they should investigate — not blindly extend the baseline.

Never add a real secret to the baseline. If detect-secrets fires on something that looks like a real credential: rotate it, then add the pattern to the baseline only after the credential is invalidated.
