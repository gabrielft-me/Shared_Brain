# Task 22 — Backend HTTPS deployment; drop cleartext on the client

**Status:** todo
**Depends on:** 17, 18, 19, 20 (deploy after real intelligence is in)

## Goal
`memory/decisions.md` D-014 explicitly says: "a real deployment must switch
to HTTPS + drop the flag before ship". Today the client runs with
`usesCleartextTraffic="true"` and points at `10.0.2.2:8000`. This task
retires both.

## Deliverables
- Backend containerized (`backend/Dockerfile`) and deployed behind HTTPS
  (Cloud Run / Fly.io / Render — pick one, document in `memory/decisions.md`).
- Health check `GET /healthz` reachable over the public URL.
- Auth: bearer-token check on `/v1/*` endpoints (rotate via env var).
- Rate limit per session id (e.g. 10 hints/min).
- Client changes:
  - `BuildConfig.TUTOR_BASE_URL` switched via build variant: `debug` still
    hits `http://10.0.2.2:8000/`, `release` hits the HTTPS URL.
  - `android:usesCleartextTraffic="false"` on release; `network_security_config.xml`
    permitting cleartext only for `10.0.2.2` on debug.
  - `Authorization: Bearer …` header from a `BuildConfig` field or, better,
    from a login flow (out of scope for this task — file follow-up).
- New decision entry `D-017` recording the chosen host + auth model.

## Acceptance
- Release APK cannot reach `http://…`; makes the round-trip over HTTPS.
- Debug APK still works against the local backend unchanged.
- Rate-limit trips return HTTP 429 with a JSON body the client can render.

## Log
- (fill in when working)
