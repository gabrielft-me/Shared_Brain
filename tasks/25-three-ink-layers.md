# Task 25 — Three ink layers in LuminaBoardView (work / AI / annotation)

**Status:** in-progress (phase 2 done 2026-08-07; phases 3-7 pending)
**Depends on:** lumina-app `LuminaBoardView` (D-021 canvas), 17/18 (backend grounding), 20 (session)
**Supersedes:** 24 client sections (written for the deleted Compose workspace)
**Related decisions:** D-018, D-021 (ai_strokes), D-022, D-024

## Goal

The board holds three separate ink layers:

1. **Work** — the student's real, permanent ink. Source of truth: it is what
   gets serialized, and (with the annotation layer) what gets rasterized and
   sent to the model.
2. **AI** — the tutor's explanation ink. Injected from `TutorResponse.
   ai_strokes`, rendered in the same androidx.ink pipeline, and **fades away**
   after a hold period. Never interactive, never undoable, never persisted,
   never visible to the model.
3. **Annotation** — the red "ask about this" ink the student circles with.
   Only interactive in ANNOTATE mode; cleared after every upload.

## Layer model (bottom → top)

| Layer | Collection | Interactive | Undo/eraser | Lifetime | In exported frame |
|---|---|---|---|---|---|
| Work | `workStrokes` (today's `strokes`, `LuminaBoardView.kt`) | WRITE mode | yes | persistent | yes |
| AI | `aiStrokes` | never | no | fade after hold (~8 s) or `clearAi`/next Ask | **never** |
| Annotation | `annotationStrokes` | ANNOTATE mode | no | cleared after upload | yes |

Render order in `BoardSurface.onDraw`: paper → work → AI → annotation, with
`InProgressStrokesView` staying on top for the live stroke (per the known
compositing constraint: never draw finished strokes in the FrameLayout's own
`onDraw`).

## Kotlin steps (`board/`)

1. **Mode routing.** `enum Mode { WRITE, ANNOTATE }`. Capture the mode at
   `ACTION_DOWN`; `onStrokesFinished` routes the drained stroke into
   `workStrokes` or `annotationStrokes` accordingly. In ANNOTATE the brush is
   forced to a fixed red marker (no tool/color selection) and the eraser is
   disabled.
2. **Scoped history.** Undo/redo/eraser operate on `workStrokes` only — the
   layer separation is load-bearing (D-018, D-021): student history must never
   capture AI or annotation ink.
3. **AI fade.** Per injection, one `ValueAnimator`: hold `holdMs` (default
   8000, settable from JS) → fade to 0 over ~800 ms → remove. Also cleared by
   `clearAi`, by a new `injectAiInk` (replace semantics), and on export/Ask.
   Draw the AI pass inside `canvas.saveLayerAlpha(...)` — `CanvasStrokeRenderer`
   has no per-draw alpha.
4. **`AiInkSynthesizer.kt`** — semantic payload → `Stroke` via a
   programmatically built `StrokeInputBatch` (synthetic pressure profile for
   taper), rendered with a pressure-pen brush in a dedicated AI color:
   - `circle`: ellipse inscribing the box padded ~8%, ~80 samples, slight
     wobble + small overshoot past closure so it reads hand-drawn.
   - `arrow`: shaft + two-segment head, ease-out point spacing.
   - `underline`: near-straight segment with a slight sag.
   - `label` (phase 2): text rendered through a single-stroke/handwriting
     vector font (Hershey-style or Caveat outlines) sampled into ink strokes;
     LaTeX → SVG (MathJax) → path sampling is phase 3.
   The model never emits raw point lists — quality lives client-side (D-024).
5. **Bridge commands** (`LuminaBoardViewManager`): `setMode`,
   `clearAnnotation`, `clearAi`, `injectAiInk(payloadJson)`,
   `exportFrame(requestId)`, `setAiHoldMs`. New events: `onAnnotationChange
   {count}` (enables the Send affordance), `onFrameExported {requestId,
   pngBase64, sceneGraph}`.
6. **`exportFrame`** — rasterize paper + work + annotation into a `Bitmap`
   (AI layer excluded — the model must never see its own ink, D-021) → PNG
   base64 + scene-graph JSON (per stroke: id, layer, normalized bbox). This
   replaces MediaProjection as the perception source for the tutor loop
   (D-024): `Geometry.image_*` = view size, canvas space becomes the canonical
   coordinate system, no capture permission or FGS needed.

## JS flow (`lumina-app/src/components/LessonRunner.tsx`)

Ask AI → `setMode('annotate')` → student circles (`onAnnotationChange`
enables Send) → `exportFrame` → `POST /v1/tutor/query` (image + geometry +
`session_id`) → render `hint` in the bubble + `injectAiInk(response.
ai_strokes)` → `clearAnnotation` + `setMode('write')`.

## Backend contract (revises the task 24 schema)

Semantic primitives instead of raw `points` lists:

```python
class AiStroke(BaseModel):
    kind: Literal["circle", "arrow", "underline", "label"]
    center: NormalizedPoint | None = None   # circle
    rx: float | None = None                 # circle
    ry: float | None = None                 # circle
    start: NormalizedPoint | None = None    # arrow / underline
    end: NormalizedPoint | None = None      # arrow / underline
    text: str | None = None                 # label
    height: float | None = None             # label, normalized
    color: str = "#C4685E"
    width_dp: float = 4.0

class TutorResponse(...):
    ai_strokes: list[AiStroke] = []
```

Backward compatible both ways: Moshi ignores unknown response fields until the
Kotlin DTO adds `ai_strokes` (default `emptyList()`), and `[]` keeps old
behavior. Backend generation starts geometric: derive `circle` from the
existing `bbox` (noop and anthropic paths alike); the reasoner may later
choose `arrow`/`underline`/`label`.

## Phases / pushes

1. Docs — this file + D-024 + task 24 cross-ref. *(this push)*
2. Kotlin — 3-layer refactor: mode routing, scoped history, render order,
   commands/events.
3. Kotlin — `AiInkSynthesizer` (circle/arrow/underline) + `injectAiInk` +
   fade animator.
4. Kotlin — `exportFrame` rasterization + scene graph.
5. Backend — `AiStroke` schema + generator wired into `/v1/tutor/query`.
6. JS — LessonRunner Ask-AI flow.
7. Verify — emulator round-trip in noop mode: red circle lands on the fixed
   bbox, fades after the hold, exported PNG contains work + annotation only.

## Acceptance

- Student ink is permanent; undo/eraser touch only the work layer.
- Annotation ink: red, only in ANNOTATE mode, wiped after each upload.
- AI ink renders above work / below annotation, is not erasable/undoable,
  fades after the hold or on the next Ask, and never appears in the exported
  frame.
- `exportFrame` PNG + scene graph round-trips through the backend in noop
  mode with the existing contract untouched.

## Non-goals

- Model-generated freeform handwriting (stays deferred per D-021).
- Persisting AI or annotation layers.
- Animated progressive reveal (polish, separable).

## Log
- 2026-08-07 — Planned. Layer spec from product direction: permanent user
  ink as the data source, fading AI explanation ink, red circling ink.
- 2026-08-07 — **Phase 2 done.** `LuminaBoardView`: `workStrokes` /
  `annotationStrokes` / `aiStrokes` collections; `Mode { WRITE, ANNOTATE }`
  with per-stroke routing via a `strokeLayer` map keyed on
  `InProgressStrokeId` (recorded at ACTION_DOWN, resolved in
  `onStrokesFinished`); ANNOTATE forces a fixed red marker (`ANNOTATION_RED`
  #FF3B30, tool + eraser ignored); undo/redo/eraser/clear scoped to
  `workStrokes` only; render order paper → work → AI (inside
  `saveLayerAlpha(aiAlpha)`) → annotation; `clearAnnotation()`/`clearAi()`.
  Manager: commands `setMode`/`clearAnnotation`/`clearAi`, event
  `onAnnotationChange {count}`. `:app:compileDebugKotlin` green.
  Note: `lumina-app/` is still untracked in the main repo (refactor 23
  pushes 5/6-6/6 pending), so the Kotlin change rides in the working tree
  until that lands.
