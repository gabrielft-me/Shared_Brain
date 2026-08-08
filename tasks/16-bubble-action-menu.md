# Task 16 — Expand BubbleOverlay actions (§4A)

**Status:** todo
**Depends on:** 12 (needs session state)

## Goal
Plan §4A lists four bubble responsibilities: start-annotation, request-hint,
collapse/expand, stop-tutoring. Today only the first is wired. `memory/current-state.md`
flags this as *Missing → BubbleOverlay actions incomplete*.

## Deliverables
- Tap-vs-long-press-vs-drag disambiguation in `BubbleOverlay.DragTouch`
  (existing touch slop stays for drag detection).
- Long-press → expand a small radial or list menu overlay (a second
  `TYPE_APPLICATION_OVERLAY` window) with:
  - **Ask about this screen** — captures the current frame without an
    annotation and uploads it (uses `SessionState.Capturing` directly).
  - **Collapse** — replaces the AI dot with a smaller pin at the edge; tap
    to re-expand.
  - **Stop session** — sends the stop intent to `TutorOverlayService` and
    dismisses `ScreenCaptureService`.
- Menu window uses `WRAP_CONTENT` + `FLAG_NOT_FOCUSABLE` so it doesn't
  block the underlying app.
- Menu auto-closes on outside-tap or after 4 s of inactivity.

## Acceptance
- Short tap still enters ANNOTATE mode (regression check).
- Long-press opens the menu; each action does what its label says.
- Stop-session works without going back to `MainActivity`.

## Log
- (fill in when working)
