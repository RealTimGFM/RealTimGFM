# Troubleshooting Tips (Engineering Playbook)

A growing collection of short, reusable troubleshooting rules for DevOps, CI/CD, backend, and deployment issues.

## How to use this file
- Search (Ctrl+F) for keywords: **Expected**, **Render**, **CodeQL**, **Dependency Review**, **SQLAlchemy**, **Docker**, etc.
- Copy the **Rule** block and apply the checklist.
- After every incident, add a new entry using the template below.

---

## Table of Contents
> Add a new bullet for each rule you add.
- [TR-001 — “Expected” required check = enforcement/config mismatch](#tr-001--expected-required-check--enforcementconfig-mismatch)
- [TR-002 — Render deploy fails with missing DB driver (psycopg2)](#tr-002--render-deploy-fails-with-missing-db-driver-psycopg2)

---

## Entry Template (copy/paste for new tips)

### TR-XXX — <Short title>
**Category:** CI/CD | Deploy | Backend | DB | Security | Tooling  
**Severity:** Low | Medium | High  
**Trigger (symptoms):**
- <what you see in UI/logs>

**Most likely cause:**
- <one sentence root cause>

**Fast confirmation (2–5 min):**
1) <check 1>
2) <check 2>

**Fix (do in order):**
1) <fix 1>
2) <fix 2>

**Prevention (next time):**
- <guardrail or standard>

**Notes:**
- <links, gotchas, exact strings, etc.>

---

## TR-001 — “Expected” required check = enforcement/config mismatch

**Category:** CI/CD  
**Severity:** High (blocks merges)  

**Trigger (symptoms):**
- PR shows a required check as:
  - **Expected**
  - **Waiting for status to be reported**
  - **Missing / not found**
- Other CI jobs may be green, but merge is blocked.

**Most likely cause:**
- GitHub is enforcing a **status-check context name** that is **not being reported** for the PR head/merge commit (ruleset/branch protection/code scanning drift).

**Fast confirmation (2–5 min):**
1) Copy the exact label of the “Expected” check (case + spacing matters).
2) Compare it to the actual check-run names shown in the PR.
3) Inventory enforcement sources:
   - Settings → **Rules → Rulesets**
   - Settings → **Branches** (legacy protection rules)
   - Security → **Code scanning** (repo-level checks)

**Fix (do in order):**
1) Temporarily disable enforcement (ruleset) to unblock iteration.
2) Make required checks match the **exact** check names being reported.
3) Remove/disable any stale legacy branch protection rules if using rulesets.
4) If “Add checks” dropdown is empty, run the workflow on **main** (or target branch) so contexts exist and can be selected.
5) Re-run checks on the latest commit (or push an empty commit) to ensure the correct context is posted.

**Prevention (next time):**
- Treat **Expected** as configuration mismatch, not a slow CI run.
- Prefer one enforcement system (rulesets OR legacy branch protection), not both.
- Avoid adding an “aggregator” gate job until underlying checks are stable.

**Notes:**
- If “Require branches to be up to date” is ON, checks must run on the **merge commit**.
- Optional/neutral checks should not be required unless intentionally enforced.

---

## TR-002 — Render deploy fails with missing DB driver (psycopg2)

**Category:** Deploy | DB  
**Severity:** High (app won’t boot)  

**Trigger (symptoms):**
- Render deploy fails during `gunicorn wsgi:app` with:
  - `ModuleNotFoundError: No module named 'psycopg2'`

**Most likely cause:**
- SQLAlchemy is using the `postgresql+psycopg2` dialect because the DB URL does not specify a driver, while the project installed **psycopg v3** (`psycopg[binary]`) instead of psycopg2.

**Fast confirmation (2–5 min):**
1) Check Render logs: confirm it crashes while importing the DB driver.
2) Check `requirements.txt`: verify you have `psycopg[binary]` and not `psycopg2`.
3) Check `DATABASE_URL` scheme:
   - Render often provides `postgres://...` or `postgresql://...`
   - SQLAlchemy may default to psycopg2 unless driver is explicit.

**Fix (do in order):**
1) Normalize `DATABASE_URL` in code:
   - `postgres://...` → `postgresql://...`
   - `postgresql://...` → `postgresql+psycopg://...`
2) Redeploy.

**Prevention (next time):**
- Always force the SQLAlchemy driver explicitly for Postgres:
  - psycopg v3: `postgresql+psycopg://...`
- Add a small helper in `create_app()` that normalizes Render-provided URLs.

**Notes:**
- Alternative fix: change Render env var to use `postgresql+psycopg://...` directly (Render dashboard → Environment).

---
