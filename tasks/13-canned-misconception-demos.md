# Task 13 — Three canned misconception demos (§19 step 32)

**Status:** todo
**Depends on:** 10 (needs a live install), 12 (session state for scripted flow)

## Goal
Ship three reproducible demos that make the tutor visibly useful without
depending on real ML. Plan §19 step 32.

## Deliverables
- Three demos under `Shared_Brain/demos/`:
  1. **Distributive property** — `3(x + 4) = 21`; misconception: only
     multiply the `x`. Hint: "What else does the 3 multiply inside here?"
  2. **Sign flip on subtraction** — `5 - (2x - 3) = ?`; misconception:
     drops the sign. Hint: "What happens to the −3 when you distribute the
     minus sign?"
  3. **Fraction addition** — `1/2 + 1/3`; misconception: sums numerators
     and denominators. Hint: "Do the denominators need to match first?"
- Each demo folder contains: `screen.png` (target screenshot at emulator
  resolution), `annotation.json` (fixture strokes), `expected-response.json`
  (`TutorResponse`), `README.md` (steps to run).
- Backend flag `TUTOR_DEMO_MODE=1` that pattern-matches the incoming image
  hash against the fixtures and returns the fixture response instead of
  the generic stub. This is the only place stub → fixture branching lives.

## Acceptance
- With the emulator on the demo screenshot and demo mode on, the tutor
  card shows the correct hint for each of the three demos.
- With demo mode off, behavior is unchanged.

## Log
- (fill in when working)
