# Task 12 — Client-side session state machine (§19 step 31)

**Status:** todo
**Depends on:** 11
**Blocks:** 14

## Goal
Replace the implicit mode transitions in `OverlayManager` with an explicit
`SessionState` machine so we can reason about, log, and demo the tutor's
behavior. Called out in `memory/current-state.md` (client session state
missing) and plan §19 step 31.

## Deliverables
- `tutor/SessionState.kt`:
  ```
  sealed interface SessionState {
      data object Idle : SessionState                     // service not started
      data object Passive : SessionState                  // bubble visible, monitoring
      data object Annotating : SessionState               // canvas attached
      data object Capturing : SessionState                // composite in flight
      data class AwaitingResponse(val requestId: String) : SessionState
      data class ShowingHint(val response: TutorResponse) : SessionState
      data class Error(val message: String) : SessionState
  }
  ```
- `tutor/SessionStateMachine.kt` — `StateFlow<SessionState>`; single mutator
  `transition(next)`; illegal transitions log + no-op.
- `OverlayManager` and `TutorSession` become consumers of the state flow,
  not owners of ad-hoc booleans.
- History of the last N transitions kept in-memory for the teacher summary
  (Task 14).

## Acceptance
- Every state change is observable via a single `Flow`.
- Attempting `Annotating → ShowingHint` without going through `Capturing`
  logs a warning and does not mutate state.
- Rotating the device does not desync the machine (no double-attach).

## Log
- (fill in when working)
