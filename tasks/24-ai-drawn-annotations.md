# Task 24 — AI-drawn annotations rendered inside the workspace canvas

**Status:** todo
**Depends on:** 23 (in-app workspace), 06 (AI pipeline client), 08 (backend skeleton)
**Related decisions:** D-005, D-007, D-018, D-021

## Goal
When the tutor responds, the backend returns not just a `hint` string but a
list of strokes that the client renders directly inside `WorkCanvas` — so
the AI *draws* into the student's page (circle around the target, arrow,
underline, later on: short handwritten steps) instead of only popping a
text card. Reuses the existing stroke-rendering pipeline; the persistent
work canvas already knows how to render `List<InkStroke>`.

## Why this is cheap
- `WorkCanvas` (`app/src/main/java/com/oakland/tutor/ui/workspace/WorkCanvas.kt`)
  already renders arbitrary `InkStroke`s via `drawPath` on a Compose Canvas.
- `AnnotationLayer` proves we can stack a second ink surface over the work
  canvas without collisions.
- `CoordinateMapper` already converts backend normalized coordinates
  (D-005) to canvas pixels; the same mapping applies to stroke points.
- `TutorResponse` (Pydantic + Kotlin data class) is a single contract to
  extend on both sides.

## Contract change (backend + client)

Add to `backend/app/schemas.py`:

```python
class AiStroke(BaseModel):
    points: list[tuple[float, float]]        # normalized 0..1, image space
    color: str = "#FF3B30"                   # AnnotationRed by default
    width_dp: float = 4.0
    kind: Literal["circle", "arrow", "underline", "handwriting"] = "circle"

class TutorResponse(...):
    ...
    ai_strokes: list[AiStroke] = []          # empty = old behavior
```

Mirror in `app/src/main/java/com/oakland/tutor/tutor/TutorResponse.kt`.

Backwards compatibility is free because the default is `[]` — old clients
and old backends stay compatible until either side starts populating it.

## Client rendering
- New composable `ui/workspace/AiStrokeLayer.kt` — pointer-input-**none**
  Canvas positioned between `WorkCanvas` and `AnnotationLayer` in
  `TutorWorkspaceScreen`. Draws from a `List<InkStroke>` owned by
  `WorkspaceController` under a new `aiStrokes` state field.
- `WorkspaceController.onTutorResponse(...)` maps normalized image
  coordinates → canvas pixels via `CoordinateMapper` and appends to
  `aiStrokes`. On next `beginAsk()` or on toolbar `Clear AI`, the list is
  wiped.
- **Not undoable** by the student's undo stack. AI marks are separate from
  the student's work per D-018 (extended in D-021).

## Backend generation (start narrow)
`backend/app/services/tutor.py` gains a `strokes(understanding, target)`
helper that emits geometric primitives from the existing `point` and
`bbox` fields:

- `kind="circle"` — sample ~64 points around an ellipse inscribing `bbox`,
  padded ~8%.
- `kind="arrow"` — a short segment terminating just outside `bbox` with a
  small arrowhead (two more segments).
- `kind="underline"` — one segment along the lower edge of `bbox`.

Real handwritten steps come later and stay in `kind="handwriting"` so the
client can style them differently (thinner width, black, animated reveal).

## Optional polish (worth doing, but separable)
- **Progressive reveal.** In `AiStrokeLayer`, animate a `progressPoints`
  counter and only draw the first N points of each stroke; increment at
  ~60 points/sec via `LaunchedEffect`. Feels like a hand writing.
- **LaTeX → strokes.** Deferred; when we get there, use a vector font
  (e.g. STIX or CMU Serif rendered to Compose `Path` via
  `PathParser`), then sample the path uniformly into `points`. Store as
  `kind="handwriting"`.

## Split into pushes
1. Docs — this task file + D-021 + `current-state.md` addendum. (this push)
2. Schema — extend `schemas.py` and Kotlin `TutorResponse`; default `[]`.
3. Backend — `tutor.strokes(...)` for the three geometric kinds; wire into
   `main.query`.
4. Client — `AiStrokeLayer` + controller wiring + toolbar "Clear AI"
   affordance; render in `TutorWorkspaceScreen`.
5. Verify — `./gradlew :app:assembleDebug lintDebug`, and a manual
   round-trip against the running FastAPI showing the circle land on the
   target.

## Acceptance
- Tapping Ask AI still returns a `TutorResponse` with the same `hint`
  card, and additionally renders a red ellipse around the target inside
  the work canvas.
- AI strokes render above student work, below annotation strokes.
- Student's undo/eraser cannot remove AI strokes; "Clear AI" wipes them.
- Old backend responses with `ai_strokes = []` behave exactly as today.
- Build + lint stay green (0 lint errors, previously-noted warnings
  unchanged).

## Non-goals
- Realistic handwriting synthesis (deferred).
- Streaming stroke updates (single response, single render — no SSE yet).
- Server-side rasterization; strokes stay vector.

## Log
- 2026-08-07 — Filed. Motivated by a design discussion about having the
  AI "respond by drawing inside the canvas" instead of only via the hint
  card.
