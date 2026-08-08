# Task 10 — Install to Medium_Tablet AVD and walk plan §18 end-to-end

**Status:** todo
**Depends on:** 09
**Blocks:** 13 (demos build on the same walk-through), 15 (S Pen checklist)

## Goal
Demonstrate the full runtime sequence from plan §18 on a live device: bubble
over another app → annotate → composite → backend → tutor card. Flagged in
`memory/current-state.md` as *Not verified → Android runtime* and *Missing → §18 walk-through*.

## Deliverables
- Backend running locally (`backend/.venv/bin/uvicorn app.main:app --port 8000`).
- `Medium_Tablet` AVD booted (`emulator -avd Medium_Tablet`).
- `./gradlew :app:installDebug` succeeds.
- Manual walk-through:
  1. Launch app; tap **Grant overlay permission** → grant in Settings.
  2. Return; tap **Grant screen capture** → accept the system consent dialog.
  3. Tap **Start tutoring session** → bubble appears; notification shows for
     both `ScreenCaptureService` and `TutorOverlayService`.
  4. Open Chrome (or the emulator's browser) to a page with a math expression.
  5. Tap the bubble → ANNOTATE mode; circle the expression.
  6. On pointer-up: composite is uploaded; `TutorCardOverlay` appears near
     the target with the stubbed hint.
  7. Tap **Stop tutoring session** from `MainActivity` → all overlays gone,
     both notifications cleared.
- Screenshots of each step attached under `Shared_Brain/testing/walkthrough-10/`.

## Acceptance
- All seven steps succeed without a crash.
- The tutor card is placed off the target rectangle (right/left/below/above,
  never covering the circle).
- Nothing about the underlying Chrome page becomes uninteractive outside of
  ANNOTATE mode.

## Log
- (fill in when working)
