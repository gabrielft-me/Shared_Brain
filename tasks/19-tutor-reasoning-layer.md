# Task 19 — Tutor reasoning layer (§16 backend)

**Status:** done (code + noop verified; 20-frame manual review awaits live-key run)
**Depends on:** 17, 18, 20 (session state)

## Goal
`services/tutor.py` currently returns a single canned Socratic prompt.
Plan §16 and §12 require a real reasoning layer that turns
`(Understanding, target, session history)` into a pedagogical hint.

## Deliverables
- `TutorReasoner.hint(understanding, target_description, session_history) -> HintResult`
  returning `{ hint: str, follow_up_questions: list[str], next_step: enum }`.
- System-prompted Claude call with:
  - The composite image.
  - A concise system prompt encoding the Oakland tutoring rubric (Socratic,
    never give the answer, one hint at a time, adapt to grade level).
  - The last N turns of session history from Task 20.
- Guard-rails: if the model returns the numerical answer, retry once with
  a stricter prompt; if it still leaks the answer, fall back to a generic
  probing question and log a rubric violation.
- Deterministic replay path for the demo fixtures so Task 13 keeps working.

## Acceptance
- Manual review of 20 diverse composite frames: no direct answer given;
  each hint targets the circled region.
- Rubric-violation log is empty in the happy path.

## Log
- 2026-08-07 — `TutorReasoner.hint(image, understanding, target_description,
  session_history) -> HintResult {hint, follow_up_questions, next_step}`
  (`next_step` ∈ probe/hint/encourage/review, Pydantic model in `schemas.py`).
  `NoopTutorReasoner` is the deterministic replay path for the Task 13 demos
  (same three hints as before, plus follow-ups). `AnthropicTutorReasoner`:
  composite image + Oakland Socratic rubric system prompt (never give the
  answer, one hint/turn, ≤2 sentences, adapt to grade level, build on
  history) + last 6 session turns from Task 20. `TUTOR_REASONING_PROVIDER`
  env, default `noop`.
- 2026-08-07 — Guard-rail: regex leak check (`answer is`, `x = <n>`,
  `equals <n>` — deliberately not bare `= <n>` so restating the problem's own
  equation doesn't trip it) → strict retry once → generic-probe fallback, with
  `tutor.rubric` logger warnings on both violations. Heuristics unit-tested.
- 2026-08-07 — 20-frame manual rubric review pending a machine with
  `ANTHROPIC_API_KEY`.
