# Task 03 — Phase 3: Annotation Canvas + interaction modes

**Status:** todo
**Depends on:** 01
**Blocks:** 05

## Goal
Full-screen transparent Canvas the student can circle / underline on, plus PASSIVE / ANNOTATE / RESPONSE mode transitions. Plan §5, §19 Phase 3.

## Deliverables
- `overlay/OverlayMode.kt` — `enum class OverlayMode { PASSIVE, ANNOTATE, RESPONSE }`.
- `overlay/AnnotationOverlay.kt` — full-screen `MATCH_PARENT × MATCH_PARENT`, `PixelFormat.TRANSLUCENT`, `TYPE_APPLICATION_OVERLAY`. Touchable *only* in ANNOTATE mode; in PASSIVE/RESPONSE the view is removed (not just hidden) so touches pass through.
- `ink/InkCanvas.kt` — Compose canvas capturing pointer input, building a `Path`.
- `ink/AnnotationState.kt` — `data class` holding stroke list + screen dimensions.
- `ink/AnnotationRenderer.kt` — draws `AnnotationState` onto a `android.graphics.Canvas` (used later by Phase 5).
- `OverlayManager` gains `setMode(OverlayMode)`; transitions add/remove the annotation overlay accordingly.
- Stroke completion callback: on pointer-up, if a closed-ish stroke, notify tutor to exit ANNOTATE.

## Acceptance
- In PASSIVE, taps hit the underlying app (annotation overlay is not attached).
- In ANNOTATE, drawing works with finger or S Pen.
- Exiting ANNOTATE removes the touchable overlay before the tutor card appears.

## Log
- (fill in when working)
