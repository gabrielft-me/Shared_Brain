# Task 14 — Teacher-facing session summary UI (§19 step 33)

**Status:** todo
**Depends on:** 12 (needs the session history feed)

## Goal
Replace the placeholder in `ui/summary/SessionSummary.kt` with a real
Compose screen a teacher can look at after a tutoring session. Plan §19
step 33.

## Deliverables
- `ui/summary/SessionSummary.kt` — Compose screen bound to a
  `SummaryViewModel` fed by the session history from Task 12.
- Shows, per session:
  - Session start / stop timestamps + duration.
  - Number of hints requested.
  - For each hint: timestamp, target description, hint text, and a small
     thumbnail of the composited frame that was uploaded.
  - Any errors encountered (from `SessionState.Error`).
- `MainActivity` gains a **View last session** button that opens the summary.
- Session records persisted to a Room database (`SessionDao`,
  `SessionEntity`, `HintEntity`) so summaries survive process death.

## Acceptance
- After running any of the demos in Task 13, opening the summary shows
  the correct record.
- Killing and relaunching the app still shows the previous session.

## Log
- (fill in when working)
