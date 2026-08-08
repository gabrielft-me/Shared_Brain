# Task 20 — Backend session state (§16 backend)

**Status:** todo
**Depends on:** 08

## Goal
Plan §16 lists "Session state" as a backend responsibility. Today every
`POST /v1/tutor/query` is stateless, so the tutor can't remember what it
just said or notice the student stuck on the same misconception.

## Deliverables
- `app/session.py` — `SessionStore` interface + `InMemorySessionStore`
  (dev) and `SqliteSessionStore` (persistent).
- `POST /v1/session/start` returns `{session_id, started_at}`. The Android
  client stores this in the `TutorSession` and includes it as a
  `session_id` form part on every subsequent `/v1/tutor/query`.
- Each stored session record contains the ordered list of hints returned,
  target descriptions, timestamps, and thumbnails.
- `POST /v1/session/{id}/end` returns the full transcript; the client
  writes it into the local Room summary (Task 14).
- Sessions expire after 2 h idle.

## Acceptance
- Two sequential queries with the same `session_id` return hints that
  reference each other (via Task 19's `session_history`).
- Killing and restarting the backend loses in-memory sessions but keeps
  Sqlite ones.

## Log
- (fill in when working)
