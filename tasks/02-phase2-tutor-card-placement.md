# Task 02 — Phase 2: TutorCardOverlay + arbitrary positioning

**Status:** todo
**Depends on:** 01
**Blocks:** 06

## Goal
Given a normalized point/bbox in 0..1, place a floating tutor card near the target without covering it. Plan §13, §19 Phase 2.

## Deliverables
- `overlay/TutorCardOverlay.kt` — Compose card in a `TYPE_APPLICATION_OVERLAY` window (`WRAP_CONTENT`, `FLAG_NOT_FOCUSABLE`).
- `overlay/OverlayCoordinateMapper.kt`:
  - `normalizedToScreen(x, y, screenW, screenH) -> IntOffset`
  - `place(cardSize, targetBboxPx, screenSize) -> IntOffset` — try right → left → below → above → clamp to safe bounds.
- Debug harness in `MainActivity` (dev button): show card at hard-coded normalized `(0.6, 0.4)` and verify placement in portrait + landscape.

## Acceptance
- Card never overlaps the target bbox rectangle when there is room on at least one side.
- Card stays inside display bounds (never clipped off-screen).

## Log
- (fill in when working)
