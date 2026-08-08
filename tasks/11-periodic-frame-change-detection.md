# Task 11 — Periodic frame-change detection (§19 step 30)

**Status:** todo
**Depends on:** 04 (capture service), 05 (composite)
**Blocks:** 12

## Goal
Turn the capture service from "one-shot on annotation complete" into
"periodic passive monitoring" as described in plan §18 *Normal monitoring*
and §19 step 30. Today the tutor only reacts when the student circles
something; nothing catches passive drift.

## Deliverables
- `capture/FrameChangeDetector.kt` — perceptual hash (aHash / dHash) of a
  downsampled Bitmap; compares to previous frame; emits `Flow<Bitmap>` only
  when the hash Hamming-distance exceeds a threshold.
- `capture/CaptureCadence.kt` — coroutine loop scheduling `captureOnce()`
  every N ms (default 1500 ms; configurable), backing off when the screen
  is idle.
- `ScreenCaptureService` starts the cadence loop when projection begins,
  cancels it on `onDestroy()`.
- `TutorSession` subscribes to change events for future "understanding
  ahead of the student" hooks; today it should only log them (no upload
  until the student circles something).

## Acceptance
- With the emulator idle, no uploads fire.
- Opening a new page fires exactly one change event within ~2 s.
- Memory footprint stays flat over 10 minutes of monitoring.

## Log
- (fill in when working)
