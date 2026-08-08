# Task 21 — Migrate InkCanvas to androidx.ink for S Pen latency

**Status:** todo
**Depends on:** 15 (S Pen checklist informs whether this is worth doing)

## Goal
Plan §16 explicitly lists "Jetpack Ink / stylus handling" but the current
`ink/InkCanvas.kt` uses `awaitEachGesture` + Compose `Canvas`. Flagged in
`memory/current-state.md` as *Missing → Jetpack Ink not used*. Only pursue
this once Task 15 confirms Samsung/S Pen latency is a real problem.

## Deliverables
- Add `androidx.ink:ink-*` dependencies (authoring, rendering, brush).
- Replace `InkCanvas` internals with `InProgressStrokesView` +
  `InProgressStrokesManager` on a `SurfaceView`, keeping the same
  `AnnotationState` output contract (screen-pixel points) so
  `FrameComposer` doesn't change.
- Feature-flag both implementations behind `TUTOR_INK_BACKEND` (compose/androidx-ink)
  so we can A/B compare during Task 15 re-runs.
- Measure end-to-end pen-down → stroke-visible latency on Samsung in both
  modes; record numbers in `Shared_Brain/testing/ink-latency.md`.

## Acceptance
- Both backends compile and pass the annotation → composite → upload flow.
- androidx.ink latency < compose baseline by ≥ 15 ms on the target device.

## Log
- (fill in when working)
