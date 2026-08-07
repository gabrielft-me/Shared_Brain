# Task 03 — Phase 3: Annotation Canvas + interaction modes

**Status:** done
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
- 2026-08-07 — `overlay/OverlayMode.kt` (PASSIVE / ANNOTATE / RESPONSE).
- 2026-08-07 — `overlay/AnnotationOverlay.kt`: MATCH_PARENT×MATCH_PARENT TYPE_APPLICATION_OVERLAY, TRANSLUCENT. Attached only in ANNOTATE mode; detached on exit so touches pass through in PASSIVE/RESPONSE.
- 2026-08-07 — `ink/InkCanvas.kt`: Compose canvas capturing pointer input via `awaitEachGesture`, driving an `AnnotationState`. `onStrokeComplete` fires on pointer-up so OverlayManager can transition to RESPONSE.
- 2026-08-07 — `ink/AnnotationState.kt`: mutable stroke list in screen-pixel coords + `snapshot()` for immutable handoff.
- 2026-08-07 — `ink/AnnotationRenderer.kt`: renders strokes onto a raw `android.graphics.Canvas` with configurable scale (used by Phase 5 composer).
- 2026-08-07 — `OverlayManager.setMode()` performs the attach/detach transitions.
