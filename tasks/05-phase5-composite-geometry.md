# Task 05 — Phase 5: Composite geometry

**Status:** todo
**Depends on:** 03, 04
**Blocks:** 06

## Goal
Render the annotation layer onto a copy of the captured display frame at identical pixel dimensions, producing the image sent to the backend. Plan §11 Option B.

## Deliverables
- `capture/FrameComposer.kt`:
  - `fun compose(screen: Bitmap, annotation: AnnotationState): Bitmap`
  - Copies `screen` (ARGB_8888), draws strokes via `AnnotationRenderer` scaled to screen dimensions, returns new Bitmap.
- Unit-testable pure function (no Android framework dependencies beyond `Bitmap`/`Canvas`/`Paint`).

## Acceptance
- Composite pixel dimensions == captured screen dimensions.
- Stroke positions on the composite match on-screen positions (round-trip within ±1 px).

## Log
- (fill in when working)
