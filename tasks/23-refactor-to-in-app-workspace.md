# Task 23 — Refactor to in-app workspace (Goodnotes/Squid style)

**Status:** in-progress
**Depends on:** —
**Supersedes:** D-001, D-003 (partial), D-011, D-015 (see `../memory/decisions.md`)
**Blocks:** most future UI work — 11, 12, 13, 14, 16 all need reinterpretation
against the in-app model.

## Goal
Reverse the cross-app overlay architecture. The tutor now lives **inside**
the app as a Goodnotes/Squid-style note-taking workspace with an integrated
AI assistant. The floating agent no longer hovers over other apps.

`MediaProjection` is retained so the model still sees the whole physical
display — the app is simply the primary content on that display.

## Delivered UI (Goodnotes/Squid pattern)
- **Full-bleed writable canvas** (`WorkCanvas`) — the student writes math
  in persistent ink; this is the workspace.
- **Top toolbar** (`WorkspaceToolbar`) — pen with three quick colors,
  highlighter, eraser, undo, redo, clear, and an **Ask AI** action. Active
  tool is visually distinct.
- **AI FAB** (`WorkspaceBubble`) — a small floating dot in the corner,
  draggable within the workspace only, tap-to-ask. State-color follows the
  session state (idle/capturing/awaiting/showing).
- **Annotation layer** (`AnnotationLayer`) — transient red ink drawn only
  in ANNOTATE mode, on top of the persistent work canvas.
- **Hint card** (`TutorHintCard`) — anchored floating card near the target
  the backend returned; can be dismissed by tap.

## Removed
- `overlay/TutorOverlayService.kt`
- `overlay/OverlayComposeHost.kt`
- `overlay/BubbleOverlay.kt`, `AnnotationOverlay.kt`, `TutorCardOverlay.kt` (system windows)
- `overlay/OverlayManager.kt`
- `permissions/OverlayPermissionManager.kt`
- Manifest: `<uses-permission SYSTEM_ALERT_WINDOW>`, `FOREGROUND_SERVICE_SPECIAL_USE`,
  and the `<service ... TutorOverlayService>` block.

## Preserved
- `capture/*` — `ScreenCaptureService`, `MediaProjectionController`,
  `FrameReader`, `FrameComposer`, `FrameChangeDetector`.
- `ink/*` — `AnnotationState`, `InkCanvas` (repurposed as building block for
  both work and annotation surfaces), `AnnotationRenderer`.
- `network/*`, `tutor/*` — contract with the backend unchanged.
- `overlay/OverlayCoordinateMapper.kt` — moves to `workspace/CoordinateMapper.kt`;
  still needed to place the hint card near the target.

## Split into pushes (no author trailer)
1. Docs — this file, decisions D-017..D-020, current-state update.
2. Manifest + `permissions/OverlayPermissionManager.kt` removal.
3. Delete `overlay/TutorOverlayService.kt`, rewrite `TutorSession` to drop
   the service-owned coroutine scope.
4. New `workspace/` package: controller + composables.
5. `MainActivity` rewrite (setup → workspace navigation) + delete old
   `overlay/*` window classes.
6. `./gradlew clean assembleDebug lintDebug` proof + commit APK metrics
   into the log.

## Acceptance
- No manifest permission or service references overlay windows.
- Fresh install shows workspace after granting projection consent only.
- The three demos still work (backend contract unchanged).
- `assembleDebug` succeeds with 0 lint errors.

## Log
- 2026-08-07 — Kicked off. See push checklist above.
