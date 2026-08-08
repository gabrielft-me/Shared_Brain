# Task 25 — Three ink layers in LuminaBoardView (work / AI / annotation)

**Status:** done (all phases incl. the on-device visual golden path,
2026-08-08 — see log for the `MutableStrokeInputBatch.add` pressure-slot
bug that was blocking AI ink from ever rendering)
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
- 2026-08-07 — **Phase 3 done.** All three ink types now render:
  `AiInkSynthesizer.kt` turns the semantic payload into androidx.ink
  `Stroke`s via programmatic `MutableStrokeInputBatch.add(STYLUS, x, y, t,
  pressure)` + `Stroke(brush, batch)` (API verified against the alpha07
  AARs). Circle = 84-sample ellipse with angular wobble + 0.55 rad closure
  overshoot; arrow = eased bowed shaft + separate 2-wing head stroke;
  underline = sagged segment; all with a sin-profile pressure taper on a
  pressure-pen brush. `injectAiInk(payloadJson)` (replace semantics,
  `holdMs` in payload, default 8 s) + `ValueAnimator` hold→800 ms fade on
  `aiAlpha`, cancel-safe (`clearAi`, re-inject, detach). Manager command
  `injectAiInk` wired. `:app:compileDebugKotlin` green. Unknown kinds
  (e.g. future `label`) are skipped, not errors. Device-visual check rides
  with phase 7.
- 2026-08-07 — Concurrent session note: another agent added finger-vs-
  stylus brush fallback (`activeBrush(event)`) and touch flags to the same
  file mid-phase; merged cleanly on top.
- 2026-08-07 — **Phase 4 done.** `exportFrame(requestId)` rasterizes
  `PaperSurface` (via `View.draw`) + work + annotation — AI layer excluded —
  into an ARGB_8888 bitmap, PNG-compresses to `cacheDir/lumina-frame-<id>.png`
  off the UI thread, and emits `onFrameExported {requestId, path, sceneGraph}`
  (or `{error}`). **Deviation from plan:** the event carries a `file://` path,
  not base64 — RN multipart upload takes a file URI directly and megabyte
  strings over the bridge are pure overhead. Scene graph: `{width_px,
  height_px, density, strokes: [{id: "w0"/"a3", layer, bbox normalized from
  input points}]}`. Bboxes come from raw stroke inputs (brush width not
  added — tight boxes, fine for grounding). Software-canvas note: paper's
  `drawLine` + `CanvasStrokeRenderer` can share the export canvas — the
  mesh-invisibility bug that forced split child views is hardware-only.
  Manager: command `exportFrame`, event `onFrameExported`.
  `:app:compileDebugKotlin` green. Runtime pixel-check rides with phase 7.
- 2026-08-07 — Second concurrent-session merge: paper/ink split into
  `PaperSurface` + `BoardSurface` (hardware-canvas drawMesh bug); export
  path adapted to compose both onto the software canvas.
- 2026-08-07 — **Phase 5 done.** Backend: `AiStroke` Pydantic model
  (semantic primitives, D-024 shape) + `TutorResponse.ai_strokes = []`;
  `services/ai_ink.py` emits a circle inscribing the located bbox (8% pad)
  when `selection_detected`. Round-trip verified: response carries the
  circle centered on the bbox; six legacy fields untouched. Oakland commit
  `e441a2b`.
- 2026-08-07 — **Phase 6 done.** `src/api/tutor.ts` (fetch client:
  `/v1/session/start` lazy + multipart `/v1/tutor/query` with file-URI
  upload, 404-expired-session retry). `LuminaBoard` bridge: commands take
  string args; new `onAnnotationChange`/`onFrameExported` props; web board
  no-ops layer commands. `LessonRunner`: sparkles toggle enters ANNOTATE,
  send bar ("circle the part first" → "ask lumina" / cancel), exportFrame →
  upload → hint into `TutorBubble` (`needs_review` mapping) + `injectAiInk`
  → `clearAnnotation` + back to WRITE; demo idle-evaluate suppressed while
  asking. `tsc --noEmit` green.
- 2026-08-07 — **Phase 7 (scriptable part) done; visual pass pending.**
  Verified: backend round-trip with `ai_strokes` (noop); `:app:assembleDebug`
  green; APK installs + boots on `emulator-5554` against the running Metro;
  no `ReactNativeJS` errors; LessonRunner renders (paper + problem card);
  ask-mode UI works end-to-end (sparkles toggles, send bar states);
  stroke-end events flow native→JS (demo evaluate reacts to injected
  swipes). **Not concluded:** on-screen ink visibility + the full
  circle→ask→AI-ink visual round-trip — a concurrent agent session was
  hot-reloading the same emulator mid-walkthrough (app state reset between
  taps), and finished-stroke visibility on hardware canvas is that
  session's active workstream. Re-run the golden path (write → sparkles →
  circle → ask lumina → red AI circle fades ~8 s; `curl backend + adb`)
  once the emulator is quiet and the rendering fix lands.
- 2026-08-08 — **App de-hardcoded; every tutor path is the real backend.**
  By user direction: `LessonRunner`'s simulated `evaluate()` (canned hint
  array + forceCorrect) is gone. Idle-read, "I'm stuck" and circle-and-ask
  all rasterize via `exportFrame` and hit `/v1/tutor/query`; requestId
  prefix (`read`/`ask`) routes the response. Backend `HintResult` +
  `TutorResponse` gained `status` (correct/incorrect/needs_review) and
  `misconception` (snake_case id) — the reasoner assesses the visible work
  in the same call; reasoner now also runs on annotation-less reads
  (target: "the student's overall working"). `preferences.demoAlwaysCorrect`
  survives as the labeled offline demo toggle. Noop canned hints gained
  statuses/misconception ids (still the documented offline demo mode).
  Verified: noop round-trip asserts status+misconception; `tsc` green.
  `ANTHROPIC_API_KEY` now lives in `backend/.env` (gitignored, Oakland
  `b71e1eb`); backend runs with `TUTOR_*_PROVIDER=anthropic` on
  `claude-sonnet-4-6` (D-025). Assessment commit: Oakland `d8863a9`.
- 2026-08-08 — **Phase 7 visual pass done; AI-ink-never-rendered bug found
  and fixed.** Root cause: alpha07's `MutableStrokeInputBatch.add(type, x,
  y, elapsedTimeMillis, ...)` takes `strokeUnitLengthCm` as the **5th
  positional parameter** — `pressure` is 6th. Both `AiInkSynthesizer.stroke`
  and `LuminaBoardView.addPoint` passed pressure positionally, so it landed
  in `strokeUnitLengthCm`: the synthesizer's varying taper (0.3→0.7) made
  the native runtime reject the whole batch ("all inputs must report the
  same stroke_unit_length" — injectAiInk died as an unhandled ViewCommand
  exception, invisible to JS), and the touch path's constant finger
  pressure explains both the historic "1cm vs 0cm" batch rejections and
  pressurePen never seeing real stylus pressure. Fix: named `pressure =`
  args in both call sites + try/catch per primitive in `synthesize` so one
  rejected stroke can't kill the whole injection. Emulator golden path now
  verified end-to-end: write → idle-read auto-evaluates → sparkles →
  red circle annotation → "ask lumina" → annotation cleared, terracotta
  AI circle (wobble/taper, #C4685E) renders above work ink → fades out
  after the 8 s hold. Backend serving noop canned hints during the pass
  (100–160 ms responses); wire format identical to the anthropic path.
- 2026-08-08 — **Upload bug found + fixed; real-Claude loop closed.**
  Expo SDK 57's WinterCG global fetch rejects RN's legacy `{uri, name,
  type}` FormData parts ("Unsupported FormDataPart implementation" —
  thrown in `expo/src/winter/fetch/convertFormData.ts`). Fix without a
  native rebuild: `queryTutor` uploads via `XMLHttpRequest` (rides RN's
  classic networking stack, streams file:// URIs; 120 s timeout).
  Long-term alternative: `expo-file-system`'s `File` (implements Blob) —
  needs the package installed + APK rebuild.
- 2026-08-08 — **Latency 90 s → 13 s** (Oakland `8fcd22c`): frames
  downscaled to 1568 px before the API (Sonnet's effective ceiling),
  vision/pointing at `effort: medium`, and a new `mode` form part —
  `read` (idle/I'm-stuck) skips the pointing call, `ask` keeps grounding
  + `ai_strokes`. Live read verified end-to-end: real handwriting frame →
  Sonnet assessment (`incorrect`/`incorrect_sum`) + Socratic hint in 13 s.
