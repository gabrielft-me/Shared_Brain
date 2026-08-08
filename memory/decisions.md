# Decisions

Source of truth for architectural + tooling decisions on the Oakland Education tutor.
Append-only; do not rewrite history. If a decision changes, add a new entry that supersedes the old one.

---

## D-001 — Product architecture: cross-app floating overlay, not an in-app workspace
- **Date:** 2026-08-07
- **Decision:** The Android app's Activity is a setup/control surface. Tutoring happens over any foreground Android app via `TYPE_APPLICATION_OVERLAY` windows fed by `MediaProjection`.
- **Rationale:** Plan §1 — students work in Chrome / PDFs / worksheets / other math apps. Forcing them into our Activity defeats the product.

## D-002 — Native `WindowManager` + `TYPE_APPLICATION_OVERLAY` over FloatingX
- **Date:** 2026-08-07
- **Decision:** Use native Android window APIs for the overlay layer. FloatingX is *allowed* only as an optional ergonomic aid if it saves time on bubble drag/edge behavior.
- **Rationale:** Plan §15 — we need precise x/y, multiple window types, custom stylus, tight MediaProjection integration, controlled touch pass-through.

## D-003 — Three separate overlays, not one giant one
- **Date:** 2026-08-07
- **Decision:** `BubbleOverlay` (small persistent), `AnnotationCanvasOverlay` (full-screen transparent, touchable only in ANNOTATE mode), `TutorCardOverlay` (positioned near target).
- **Rationale:** Plan §4 — a permanent full-screen touchable overlay would block the underlying app.

## D-004 — Explicit interaction modes: PASSIVE / ANNOTATE / RESPONSE
- **Date:** 2026-08-07
- **Decision:** OverlayManager owns a single `OverlayMode` state. Annotation canvas is only touchable in ANNOTATE.
- **Rationale:** Plan §5 — resolves the touch-passthrough conflict.

## D-005 — Canonical coordinate system is device-screen normalized
- **Date:** 2026-08-07
- **Decision:** Every frame sent to backend carries `screen_width_px`, `screen_height_px`, `image_width_px`, `image_height_px`, `orientation`, `density_dpi`. Backend returns `{point:{x,y}, bbox:{x,y,width,height}}` in 0..1. Android does final placement math locally.
- **Rationale:** Plan §7, §13 — model identifies the target; Android owns layout.

## D-006 — MediaProjection is the perception source (not our Canvas)
- **Date:** 2026-08-07
- **Decision:** `MediaProjectionManager` → `MediaProjection` → `VirtualDisplay` → `ImageReader`. One projection kept alive per session (Android 14+ requires consent per session).
- **Rationale:** Plan §8 — the tutor must see *another* app's screen.

## D-007 — Deterministic composite over "hope the overlay is captured"
- **Date:** 2026-08-07
- **Decision:** Option B from plan §11: capture underlying display, render annotation onto a copy at identical pixel dimensions, send composite.
- **Rationale:** Preserves geometry contract with backend without device-specific behavior of MediaProjection re: overlays.

## D-008 — Language / UI stack: Kotlin + Jetpack Compose
- **Date:** 2026-08-07
- **Decision:** Kotlin, Jetpack Compose, coroutines/Flow, Retrofit for network. Overlay hosts a `ComposeView` with a small lifecycle/viewmodel/savedstate owner shim.
- **Rationale:** Plan §14, §16.

## D-009 — Min SDK 26, target SDK 34
- **Date:** 2026-08-07
- **Decision:** `minSdk = 26` (required for `TYPE_APPLICATION_OVERLAY`), `targetSdk = 34` (required for `FOREGROUND_SERVICE_MEDIA_PROJECTION` type + per-session projection consent).
- **Rationale:** Plan §2, §9.

## D-010 — Package name `com.oakland.tutor`
- **Date:** 2026-08-07
- **Decision:** Application ID `com.oakland.tutor`. Package structure follows plan §17.
- **Rationale:** Reasonable default; can be renamed before Play submission.

## D-011 — Two separate foreground services
- **Date:** 2026-08-07
- **Decision:** `ScreenCaptureService` (foregroundServiceType=`mediaProjection`) owns projection/virtual-display/image-reader. `TutorOverlayService` owns WindowManager + overlay lifecycle. Overlay service starts/stops with capture service.
- **Rationale:** Plan §9, §10 — keep capture and UI concerns split; hackathon can bind lifecycles.

## D-017 — In-app workspace, not cross-app overlays (supersedes D-001, D-011, D-015)
- **Date:** 2026-08-07
- **Decision:** The tutor lives inside the app as a Goodnotes/Squid-style
  workspace. `TYPE_APPLICATION_OVERLAY` windows are gone. `SYSTEM_ALERT_WINDOW`,
  `FOREGROUND_SERVICE_SPECIAL_USE`, and `TutorOverlayService` are all removed.
  `ScreenCaptureService` (foregroundServiceType=`mediaProjection`) remains
  because the perception layer still captures the whole physical screen
  (D-006 unchanged).
- **Rationale:** Product pivot: student's math work happens inside the app's
  own canvas. Removing the system-wide overlay path deletes a huge chunk of
  permission friction and z-order bugs, and aligns with proven pen-note UX
  (Goodnotes, Squid, Notability).
- **Consequences:** Plan §1, §4 (three system windows), §10 (overlay
  service), and §21 (overlay limitations) are historically informative but
  no longer describe the shipping architecture.

## D-018 — Two ink layers: persistent work + transient annotation
- **Date:** 2026-08-07
- **Decision:** The workspace has two overlaid ink surfaces. The lower
  `WorkCanvas` holds the student's persistent math work (undoable, erasable,
  color-selectable). The upper `AnnotationLayer` is only interactive in
  ANNOTATE mode, holds transient red "ask about this" strokes, and is
  cleared after every upload.
- **Rationale:** Prevents the student's work strokes from being confused
  with a "circle this" gesture at composite time; matches the two-layer
  Option B compositing already decided in D-007.
- **How to apply:** `FrameComposer` still composites over the MediaProjection
  frame; the annotation strokes are the transient layer.

## D-019 — Toolbar tool set: pen (3 colors) / highlighter / eraser / undo / redo / clear / AI
- **Date:** 2026-08-07
- **Decision:** Minimum viable Goodnotes-style toolbar. Pen tool has three
  quick colors (black / blue / red). Highlighter is a semi-transparent
  yellow stroke. Eraser removes whole strokes (stroke-eraser, not pixel).
  Undo/redo affect the work canvas only. Clear wipes the work canvas.
  **Ask AI** enters ANNOTATE mode.
- **Rationale:** Small, decisive tool set to ship. Deliberately no lasso,
  no shapes, no page navigation, no layers panel — those are out of scope
  for the tutor MVP.
- **How to apply:** Adding tools is fine later; do not reshape the
  workspace to accommodate them (e.g. no left-side panel).

## D-020 — WorkspaceBubble lives in-app, draggable within workspace only
- **Date:** 2026-08-07
- **Decision:** The AI dot from plan §4A survives as an in-app FAB rendered
  by `WorkspaceBubble`. Drag is bounded by the workspace layout. Tap enters
  ANNOTATE. Its background color reflects `SessionState` (idle/capturing/
  awaiting/showing) as in the pre-refactor `BubbleOverlay`.
- **Rationale:** Preserves the familiar AI-dot interaction without needing
  a system overlay. Redundant with the toolbar's Ask AI button on purpose:
  the toolbar is thumb-reach for pen-holders, the FAB is elbow-reach for
  the free hand.
- **How to apply:** Do not add a second draggable in the workspace; the
  student's ink stroke would race with drag detection.

## D-021 — AI responses render as strokes on a dedicated in-canvas layer
- **Date:** 2026-08-07
- **Decision:** `TutorResponse` gains an `ai_strokes: list[AiStroke]`
  field (normalized image coordinates, per D-005). The client renders
  them on a new non-interactive `AiStrokeLayer` sandwiched between
  `WorkCanvas` (student work, bottom) and `AnnotationLayer` (transient
  red "circle-this", top). AI strokes are owned by
  `WorkspaceController.aiStrokes`; they are **not** part of the
  student's undo/eraser stack and are cleared on the next Ask or via a
  toolbar "Clear AI" action.
- **Rationale:** Extends D-018's two-layer model to three layers so the
  AI's response is visually native to the page instead of only living in
  a floating card. Reuses the existing `InkStroke` rendering pipeline
  and the normalized coordinate contract, so the incremental cost is
  small. Keeps student ink strictly separated from AI ink at every
  stage (undo, erase, composite).
- **How to apply:** New AI marks always come from the backend — never
  synthesized client-side. First rollout ships geometric primitives only
  (`kind` ∈ {`circle`, `arrow`, `underline`}); handwriting-style strokes
  (`kind="handwriting"`) are deferred until a real vectorization path
  exists. Do not merge AI strokes into `WorkCanvas.strokes` even for
  convenience; the separation is load-bearing for undo behavior and for
  Option-B compositing (D-007) — the AI's drawing must not show up in
  the frame we send back to the model on the next turn.

## D-013 — Toolchain bump: AGP 8.7.3 / Kotlin 2.0.21 / compileSdk 36 (supersedes D-009 SDK target)
- **Date:** 2026-08-07
- **Decision:** Root plugins now `com.android.application 8.7.3` + `org.jetbrains.kotlin.android 2.0.21` + `org.jetbrains.kotlin.plugin.compose 2.0.21`. `compileSdk = 36`, `targetSdk = 36`, `minSdk = 26` unchanged.
- **Rationale:** The only Android platform installed locally is `android-36`, and the standalone Compose Compiler plugin only exists on Kotlin 2.0+. Bumping avoids a manual `sdkmanager` install of `android-34` and unblocks the build. D-009's `targetSdk = 34` decision is superseded here.

## D-014 — Cleartext HTTP allowed for emulator loopback
- **Date:** 2026-08-07
- **Decision:** `android:usesCleartextTraffic="true"` on `<application>`. `BuildConfig.TUTOR_BASE_URL` defaults to `http://10.0.2.2:8000/` for the emulator loop-back to the local FastAPI backend.
- **Rationale:** Development ergonomics; a real deployment must switch to HTTPS + drop the flag before ship.

## D-015 — TutorOverlayService uses specialUse FGS + matching permission
- **Date:** 2026-08-07
- **Decision:** Manifest declares `android.permission.FOREGROUND_SERVICE_SPECIAL_USE` alongside the existing `FOREGROUND_SERVICE_MEDIA_PROJECTION`. `TutorOverlayService` runs with `foregroundServiceType="specialUse"` and a `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` explaining why.
- **Rationale:** Android 14+ requires the matching permission per FGS type; without it the service crashes on start.

## D-016 — Gradle wrapper committed (Gradle 8.9)
- **Date:** 2026-08-07
- **Decision:** `gradlew`, `gradlew.bat`, and `gradle/wrapper/gradle-wrapper.jar` are tracked in git so a fresh clone can `./gradlew :app:assembleDebug` without installing Gradle system-wide.
- **Rationale:** Standard AOSP practice; keeps CI + local builds hermetic.

## D-012 — Never modify `Shared_Brain/memory/goal.md`
- **Date:** 2026-08-07
- **Decision:** Per `agents.md` rule 5, `goal.md` is human-owned. Agents can read it but never write. Currently it does not exist; leave it missing until the human creates it.
- **Rationale:** Explicit repo instruction.

## D-022 — Backend intelligence: Claude Opus 4.7 for all three layers; no dedicated grounding model
- **Date:** 2026-08-07
- **Decision:** Tasks 17/18/19 are implemented as provider-agnostic interfaces
  (`VisionAnalyzer`, `TargetLocator`, `TutorReasoner`) with two implementations
  each: `Noop*` (deterministic canned demos, the default) and `Anthropic*`.
  The Anthropic path uses one model for all three layers — `claude-opus-4-7`
  via the `anthropic` Python SDK, `messages.parse` structured outputs into
  Pydantic schemas, adaptive thinking, and a prompt-cached system prompt.
  Pointing (Task 18's option A) uses Claude with a strict JSON schema
  returning normalized 0..1 coordinates, clamped server-side; no dedicated
  grounding model (Molmo / Grounding DINO) is deployed.
- **Rationale:** Opus 4.7 vision returns pixel-accurate coordinates
  (high-res image support up to 2576 px long edge), which removes the main
  argument for a separate grounding model; one provider keeps ops surface and
  keys to a single vendor. Structured outputs remove hand-rolled JSON parsing.
  Env-var selection (`TUTOR_{VISION,POINTING,REASONING}_PROVIDER`, default
  `noop`; `TUTOR_MODEL` override) keeps demos runnable with no API key.
- **Consequences:** Task 18's IoU acceptance and Task 19's rubric review still
  need a live-key run. If pointing IoU disappoints on real fixtures, revisit a
  dedicated grounding model behind the same `TargetLocator` interface.

## D-023 — Session state: in-memory default, sqlite opt-in, session_id optional on the wire
- **Date:** 2026-08-07
- **Decision:** Task 20 ships `SessionStore` with `InMemorySessionStore`
  (default) and `SqliteSessionStore` (`TUTOR_SESSION_STORE=sqlite`).
  `session_id` is an *optional* multipart field on `/v1/tutor/query`;
  `POST /v1/session/start` / `POST /v1/session/{id}/end` bracket a session.
  Sessions expire after 2 h idle; transcripts carry hint, target,
  timestamp, and a 160 px thumbnail per turn.
- **Rationale:** The shipping Kotlin client (`lumina-app` `TutorApi.kt`)
  sends only image+geometry today, so session support must not break the
  existing contract. The Kotlin `TutorResponse` DTO also stays frozen —
  `HintResult.follow_up_questions`/`next_step` remain server-side (session
  transcript) until the client grows fields for them.

## D-024 — Three ink layers in LuminaBoardView; fading AI ink; canvas rasterization replaces MediaProjection for the tutor loop
- **Date:** 2026-08-07
- **Decision:** `LuminaBoardView` gets three stroke collections rendered
  bottom→top: **work** (student's permanent ink — the data source; the only
  layer undo/eraser touch), **AI** (tutor explanation ink injected from
  `TutorResponse.ai_strokes`, auto-fading after a ~8 s hold; never
  interactive, undoable, persisted, or exported), **annotation** (fixed red
  "ask about this" ink, interactive only in ANNOTATE mode, cleared after
  every upload). Perception pivots from MediaProjection screenshots to an
  `exportFrame` command that rasterizes paper + work + annotation (AI layer
  excluded) into a PNG plus a vector scene graph — canvas space becomes the
  canonical coordinate system. AI ink payloads are **semantic primitives**
  (`circle`/`arrow`/`underline`/`label` with geometry params), synthesized
  into androidx.ink `Stroke`s client-side; the model never emits raw point
  lists. Full plan: `tasks/25-three-ink-layers.md`.
- **Rationale:** All tutoring now happens on our own canvas (D-017/D-021),
  so rasterizing it directly is deterministic and deletes the
  MediaProjection permission/FGS/consent friction; vector metadata makes
  grounding exact and injection trivial. Client-side synthesis keeps stroke
  quality engineering-controlled (Noteshelf-grade taper/wobble) instead of
  sampling-controlled.
- **Consequences:** Supersedes the MediaProjection-retention clause of
  D-017 (and D-006/D-007 for the in-app flow) — `capture/` stays dead code
  (already flagged for deletion). Task 24's client sections (Compose
  `AiStrokeLayer`, `CoordinateMapper`) are superseded by task 25; its
  contract section is revised there to semantic primitives.

## D-025 — Default backend model downsized to Sonnet 4.6 (amends D-022)
- **Date:** 2026-08-07
- **Decision:** `TUTOR_MODEL` default changes `claude-opus-4-7` →
  `claude-sonnet-4-6` for all three intelligence layers, by explicit user
  direction (cost/latency). Opus stays one env var away
  (`TUTOR_MODEL=claude-opus-4-7`); `claude-haiku-4-5` is the floor option.
- **Consequences:** Amends D-022's rationale: Opus 4.7's high-res vision
  (2576 px, pixel-accurate coords) was part of why no dedicated grounding
  model was adopted. Sonnet 4.6 handles images at standard resolution, so
  2560×1600 canvas exports are downscaled by the API — normalized 0..1
  coordinates still work, but grounding precision may drop. If task 18's
  IoU acceptance fails on Sonnet, first retry on Opus before reaching for
  a dedicated grounding model.

## D-021 — App shell is React Native + Expo; ink canvas stays native Kotlin (supersedes D-008 UI stack)
- **Date:** 2026-08-07
- **Decision:** The Android app now lives in `lumina-app/` (Expo SDK 57,
  expo-router, moti/reanimated, TypeScript). `lumina-web/` is the source of
  truth for design and product flow; the RN app ports its tokens, screens and
  copy 1:1. The handwriting board is the only UI surface implemented natively:
  `android/app/src/main/java/com/oakland/tutor/board/LuminaBoardView.kt` on
  `androidx.ink` (`InProgressStrokesView`), exposed to RN as `LuminaBoardView`
  via a `SimpleViewManager`. **No fallback**: if androidx.ink fails to
  initialise, the view throws and the failure is surfaced, by explicit request.
- **Rationale:** Compose rebuilds could not hit pixel-feel parity with the web
  design system at acceptable cost. Porting React to React Native reuses the
  web components almost directly; only the latency-critical ink path needs
  Kotlin.
- **Consequences:** The old Gradle root project and `app/` (Compose) are
  deleted. `capture/*`, `network/*` and the MediaProjection permission manager
  were absorbed into `lumina-app/android/` and still compile; they are not yet
  wired into the RN flow. D-008's "Jetpack Compose" choice is superseded;
  D-006 (MediaProjection perception) and the backend contract (D-005, D-007)
  are unchanged.
