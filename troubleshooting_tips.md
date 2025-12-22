# Troubleshooting Tips

This file is meant to grow over time. Each entry is one “playbook card” you can reuse.

---

## Tip format (copy/paste)

### [TIP-ID] Short title
**Tags:** tag1, tag2, tag3

**Symptoms**
- What you see (UI text, error message, logs)

**Scope**
- Where it happens (local, CI, deploy, prod)
- What changed right before it happened

**Fast triage (5 minutes)**
1) Quick check #1
2) Quick check #2
3) Quick check #3

**Root causes (most common)**
- Cause A
- Cause B

**Fix**
- Step-by-step, minimal-risk steps

**Prevention**
- Guardrails (tests, checks, naming conventions, automation)

**Notes / links**
- (Optional) Relevant links, commands, or internal notes

---

## CI / CD

### [CI-001] Required check stuck on “Expected — Waiting for status to be reported”
**Tags:** github-actions, branch-protection, status-checks, cicd

**Symptoms**
- PR shows a required check as **Expected** (not running)  
  Example: `CI / gate — Expected — Waiting for status to be reported`
- You can see other checks are green, but merge is blocked.

**Scope**
- GitHub branch protection / rulesets + GitHub Actions.

**Fast triage (5 minutes)**
1) Open the PR → **Checks** tab → copy the *exact* names of the checks that actually ran  
   Example: `CI / tests (pull_request)`, `CI / gate (pull_request)`
2) Go to repo → **Settings → Rules → Rulesets / Branch protection**  
   Compare required checks vs actual check names **character-for-character**.
3) Confirm the workflow is configured to run for the same event as the PR (`pull_request`).

**Root causes (most common)**
- Branch protection requires a check name that **no longer exists** (renamed job/workflow, deleted workflow, changed event).
- You required an “aggregator” check (like `gate`) but the actual check run name is different (e.g., includes `(pull_request)`).
- You enabled Code Scanning “default setup” and it created a separate neutral check (`Code scanning results / CodeQL`) that doesn’t match your workflow.

**Fix**
1) Update required checks to match the *actual* check run names in the PR.
2) Prefer requiring the real checks directly (tests / CodeQL / dependency review) instead of a `gate` aggregator.
3) If you keep `gate`, make its name stable:
   - set `name:` at workflow + job level explicitly
   - avoid multiple workflows producing similarly named checks

**Prevention**
- Treat check names as an API: rename deliberately and update branch protection in the same PR.
- Keep workflows separated and clearly named:
  - `CI / tests`
  - `CodeQL / analyze (python)`
  - `Dependency Review / dependency-review`
- Add a short “CI contract” note in the repo (README) listing required checks.

---

## Deploy

### [DEPLOY-001] Render deploy fails with “ModuleNotFoundError: No module named 'psycopg2'”
**Tags:** render, flask, sqlalchemy, postgres, gunicorn

**Symptoms**
- Render build succeeds, but deploy crashes on startup
- Error mentions: `ModuleNotFoundError: No module named 'psycopg2'`

**Scope**
- App uses SQLAlchemy + Postgres, but the Postgres driver is missing/mismatched.

**Fast triage (5 minutes)**
1) Confirm what driver SQLAlchemy is trying to load (error stack shows `dialects/postgresql/psycopg2.py` vs `psycopg.py`)
2) Check `requirements.txt`:
   - `psycopg[binary]` for psycopg v3, OR
   - `psycopg2-binary` for psycopg2
3) Check `DATABASE_URL` scheme:
   - `postgresql://...` defaults to psycopg2 in many setups
   - `postgresql+psycopg://...` forces psycopg v3

**Fix**
- If using psycopg v3:
  1) Add `psycopg[binary]` to `requirements.txt`
  2) Normalize `DATABASE_URL` in code so it uses `postgresql+psycopg://...`

- If using psycopg2:
  1) Add `psycopg2-binary` to `requirements.txt`
  2) Keep `postgresql://...`

**Prevention**
- Add a minimal startup test in CI that imports the WSGI app (`python -c "import wsgi; print('ok')"`)
- Document DB driver expectations in README.
