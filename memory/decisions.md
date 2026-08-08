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
