# Task 04 — Phase 4: MediaProjection foreground service

**Status:** done
**Depends on:** 00
**Blocks:** 05

## Goal
Capture the device display into a `Bitmap` via a properly-typed foreground service. Plan §8, §9, §19 Phase 4.

## Deliverables
- `permissions/MediaProjectionPermissionManager.kt` — wraps `MediaProjectionManager.createScreenCaptureIntent()` + `ActivityResultLauncher`.
- `capture/ScreenCaptureService.kt` — foreground service with `foregroundServiceType="mediaProjection"`, sticky notification, receives `(resultCode, data)` via start Intent.
- `capture/MediaProjectionController.kt` — creates/stops `MediaProjection`; keeps a single instance alive per session.
- `capture/VirtualDisplayController.kt` — creates `VirtualDisplay` using screen metrics + `ImageReader.surface`.
- `capture/FrameReader.kt` — `ImageReader.OnImageAvailableListener`, converts latest `Image` → `Bitmap` (handling row-padding). Exposes `suspend fun captureOnce(): Bitmap`.

## Acceptance
- Consent dialog appears once per session.
- Service runs in foreground with visible notification (Android requirement).
- `captureOnce()` returns a Bitmap whose dimensions match the display metrics.

## Log
- 2026-08-07 — `permissions/MediaProjectionPermissionManager.kt`: wraps `createScreenCaptureIntent()` behind an `ActivityResultLauncher`.
- 2026-08-07 — `capture/ScreenCaptureService.kt`: foreground service with `foregroundServiceType=mediaProjection`; receives `(resultCode, data)` via start Intent; exposes a `current()` singleton so `TutorSession` can request `captureOnce()`.
- 2026-08-07 — `capture/MediaProjectionController.kt`: builds `MediaProjection` → `ImageReader` (RGBA_8888) → `VirtualDisplay` on a dedicated HandlerThread. Registers a Callback that tears down on system-triggered stop.
- 2026-08-07 — `capture/FrameReader.kt`: coroutine-friendly `awaitLatest()`; converts `Image` planes to `Bitmap` with correct row-padding handling.
- 2026-08-07 — MainActivity wires the consent launcher and starts `ScreenCaptureService.start(this, resultCode, data)`.
