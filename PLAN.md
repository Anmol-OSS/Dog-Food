# DOGFOOD 2026 — build plan (pre-kickoff notes, no project code)

Kickoff: Fri 25 Sep 18:00 UTC (23:30 IST). Freeze: Mon 28 Sep 18:00 UTC (23:30 IST).
Target claim: **T1 + T2**, plus one bonus done properly: **Normalization Proof**.
T3/T4 only if T2 is green with time left — never claimed unless finished.

## Stack (Anmol's choice, with fixes)
- Python 3.12, FastAPI (current stable, pinned), SQLAlchemy 2.0 Core (sync), Alembic, Jinja2
- SQLite in WAL mode on a **named Docker volume** (not a Windows bind mount); DB URL from env so Postgres is a config change
- CSS: Tailwind compiled at **image build time** (standalone CLI) → static file. **No CDN** (must run with network off)
- Auth: **server-side `sessions` table** (random token → user_id, expiry, revocable). Seeded fixed tokens for the checker
  (`org_7f2a`, `jdg_a_91bc`, `jdg_b_44de`, `prt_2e88`). Passwords: argon2-cffi. CSRF token on all HTML form posts, SameSite=Lax
- Boot: entrypoint `alembic upgrade head && python -m app.seed && uvicorn` (seed idempotent, not in startup_event)
- Isolation: `Depends(require_role(...))` on routes **and** every judge query scoped by `judge_id = current_user.id`
- Single image, `docker compose up` on :8080. License: MIT. Tests: pytest + FastAPI TestClient, plus run.py

## What fixtures.json actually contains (checked)
- 1 event (`submissions_close` 2026-03-01T18:00Z, i.e. closed), 8 tracks, 30 judges, 40 teams, **41** projects, 126 scores
- Rubric in the data: functionality / quality / innovation, values 2–5 only (no 1s)
- Gallery check looks for the first three titles: Glass Signal, Small Meadow, Deep Compass → page 1 must include them
- **Flat judge:** jdg_07 gave 4/4/4 to all 3 projects (std = 0). jdg_01 has 1 review (2/2/2), jdg_23 has 1 review
- **Duplicate submission:** prj_41 "Dry Harbour" = prj_07 (same team tm_07, same repo, resubmitted 3 min before close). Both were scored (5 + 4 reviews)
- **Uneven coverage:** projects have 2 (×8), 3 (×26), 4 (×3) or 5 (×4) reviews. Judges have 1–11 reviews
- Every score is by a judge assigned to that project's track (no off-track scores)
- No assignment records in the file → seed assignments from scores, so "unfinished batches" show as incomplete coverage on the dashboard

## Data model (sketch)
User (email, name) · Membership(user, event, role ∈ participant/judge/organizer) · global `is_staff` = admin
Event (name, slug, opens_at, submissions_close, judging_close, results_published_at)
Track, Prize (event, track nullable) · Team (event, name, invite_token) · TeamMember
Project (team, track, title, summary, repo_url, status draft/submitted, submitted_at, `duplicate_of` nullable)
Criterion (event, key, label, weight, min, max) — organizer-configurable weighted rubric
Assignment (judge, project, created_at, completed_at) · Score (assignment, criterion, value) + ScoreComment
AuditLog (actor, action, object, before/after JSON, at) — append-only, readable page for organizers
ApiSession tokens: seeded fixed session keys for the checker (`org_7f2a`, `jdg_a_91bc`, `jdg_b_44de`, `prt_2e88`)

## Isolation (the check that loses most teams points)
One queryset helper: judges only ever get `Score.objects.filter(assignment__judge=request.user)`.
`/api/judge/scores?judge=X` → 403 unless X is the caller (or caller is organizer). Participants → 403.
Deadline enforced in the model/service layer (`Project.can_edit(now)`), not the form; POST after close → 403/409.

## Normalization (JUDGING.md + bonus)
Weighted total per review → per-judge z-score with shrinkage:
judge mean and std are pulled toward the event-wide mean/std in proportion to how few reviews the judge has
(k-pseudo-review prior), and std has a floor so a flat judge (jdg_07) contributes ~0 signal instead of dividing by zero.
Project score = mean of normalized reviews, shown with review count and a confidence band. Raw and normalized both exported.
Duplicate prj_41 is flagged and excluded from ranking (organizer can override); decision logged in the audit trail.

## Hour plan (from kickoff)
0–10h: skeleton, models, fixture loader, auth/roles, gallery + search/filter, submission + deadline → run.py T1 green
10–30h: invites/assignment, rubric config, scoring form, isolation, CSV, organizer dashboard → run.py T2 green
30–45h: normalization + proof notebook-style doc, audit log page, tests
45–60h: docs (README, ARCHITECTURE, DATA-MODEL, JUDGING), import/export, polish
60–72h: buffer, acceptance-report.txt, demo video script
