# Task 06 — Phase 6: AI pipeline (client side)

**Status:** done
**Depends on:** 02, 05
**Blocks:** 07

## Goal
Send composite frame + geometry metadata to backend, receive normalized point/bbox + tutor hint, render `TutorCardOverlay` at the mapped position. Plan §6, §12, §13, §19 Phase 6.

## Deliverables
- `network/TutorApi.kt` — Retrofit interface:
  - `@Multipart POST /v1/tutor/query` with `image`, `geometry` (JSON part), returns `TutorResponse`.
- `network/TutorClient.kt` — configured Retrofit + OkHttp (30s call timeout, gzip). Base URL from `BuildConfig.TUTOR_BASE_URL`.
- `tutor/TutorResponse.kt` — matches plan §12 pointing schema + `hint: String`.
- `tutor/TutorSession.kt` — orchestrates: ANNOTATE done → capture → composite → upload → mapper → `showTutorCard()`.
- Coroutine scope tied to `TutorOverlayService` lifecycle.

## Acceptance
- End-to-end: circle expression → card appears near it with hint text.
- Malformed / non-200 responses do not crash the service; user sees a small error toast/card.

## Log
- 2026-08-07 — `network/TutorApi.kt`: Retrofit `@Multipart POST v1/tutor/query` with image part + geometry JSON part.
- 2026-08-07 — `network/TutorClient.kt`: OkHttp (30s call timeout, HTTP logging in debug) + Moshi + `TutorClient.query(bitmap, geometry)`; base URL from `BuildConfig.TUTOR_BASE_URL` (defaults to `http://10.0.2.2:8000/` for emulator).
- 2026-08-07 — `tutor/TutorResponse.kt`: matches plan §12 schema (`selection_detected`, `target_description`, `point`, `bbox`, `confidence`, `hint`) + `Geometry` DTO.
- 2026-08-07 — `tutor/TutorSession.kt`: capture → composite → upload → `onResult(TutorResponse)`; the overlay layer renders the returned point via `OverlayManager.showTutorCard()`. On failure returns a synthetic response with an error hint so the UI still surfaces feedback.
- 2026-08-07 — Backend counterpart under `backend/` is the Task 08 stub; contract types mirror on both sides.
