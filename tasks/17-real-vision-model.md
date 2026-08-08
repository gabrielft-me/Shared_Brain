# Task 17 — Real vision-understanding model (§16 backend)

**Status:** done (live-verified 2026-08-08 on claude-sonnet-4-6)
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
- 2026-08-07 — `VisionAnalyzer` ABC with `NoopVisionAnalyzer` (canned scenarios
  1/2/3, now enriched with domain/progress/misconceptions) and
  `AnthropicVisionAnalyzer` (Claude `claude-opus-4-7`, adaptive thinking,
  structured outputs via `messages.parse(output_format=Understanding)`,
  prompt-cached system prompt). Typed `Understanding` Pydantic model added to
  `schemas.py`. Selection via `TUTOR_VISION_PROVIDER` (default `noop`);
  model override via `TUTOR_MODEL`. Key read from `ANTHROPIC_API_KEY` only.
- 2026-08-07 — Noop acceptance verified end-to-end against the frontend
  contract. Anthropic acceptance (distributive fixture names the distributive
  property) not yet run: no `ANTHROPIC_API_KEY` on this machine.
- 2026-08-08 — Live smoke pass on `claude-sonnet-4-6` (D-025): a real
  `exportFrame` PNG (actual student handwriting pulled from the emulator)
  produced a faithful Understanding — the reasoner's hint referenced the
  exact expressions on the page. Formal distributive-fixture acceptance
  still worth a scripted run.
