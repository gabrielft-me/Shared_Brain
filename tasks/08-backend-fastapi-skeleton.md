# Task 08 — Backend FastAPI skeleton

**Status:** done (stub)
**Depends on:** —
**Blocks:** 06 (only for real end-to-end)

## Goal
Minimal FastAPI service matching the client's contract from plan §12/§16. Can start as a stub returning canned responses; swap in real vision + pointing models later.

## Deliverables
- `backend/` (Python 3.11+):
  - `pyproject.toml` / `requirements.txt` (fastapi, uvicorn, pydantic, python-multipart, pillow).
  - `app/main.py` — `POST /v1/tutor/query`, multipart image + `geometry` JSON part.
  - `app/schemas.py` — Pydantic models for `Geometry`, `TutorResponse` (mirrors client `tutor/TutorResponse.kt`).
  - `app/services/vision.py` — interface stub `analyze(image) -> understanding`.
  - `app/services/pointing.py` — interface stub `locate(image, understanding) -> point + bbox`.
  - `app/services/tutor.py` — interface stub `hint(understanding, point) -> str`.
  - `app/main.py` initial impl: return a deterministic stub `TutorResponse` so client integration is unblocked.
- `README.md` — `uvicorn app.main:app --reload` instructions.

## Acceptance
- `curl -F image=@screenshot.png -F 'geometry={"screen_width_px":2560,...}' localhost:8000/v1/tutor/query` returns a valid `TutorResponse`.

## Log
- 2026-08-07 — `backend/requirements.txt` (fastapi, uvicorn, pydantic, python-multipart, pillow).
- 2026-08-07 — `backend/app/schemas.py`: Pydantic `Geometry`, `NormalizedPoint`, `NormalizedBox`, `TutorResponse` — mirrors the Kotlin DTOs.
- 2026-08-07 — `backend/app/services/{vision,pointing,tutor}.py`: interface stubs returning deterministic values so client integration is unblocked.
- 2026-08-07 — `backend/app/main.py`: `GET /healthz`, `POST /v1/tutor/query` (multipart image + `geometry` JSON) → `TutorResponse` using the stubs.
- 2026-08-07 — `backend/README.md`: run instructions.
