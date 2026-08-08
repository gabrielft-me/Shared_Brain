# Task 15 — Samsung tablet + S Pen manual test checklist (§19 step 34)

**Status:** todo
**Depends on:** 10 (must have completed the emulator walk-through first)

## Goal
Run the full plan §18 flow on the real target hardware (Samsung tablet
with S Pen) and record what works, what jitters, what breaks. Plan §19
step 34; also called out in `memory/decisions.md` D-001 as the reason the
product exists in this form.

## Deliverables
- `Shared_Brain/testing/samsung-spen-checklist.md`:
  - Device model + Android version + build fingerprint.
  - For each plan §18 step: pass / fail / observation, screen recording.
  - S Pen–specific checks: stroke latency (subjective + `frame_metrics`
    dump), palm rejection with hand resting on screen, hover cursor visible
    in ANNOTATE mode.
  - MediaProjection consent flow: does the "start now" dialog behave
    correctly on One UI? Does it survive orientation change mid-consent?
  - Overlay behavior in Samsung DeX (external monitor) — does the bubble
     land on the internal or external screen?
  - Split-screen behavior (§21) — bubble position across the split.
  - Battery drain over a 30-minute session (adb `dumpsys batterystats`).
- Bugs filed as new tasks (`Shared_Brain/tasks/NN-...`), referencing the
  checklist entry.

## Acceptance
- Every §18 step is either pass or has a filed follow-up task.
- S Pen latency measured (report the number, don't just say "feels OK").

## Log
- (fill in when working)
