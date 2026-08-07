# Task 01 — Phase 1: Overlay proof

**Status:** todo
**Depends on:** 00
**Blocks:** 02, 03

## Goal
Show a single draggable `TYPE_APPLICATION_OVERLAY` bubble that stays visible over other apps. Plan §19 Phase 1.

## Deliverables
- `permissions/OverlayPermissionManager.kt` — checks `Settings.canDrawOverlays`, launches `ACTION_MANAGE_OVERLAY_PERMISSION`.
- `overlay/TutorOverlayService.kt` — foreground service; owns `WindowManager`; hosts `OverlayManager`.
- `overlay/OverlayManager.kt` — public surface: `showBubble()`, `showTutorCard(x,y)`, `showAnnotationCanvas()`, `hideAnnotationCanvas()`, `moveWindow(view, x, y)`, `removeAll()`.
- `overlay/BubbleOverlay.kt` — `ComposeView` inside a `TYPE_APPLICATION_OVERLAY` window (`WRAP_CONTENT`, `FLAG_NOT_FOCUSABLE`), draggable via `updateViewLayout` on touch move.
- `overlay/OverlayComposeHost.kt` — small `LifecycleOwner` / `SavedStateRegistryOwner` / `ViewModelStoreOwner` shim so Compose can live inside a Service (plan §14).
- `MainActivity` — button to start `TutorOverlayService` after overlay permission granted.

## Acceptance
- Manually: install, grant overlay permission, tap Start, open Chrome — bubble is visible and draggable.
- No touches are blocked outside the bubble's rectangle (FLAG_NOT_FOCUSABLE + WRAP_CONTENT).

## Log
- (fill in when working)
