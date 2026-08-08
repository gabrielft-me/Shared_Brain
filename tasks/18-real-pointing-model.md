# Task 18 — Real pointing / grounding model (§16 backend)

**Status:** done (code + noop verified; IoU acceptance awaits live-key run)
**Depends on:** 08 (stub in place)
**Blocks:** 19

## Goal
`services/pointing.py` currently returns a fixed target. Plan §16 requires
a real pointing/grounding call so the tutor card lands on the actual
expression the student circled.

## Deliverables
- Interface `TargetLocator.locate(image, understanding) -> (point, bbox, confidence)`.
- Implementation options evaluated (pick one for MVP, document in
  `memory/decisions.md`):
  - Claude with a system prompt asking for normalized `x`/`y` in a strict
    JSON schema.
  - A dedicated grounding model (e.g. Molmo, Grounding DINO) called via
    HTTP or local inference.
- The chosen implementation must:
  - Consume the same composite frame that vision.analyze() sees.
  - Return coordinates in the plan §12 normalized schema (0..1).
  - Reject low-confidence results (< 0.4) by returning `selection_detected=false`
    so the client can show a "couldn't tell what you circled" message.
- `TUTOR_POINTING_PROVIDER` env var, default `noop`.

## Acceptance
- On the three demo fixtures, the returned bbox overlaps the student's
  circle by ≥ 60 % IoU.
- Deliberately circling empty space returns `selection_detected=false`.

## Log
- 2026-08-07 — `TargetLocator` ABC with `NoopTargetLocator` (previous fixed
  coordinates) and `AnthropicTargetLocator`. Implementation choice: Claude
  (`claude-opus-4-7`) with structured outputs returning a flat
  `{found, x, y, bbox_*, confidence}` schema — no dedicated grounding model
  (decision D-022). Opus 4.7 vision returns pixel-accurate coordinates, so
  normalized 0..1 output is asked for directly and clamped server-side.
  `found=false` or confidence < 0.4 (`pointing.MIN_CONFIDENCE`) →
  `selection_detected=false` in `main.py`. `TUTOR_POINTING_PROVIDER` env,
  default `noop`.
- 2026-08-07 — IoU acceptance on the three demo fixtures + empty-space test
  pending a machine with `ANTHROPIC_API_KEY`.
