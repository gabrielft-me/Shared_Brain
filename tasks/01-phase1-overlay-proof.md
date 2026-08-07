# Task 01 — Phase 1: Overlay proof

**Status:** done
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
- 2026-08-07 — `permissions/OverlayPermissionManager.kt`: `Settings.canDrawOverlays` + `ACTION_MANAGE_OVERLAY_PERMISSION` intent (package: URI).
- 2026-08-07 — `overlay/OverlayComposeHost.kt`: LifecycleOwner + ViewModelStoreOwner + SavedStateRegistryOwner shim wrapping a `ComposeView` (plan §14).
- 2026-08-07 — `overlay/BubbleOverlay.kt`: 56dp Compose circle in TYPE_APPLICATION_OVERLAY (WRAP_CONTENT, FLAG_NOT_FOCUSABLE, TRANSLUCENT), dragged via `updateViewLayout()` with touch-slop → distinguishes drag from tap.
- 2026-08-07 — `overlay/TutorOverlayService.kt`: foreground service with specialUse FGS type (Android 14+), starts OverlayManager, exposes `start/stop/devShowCard` static entry points.
- 2026-08-07 — `MainActivity.kt`: three buttons (grant overlay → grant projection → start/stop) + dev button to render a card at (0.6, 0.4).
