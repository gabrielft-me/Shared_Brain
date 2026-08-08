# Task 19 — Tutor reasoning layer (§16 backend)

**Status:** todo
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
- (fill in when working)
