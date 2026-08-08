# Tasks

Full project task list, derived from `oakland-ai-math-tutor-cross-app-overlay-plan.md`
and gaps captured in [`../memory/current-state.md`](../memory/current-state.md).
Read [`../memory/decisions.md`](../memory/decisions.md) before starting any task —
several tasks depend on decision entries (D-013..D-016 in particular).

Progress statuses: `todo`, `in-progress`, `done`, `superseded`.

## Setup + phased build (plan §19)

| # | File | Group | Status |
|---|------|-------|--------|
| 00 | [00-scaffold-android-project.md](00-scaffold-android-project.md) | Setup | done |
| 01 | [01-phase1-overlay-proof.md](01-phase1-overlay-proof.md) | Phase 1 — Overlay proof | done |
| 02 | [02-phase2-tutor-card-placement.md](02-phase2-tutor-card-placement.md) | Phase 2 — Arbitrary positioning | done |
| 03 | [03-phase3-annotation-canvas.md](03-phase3-annotation-canvas.md) | Phase 3 — Annotation Canvas | done |
| 04 | [04-phase4-media-projection.md](04-phase4-media-projection.md) | Phase 4 — Screen capture | done |
| 05 | [05-phase5-composite-geometry.md](05-phase5-composite-geometry.md) | Phase 5 — Composite geometry | done |
| 06 | [06-phase6-ai-pipeline.md](06-phase6-ai-pipeline.md) | Phase 6 — AI pipeline (client) | done |
| 07 | [07-phase7-polish.md](07-phase7-polish.md) | Phase 7 — Polish (umbrella) | superseded → 11..15 |
| 08 | [08-backend-fastapi-skeleton.md](08-backend-fastapi-skeleton.md) | Backend skeleton | done (stub) |

## Verification (from `current-state.md` → *Not verified*)

| # | File | Status |
|---|------|--------|
| 09 | [09-verify-android-build.md](09-verify-android-build.md) | done |
| 10 | [10-emulator-install-and-walkthrough.md](10-emulator-install-and-walkthrough.md) | todo |

## Phase 7 — polish, split from task 07 (plan §19 steps 30–34)

| # | File | Status |
|---|------|--------|
| 11 | [11-periodic-frame-change-detection.md](11-periodic-frame-change-detection.md) | todo |
| 12 | [12-client-session-state-machine.md](12-client-session-state-machine.md) | todo |
| 13 | [13-canned-misconception-demos.md](13-canned-misconception-demos.md) | todo |
| 14 | [14-teacher-summary-ui.md](14-teacher-summary-ui.md) | todo |
| 15 | [15-samsung-spen-checklist.md](15-samsung-spen-checklist.md) | todo |

## Product completeness (from `current-state.md` → *Missing vs. plan*)

| # | File | Plan link | Status |
|---|------|-----------|--------|
| 16 | [16-bubble-action-menu.md](16-bubble-action-menu.md) | §4A | todo |
| 17 | [17-real-vision-model.md](17-real-vision-model.md) | §16 backend | todo |
| 18 | [18-real-pointing-model.md](18-real-pointing-model.md) | §12, §16 | todo |
| 19 | [19-tutor-reasoning-layer.md](19-tutor-reasoning-layer.md) | §16 | todo |
| 20 | [20-backend-session-state.md](20-backend-session-state.md) | §16 (Session state) | todo |
| 21 | [21-androidx-ink-migration.md](21-androidx-ink-migration.md) | §16 (Jetpack Ink) | todo |
| 22 | [22-backend-https-deployment.md](22-backend-https-deployment.md) | D-014 | todo |

## Architecture pivot

| # | File | Status |
|---|------|--------|
| 23 | [23-refactor-to-in-app-workspace.md](23-refactor-to-in-app-workspace.md) | done |

## Working agreement

- Read [`../agents.md`](../agents.md) and [`../memory/decisions.md`](../memory/decisions.md)
  before starting a task. Some tasks explicitly cite decision entries.
- Update the task file's `Status` and add a `## Log` entry when work starts and ends.
- Commit the completed task file(s) with the code change in the main repo.
- `git pull` in `Shared_Brain/` before `git push`.
- Do not modify [`../memory/goal.md`](../memory/goal.md) — human-owned per agents.md rule 5.

## Dependency map (informal)

```
00 ── 01 ── 02 ──┐
       │         │
       ├── 03 ───┤
       │         │
00 ── 04 ── 05 ──┴── 06 ── 09 ── 10 ── 15 ── 21
                              │      │
                              ├── 11 ├── 13
                              │      │
                              └── 12 ┴── 14
                              │
                              └── 16

08 ── 17 ──┐
08 ── 18 ──┼── 19 ── 22
08 ── 20 ──┘
```
