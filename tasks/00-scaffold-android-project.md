# Task 00 — Scaffold Android project

**Status:** todo
**Depends on:** —
**Blocks:** all phase tasks

## Goal
Create a build-ready Android project skeleton matching plan §16–§17, so subsequent phase tasks have a place to live.

## Deliverables
- Root Gradle Kotlin DSL: `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`.
- `app/` module with:
  - `build.gradle.kts` — Kotlin, Compose, `minSdk = 26`, `targetSdk = 34`, `compileSdk = 34`, `applicationId = "com.oakland.tutor"`.
  - `src/main/AndroidManifest.xml` with:
    - `SYSTEM_ALERT_WINDOW`
    - `FOREGROUND_SERVICE`
    - `FOREGROUND_SERVICE_MEDIA_PROJECTION`
    - `INTERNET`
    - `<application>` with `MainActivity`, `TutorOverlayService`, `ScreenCaptureService` (foregroundServiceType="mediaProjection").
  - Package tree from plan §17: `permissions/`, `overlay/`, `ink/`, `capture/`, `network/`, `tutor/`, `ui/setup`, `ui/summary`.
- `MainActivity.kt` — minimal Compose scaffold with three buttons: grant overlay permission → grant projection consent → start/stop session.

## Acceptance
- Gradle files parse (structure only; not required to build in this environment).
- All package folders exist with placeholder files where needed for Kotlin package validity.
- Manifest declares the two services and all permissions above.

## Log
- (fill in when working)
