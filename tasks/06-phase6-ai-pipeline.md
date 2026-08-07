# Task 06 — Phase 6: AI pipeline (client side)

**Status:** todo
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
- (fill in when working)
