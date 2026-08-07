# Task 07 — Phase 7: Polish

**Status:** todo
**Depends on:** 06

## Goal
Make the tutor demo-ready. Plan §19 Phase 7.

## Deliverables
- Periodic frame-change detection (hash of downsampled frame) to avoid re-querying identical screens.
- Session state machine: idle → capturing → awaiting-response → showing-hint.
- Three canned misconception demos (scripted screenshots + expected responses) documented under `Shared_Brain/demos/`.
- `ui/summary/SessionSummary.kt` — teacher-facing summary at session end (count of hints, topics asked, timestamps).
- Samsung tablet + S Pen manual test checklist under `Shared_Brain/testing/samsung-tablet-checklist.md`.

## Acceptance
- Demo runs end-to-end for all three canned scenarios without a rebuild.
- Summary screen shows a non-empty session record.

## Log
- (fill in when working)
