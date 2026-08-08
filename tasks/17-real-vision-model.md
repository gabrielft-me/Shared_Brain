# Task 17 — Real vision-understanding model (§16 backend)

**Status:** todo
**Depends on:** 08 (stub in place)
**Blocks:** 19 (tutor reasoning consumes the understanding)

## Goal
Replace `backend/app/services/vision.py`'s canned return with a real
multimodal call. Plan §16 backend stack; `memory/current-state.md` flags
this as *Missing → Backend intelligence is stub*.

## Deliverables
- Provider-agnostic interface `VisionAnalyzer` with implementations:
  - `AnthropicVisionAnalyzer` — Claude with the composite image + system
    prompt asking for structured JSON: math domain, expression(s) present,
    student's apparent progress, likely misconception candidates.
  - `NoopVisionAnalyzer` — retains current stub for offline / demo mode.
- `services/vision.py` returns a typed `Understanding` (new Pydantic model
  in `schemas.py`) instead of a bare dict.
- Selection via `TUTOR_VISION_PROVIDER` env var, default `noop`.
- API keys read from env; never checked in.

## Acceptance
- With `TUTOR_VISION_PROVIDER=anthropic` and a valid key, hitting
  `/v1/tutor/query` with the distributive-property fixture returns an
  `Understanding` naming the distributive property.
- With `TUTOR_VISION_PROVIDER=noop`, behavior is unchanged.

## Log
- (fill in when working)
