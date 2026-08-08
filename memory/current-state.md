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
- **Task 25 phases 2-6 complete, phase 7 scriptable part verified**
  (2026-08-07, latest): all three ink layers implemented end-to-end —
  Kotlin layer refactor + AI ink synthesizer + fade + `exportFrame`/scene
  graph, backend `ai_strokes` (Oakland `e441a2b`), JS circle-and-ask flow
  in `LessonRunner` (sparkles → annotate → export → upload → hint +
  `injectAiInk`). Verified: backend round-trip with `ai_strokes`,
  `assembleDebug` + `tsc` green, app boots on emulator with the new flow
  and the ask-mode UI works. Pending: on-device ink-visual golden path —
  blocked by a concurrent session hot-reloading the same emulator and its
  in-flight finished-stroke rendering fix (see task 25 log).
- **Task 25 planned + phase 2 landed** (2026-08-07, later): three ink layers
  in `LuminaBoardView` — permanent work ink (only layer undo/eraser touch),
  fading AI ink (`ai_strokes`, injected, phase 3), red annotation ink
  (ANNOTATE mode, cleared per upload). D-024 also pivots perception from
  MediaProjection to canvas rasterization (`exportFrame`, phase 4). Kotlin
  layer refactor compiles green; task 24 superseded → 25. Caveat:
  `lumina-app/` is still untracked in Oakland-Education (refactor 23 pushes
  5/6–6/6 pending), so the phase-2 Kotlin lives only in the working tree.
- **Backend tasks 17–20 landed** (2026-08-07, later): provider interfaces
  `VisionAnalyzer` / `TargetLocator` / `TutorReasoner` with `noop` (default)
  and `anthropic` (`claude-opus-4-7`, structured outputs) implementations
  (D-022), plus session state — `/v1/session/start`, `/v1/session/{id}/end`,
  optional `session_id` on `/v1/tutor/query`, in-memory + sqlite stores
  (D-023). Verified in noop mode: response shape matches
  `lumina-app/.../TutorResponse.kt` exactly; two-query session produces a
  2-turn transcript with thumbnails; sqlite survives restart. The
  `anthropic` path is code-complete but not yet exercised — no
  `ANTHROPIC_API_KEY` on this machine.

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
| ~~Backend vision model is stub (§16)~~ done; live-key acceptance pending | 17 |
| ~~Backend pointing/grounding is stub (§12, §16)~~ done; IoU acceptance pending | 18 |
| ~~Backend tutor reasoning is stub (§16)~~ done; rubric review pending | 19 |
| ~~Backend session state missing (§16)~~ done; client doesn't send session_id yet | 20 |
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
5. **17 → 18 → 19 + 20** — done in code (D-022, D-023). Remaining: run the
   acceptance passes with a real `ANTHROPIC_API_KEY` (`TUTOR_*_PROVIDER=
   anthropic`), and wire `session_id` + `/v1/session/*` into the client.
6. **15** — re-run the checklist on real Samsung hardware.
7. **21** — androidx.ink migration, only if 15 shows latency is a real blocker.
8. **22** — HTTPS deployment before any user ships this.

## 2026-08-07 — RN shop screen: real collectible artwork

The RN `lumina-app` collection screen was rendering placeholder colored discs
where the web shows the full illustrated collectible set. Ported
`lumina-web/src/components/illustrations/Collectible.tsx` 1:1 to
`lumina-app/src/primitives/Collectible.tsx` on `react-native-svg` (all 14
shapes, same paths/gradients/faces/cast shadow; web palette key `lav` maps to
native `lavender`). Locked/muted state is desaturation-in-code + 0.55 opacity
since RN has no CSS `saturate()` filter. `app/(tabs)/shop.tsx` now passes
`shape` through (tile + detail sheet) and its local disc placeholder is
deleted. Per D-021 this keeps the RN port pixel-faithful to the web.

## 2026-08-08 — LuminaBoardView: ink was authored but never rendered (fixed)

Symptom reported as "canvas not reacting to touch": in fact touches always
registered (tutor evaluated strokes) but no ink ever appeared. Root cause:
ink-authoring's `InProgressStrokesView` front-buffer pipeline fails silently
on emulators — live strokes never render and `onStrokesFinished` never fires,
so strokes were authored yet never drawn or committed (zero logcat errors;
confirmed via instrumentation: `BoardSurface.onDraw` ran with `work=0`
forever while `startStroke` succeeded).

Rewrite in `lumina-app/.../board/LuminaBoardView.kt`:
- Authoring now uses `InProgressStroke` fed directly from `onTouchEvent`
  (`MutableStrokeInputBatch` incl. historical points + pressure), committed
  synchronously via `toImmutable()` on ACTION_UP. `InProgressStrokesView`,
  `onStrokesFinished` and the stroke-id/layer map are gone.
- Rendering: `CanvasStrokeRenderer.create(forcePathRendering = true)` — the
  default `drawMesh` path is unreliable on emulator GPUs. Live stroke draws
  in `BoardSurface.onDraw` on top of committed layers.
- Paper guides live in a separate `PaperSurface` child under `BoardSurface`.
- alpha07 native validation rejects whole input batches (duplicate
  position+time points; rare spurious `stroke_unit_length` mismatch) —
  points are deduped and `enqueueInputs` is wrapped in try/catch.

Verified on Medium_Tablet emulator: pen strokes visible on ruled/grid/dot/
blank papers, persist across paper switches, undo works, D-024 export path
untouched (`renderTo` unchanged). Trade-off: no front-buffer low-latency
path anywhere now — task 15/21 should re-measure S Pen latency on real
Samsung hardware before deciding whether to bring a front-buffer variant
back for stylus only.
