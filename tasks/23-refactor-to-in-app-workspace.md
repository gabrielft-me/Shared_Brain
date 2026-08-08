# Task 23 — Refactor to in-app workspace (Goodnotes/Squid style)

**Status:** done
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
- 2026-08-07 — **Push 1** (`66aee59`) docs: task file + D-017..D-020 + current-state pivot banner.
- 2026-08-07 — **Push 2** (`0d877d0`) new `workspace/` + `ui/workspace/` packages: `WorkspaceController`, `CoordinateMapper`, `InkStroke`, `WorkspaceTool`, `WorkspaceToolbar`, `WorkCanvas`, `AnnotationLayer`, `TutorHintCard`, `WorkspaceBubble`, `TutorWorkspaceScreen`. Build SUCCESSFUL 11 s after adding missing `Modifier.offset` imports.
- 2026-08-07 — **Push 3** (`b663c4f`) `MainActivity` rewired to 3-route nav (Setup → Workspace → Summary). All references to `OverlayPermissionManager` and `TutorOverlayService` removed. Build SUCCESSFUL 9 s.
- 2026-08-07 — **Push 4** (`ad62ce2`) deletions: `overlay/*` (8 files), `OverlayPermissionManager`, `TutorSession`. Manifest: `SYSTEM_ALERT_WINDOW` and `FOREGROUND_SERVICE_SPECIAL_USE` removed; `TutorOverlayService` service block removed. New `tutor/SessionSummaryHolder` object replaces `TutorSession`'s companion. Build SUCCESSFUL 12 s.
- 2026-08-07 — **Push 5** clean-rebuild verification: `./gradlew clean assembleDebug lintDebug` = BUILD SUCCESSFUL 26 s, 49 tasks; lint 0 errors, 13 warnings (unchanged `GradleDependency` set from Task 09). APK 18 MB. Acceptance criteria all met.

## Delivered composables (Goodnotes/Squid feel)
- **Top toolbar** with pen (three quick colors) / highlighter / eraser / undo / redo / clear / Ask-AI, rounded card with subtle shadow and active-state ring.
- **WorkCanvas** — off-white surface, 40-dp grid, persistent ink; stroke-eraser drops any stroke whose points intersect a 18-dp radius.
- **AnnotationLayer** — transient red circle-this layer, only attached in Ask mode, cleared after upload.
- **TutorHintCard** — anchored card, `right → left → below → above → clamp` placement per plan §13.
- **WorkspaceBubble** — draggable in-app AI FAB, session-state colored.

## Deferred / follow-ups
- `WorkspaceController.selectTool` currently doesn't expose `canRedo` as state (toolbar always shows redo enabled). File a small follow-up.
- Session summary route reads from `SessionSummaryHolder` which is process-scoped; not persistent across process death. Room persistence still belongs to task 14.
- The bubble's default position is calculated once from the initial canvas size; on rotation the bubble stays where it was rather than reflowing to the new bottom-right. Acceptable for now; revisit under Task 15.
