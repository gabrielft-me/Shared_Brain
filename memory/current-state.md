# Current state — 2026-08-07

Snapshot of where the Oakland tutor project stands.

> **Architecture pivot.** As of 2026-08-07 (decision D-017), the tutor no
> longer runs as system-wide `TYPE_APPLICATION_OVERLAY` windows over other
> apps. It lives inside the app as a Goodnotes/Squid-style workspace.
> `MediaProjection` is retained — the model still sees the whole physical
> display. The reference plan
> [`oakland-ai-math-tutor-cross-app-overlay-plan.md`](../oakland-ai-math-tutor-cross-app-overlay-plan.md)
> §1, §4, §10, and §21 are historical; the in-app model is the shipping one.
> Refactor tracking: [`../tasks/23-refactor-to-in-app-workspace.md`](../tasks/23-refactor-to-in-app-workspace.md).

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

## Missing vs. plan — mapped to tasks

Every gap now has an owning task under [`../tasks/`](../tasks/). See
[`../tasks/README.md`](../tasks/README.md) for the full index +
dependency map.

| Gap | Owning task |
|---|---|
| Android build never executed under new toolchain | 09 |
| App never installed / §18 never walked live | 10 |
| Phase 7 — periodic frame-change detection (§19 step 30) | 11 |
| Phase 7 — client session state machine (§19 step 31) | 12 |
| Phase 7 — three canned misconception demos (§19 step 32) | 13 |
| Phase 7 — teacher summary UI (§19 step 33) | 14 |
| Phase 7 — Samsung tablet + S Pen checklist (§19 step 34) | 15 |
| Bubble actions incomplete (§4A) | 16 |
| Backend vision model is stub (§16) | 17 |
| Backend pointing/grounding is stub (§12, §16) | 18 |
| Backend tutor reasoning is stub (§16) | 19 |
| Backend session state missing (§16) | 20 |
| Jetpack Ink not used (§16) — conditional on 15 | 21 |
| HTTPS deployment + drop cleartext (D-014) | 22 |
| AI response confined to hint card, not drawn in the workspace (D-021) | 24 |

Task 07 has been retired as an umbrella pointing at 11..15.

## Shortest path to close gaps

Order that keeps each step demoable:

1. **09 → 10** — verify build; install on `Medium_Tablet`; walk plan §18 once.
2. **11 → 12** — periodic capture + explicit session state (Phase 7 foundation).
3. **13 → 14** — three demos + teacher summary using the new session state.
4. **16** — bubble action menu (needs session state).
5. **17 → 18 → 19** — replace backend stubs with real vision, pointing,
   reasoning. Task 20 (backend session state) can run in parallel; task 19
   depends on it.
6. **15** — re-run the checklist on real Samsung hardware.
7. **21** — androidx.ink migration, only if 15 shows latency is a real blocker.
8. **22** — HTTPS deployment before any user ships this.
