# Task 26 — Error marks: line-by-line check → discreet overlay pointers

**Status:** done (backend noop path smoke-tested; RN typecheck clean;
2026-08-08)
**Depends on:** 17/19 (vision + reasoner), 25 (exportFrame path)
**Related decisions:** D-024, D-025, D-027

## Goal

Submit and "I'm stuck" (both already share `evaluate()` → `exportFrame` →
`mode=read`) now also SHOW where the work breaks, not just say it in the
bubble: the reasoner checks the student's work line by line and, when
`status == incorrect`, returns up to 3 normalized bboxes around the
handwritten line(s)/term(s) where the work first goes wrong. The client
renders them as discreet translucent pointers — an RN overlay, never ink.
Location only, no corrections (rubric still forbids revealing answers).

## Contract change (wire)

`TutorResponse` gains:

```json
"error_marks": [ { "bbox": { "x": 0.42, "y": 0.35, "width": 0.12, "height": 0.08 } } ]
```

- Populated only when `status == "incorrect"`; empty otherwise. Returned in
  BOTH `read` and `ask` modes (in ask it complements the AI-ink circle).
- No extra model round-trip: the marks ride on the existing reasoner call
  (`_RawHintResult`, unconstrained floats clamped server-side, max 3,
  zero-area boxes dropped — same pattern as pointing's `_RawLocation`).
- Older backends without the field: client defaults `error_marks` to `[]`.

## Implementation

Backend (`backend/app/`):
- `schemas.py` — `ErrorMark {bbox}`; `error_marks` on `HintResult` and
  `TutorResponse`.
- `services/tutor.py` — TUTOR_SYSTEM gains the error_marks rule (box the
  student's own wrong writing, never where the correction goes); model-facing
  `_RawHintResult` + `_clamped_marks`; canned Noop results carry marks that
  reuse the NoopTargetLocator demo coords so the offline path exercises the
  overlay end-to-end.
- `main.py` — passes `result.error_marks` through.

Client (`lumina-app/src/`):
- `api/tutor.ts` — `ErrorMarkPayload`, field on `BackendTutorResponse`,
  `??= []` back-compat.
- `components/LessonRunner.tsx` — `errorMarks` state; overlay is an
  absolute-fill `pointerEvents="none"` View over the board (normalized frame
  coords map 1:1 to the full-screen board), each mark a MotiView translucent
  rounded region (`rgba(178,58,72,0.07)` fill, hairline border, small dot at
  the left edge) with staggered fade-in. Cleared on: new evaluation, new
  ask, advance, board emptied (strokeCount 0), status != incorrect.
- Deliberately NOT `injectAiInk`: AI ink stays reserved for circle&ask
  explanations (D-024 semantics); marks are ephemeral UI, not tutor drawing.

## Log

- 2026-08-08 — Implemented as above. Verified: FastAPI TestClient noop run
  (`mode=read`, scenario 2) returns `status=incorrect` + 1 mark and 0
  ai_strokes; clamp drops zero-width boxes and negative coords; `tsc
  --noEmit` clean. NOTE: the Kotlin `tutor/TutorResponse.kt` DTO is stale
  dead code from the Compose app (lacks ai_strokes/status too) — the live
  contract is `lumina-app/src/api/tutor.ts`; Kotlin DTO left untouched.
