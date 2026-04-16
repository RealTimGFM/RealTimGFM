# Troubleshooting Tips

This file is meant to grow over time. Each entry is one “playbook card” you can reuse.

---

## Debugging rule of thumb

If a UI bug changes based on cursor speed, hover location, or tiny pointer movements, inspect event resolution before changing styling or animation.

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

- PR shows a required check as **Expected** (not running)Example: `CI / gate — Expected — Waiting for status to be reported`
- You can see other checks are green, but merge is blocked.

**Scope**

- GitHub branch protection / rulesets + GitHub Actions.

**Fast triage (5 minutes)**

1) Open the PR → **Checks** tab → copy the *exact* names of the checks that actually ranExample: `CI / tests (pull_request)`, `CI / gate (pull_request)`
2) Go to repo → **Settings → Rules → Rulesets / Branch protection**Compare required checks vs actual check names **character-for-character**.
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

### [UI-001] Drag-and-drop bug looks visual, but root cause is hit-testing / event targeting

**Tags:** frontend, drag-drop, ui, events, debugging

**Symptoms**

- Drag UI looks wrong only during certain movement patterns
- Works when moving quickly, fails when moving slowly
- Works in some cursor positions, fails in others
- Ghost/preview/placeholder seems duplicated, misplaced, or “stuck”

**Scope**

- Usually happens in interactive lists, sortable queues, kanban boards, nested buttons/cards, or drop zones with internal elements.

**Fast triage (5 minutes)**

1) Log the drag pipeline on every `dragover`:
   - pointer position
   - `event.target`
   - resolved drop target id
   - resolved insert position
2) Check whether drop logic depends on the DOM node under the cursor instead of list geometry.
3) Reproduce slowly and intentionally hover:
   - left side of a row
   - center of a row
   - right side of a row
   - gaps between rows
   - empty space inside the container

**Root causes (most common)**

- Drop target is derived from `event.target` or `closest(...)`, which changes depending on nested elements.
- Empty space inside the container resolves to “no target”, so state falls back incorrectly.
- Visual drag-preview logic is being debugged, but the real bug is incorrect target resolution.
- Reorder logic uses horizontal hit areas accidentally when only vertical ordering matters.

**Fix**

1) Separate input resolution from rendering.
2) Resolve drop position from container geometry first:
   - pointer Y
   - item rects
   - midpoint before/after logic
3) Use `event.target` only as a convenience, not as the source of truth.
4) Keep a single canonical computed result per `dragover`:
   - `targetTaskId`
   - `dropPosition`
5) Only after that, update preview/ghost/classes from that computed result.

**Prevention**

- Treat drag/drop debugging in this order:
  1) event pipeline
  2) target resolution
  3) state transitions
  4) visuals
- Add a temporary debug mode for complex interactions showing:
  - dragged id
  - hovered target id
  - computed insert position
- For sortable vertical lists, prefer pointer-to-list-geometry logic over DOM-hit logic.
- When a bug behaves differently “slow vs fast,” suspect event targeting or hover resolution early.

**Notes / links**

- Heuristic: if the bug changes with cursor speed or left/right placement, it is often not a CSS problem.
- Ask first: “What exact target does the code think I am over right now?”
