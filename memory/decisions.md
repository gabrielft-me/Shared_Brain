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

## D-012 — Never modify `Shared_Brain/memory/goal.md`
- **Date:** 2026-08-07
- **Decision:** Per `agents.md` rule 5, `goal.md` is human-owned. Agents can read it but never write. Currently it does not exist; leave it missing until the human creates it.
- **Rationale:** Explicit repo instruction.
