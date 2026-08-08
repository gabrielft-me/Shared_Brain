# Current state — 2026-08-07

Snapshot of where the Oakland tutor project stands vs.
[`oakland-ai-math-tutor-cross-app-overlay-plan.md`](../oakland-ai-math-tutor-cross-app-overlay-plan.md).
This file is a living index; overwrite when the state moves.

## Repos
- **Oakland-Education** (`github.com/christian2511/Oakland-Education`)
  - `main` at commit `69f47a4` after seven pushes today:
    `528e0cf` (scaffold), `fcf16fb` (gitignore), `e6e50d2` (toolchain bump),
    `4c8bec4` (manifest), `989be66` (overlay permission fallback),
    `69f47a4` (gradle wrapper).
- **Shared_Brain** (`github.com/gabrielft-me/Shared_Brain`)
  - `main` at commit `26475fb` after four pushes today: plan + decisions,
    task backlog, task log fill-in, D-013..D-016 toolchain decisions.

## Delivered against plan
| Plan section | Where in code | Notes |
|---|---|---|
| §1 Cross-app overlay architecture | `MainActivity`, `TutorOverlayService` | Activity is setup-only. |
| §2 `TYPE_APPLICATION_OVERLAY` | all three overlays | Native WindowManager path (no FloatingX). |
| §3 SYSTEM_ALERT_WINDOW flow | `permissions/OverlayPermissionManager.kt` | Fallback intent + user-visible Toast if the settings screen is missing. |
| §4A BubbleOverlay | `overlay/BubbleOverlay.kt` | 56 dp circle, drag via `updateViewLayout`, tap→ANNOTATE. |
| §4B AnnotationCanvasOverlay | `overlay/AnnotationOverlay.kt` + `ink/InkCanvas.kt` | Attached only in ANNOTATE mode so PASSIVE/RESPONSE let touches through. |
| §4C TutorCardOverlay | `overlay/TutorCardOverlay.kt` | WRAP_CONTENT, moved by `OverlayManager.showTutorCard`. |
| §5 PASSIVE/ANNOTATE/RESPONSE | `overlay/OverlayMode.kt`, `OverlayManager.setMode` | |
| §6 Move via LayoutParams x/y | `TutorCardOverlay.move`, `BubbleOverlay` drag | |
| §7 Screen-normalized coordinates | `tutor/TutorResponse.kt` `Geometry`, `TutorSession` | Every frame carries width/height/dpi/orientation. |
| §8 MediaProjection perception | `capture/MediaProjectionController.kt`, `FrameReader.kt` | Single projection kept alive per session. |
| §9 mediaProjection FGS | `capture/ScreenCaptureService.kt` + manifest | `FOREGROUND_SERVICE_MEDIA_PROJECTION` permission declared. |
| §10 Overlay service | `overlay/TutorOverlayService.kt` | Uses `specialUse` FGS + matching permission (D-015). |
| §11 Composite (Option B) | `capture/FrameComposer.kt` | Draws strokes 1:1 onto the captured Bitmap. |
| §12 Pointing schema | `tutor/TutorResponse.kt`, `backend/app/schemas.py` | Kotlin + Pydantic mirror. |
| §13 Placement algorithm | `overlay/OverlayCoordinateMapper.kt` | Right → left → below → above → clamp. |
| §14 Compose in overlay | `overlay/OverlayComposeHost.kt` | Lifecycle + ViewModelStore + SavedStateRegistry owners. |
| §17 Package layout | `app/src/main/java/com/oakland/tutor/**` | Exact match. |
| §19 Phases 1–6 | as above | Wired end-to-end from ANNOTATE → capture → composite → POST. |

## Verified today
- **Backend contract**: `uvicorn app.main:app` on 127.0.0.1:8000. Round-trip
  from a synthetic 2560×1600 composite PNG returned the expected
  `TutorResponse` (`hint`, `point`, `bbox`, `confidence`).

## Not verified
- **Android build**: never executed. Wrapper is committed (D-016) but
  `./gradlew :app:assembleDebug` has not been run since the toolchain bump.
- **Android runtime**: never installed. `Medium_Tablet` AVD exists but has
  not been booted; no device has ever launched the app.

## Missing vs. plan
1. **Phase 7 polish** — task `Shared_Brain/tasks/07-phase7-polish.md` is still `todo`.
   - Periodic frame-change detection / "normal monitoring" loop (plan §18, §19 step 30). Today capture only fires once, on annotation completion.
   - Explicit session state machine (idle → capturing → awaiting → showing hint).
   - Three canned misconception demos.
   - Teacher summary UI (`ui/summary/SessionSummary.kt` is a placeholder).
   - Samsung tablet + S Pen manual test checklist.
2. **BubbleOverlay actions incomplete** — plan §4A lists request-hint,
   collapse/expand, and stop-session. Today the bubble only enters ANNOTATE;
   stopping requires returning to `MainActivity`.
3. **Backend intelligence is stub** — `services/{vision,pointing,tutor}.py`
   return canned values. Plan §16 requires a multimodal vision model, a
   pointing/grounding model, a tutor-reasoning layer, and session state.
   None of these are implemented.
4. **Jetpack Ink not used** — plan §16 lists "Jetpack Ink / stylus handling".
   `InkCanvas.kt` uses `awaitEachGesture` + Compose `Canvas`. Works, but
   S Pen latency on Samsung will not match `androidx.ink`.
5. **End-to-end walk-through** — plan §18 has never been demonstrated live
   (bubble over Chrome, annotate, composite, backend, card).

## Shortest path to close gaps
1. `./gradlew :app:installDebug` on the `Medium_Tablet` AVD after starting
   the backend; walk §18 by hand.
2. Implement Phase 7 items (periodic capture loop + session state first,
   then demos and summary UI).
3. Replace backend stubs with real vision + pointing + reasoning calls.
4. Consider migrating `InkCanvas` to `androidx.ink` if Samsung/S Pen latency
   becomes a blocker.
