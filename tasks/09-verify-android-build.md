# Task 09 — Verify Android build under the new toolchain

**Status:** todo
**Depends on:** 00 (scaffold), toolchain bumps in D-013 / D-016
**Blocks:** 10

## Goal
Prove that the scaffold compiles cleanly under AGP 8.7.3 / Kotlin 2.0.21 /
`compileSdk = 36` (see `memory/decisions.md` D-013, D-016). Today the wrapper
JAR is committed but the build has never actually been run — flagged in
`memory/current-state.md` as *Not verified → Android build*.

## Deliverables
- `local.properties` (git-ignored) with `sdk.dir=/Users/gabrielfagundes/Library/Android/sdk`, or `ANDROID_HOME` set in the shell.
- `./gradlew :app:assembleDebug` completes without errors.
- Any Compose Compiler / dependency / manifest merger errors surfaced by the
  build fixed in-place; do **not** paper over them with `--warning-mode=none`
  or by silencing lint.
- Record wall-clock time + Gradle version + JDK version in `## Log`.

## Acceptance
- `app/build/outputs/apk/debug/app-debug.apk` exists after the build.
- `./gradlew :app:lintDebug` runs (warnings acceptable, errors not).

## Log
- (fill in when working)
