# Task 09 — Verify Android build under the new toolchain

**Status:** done
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
- 2026-08-07 — Created `local.properties` (`sdk.dir=/Users/gabrielfagundes/Library/Android/sdk`), git-ignored.
- 2026-08-07 — First `./gradlew :app:assembleDebug`: 89 s wall (Gradle 8.9, JDK 21 JBR, all 37 tasks UP-TO-DATE — meaning a previous local build was already cached). APK present at `app/build/outputs/apk/debug/app-debug.apk`, 18 MB.
- 2026-08-07 — Only warning: "This Android Gradle plugin (8.7.3) was tested up to compileSdk = 35." Fixed in-place per AGP team guidance by adding `android.suppressUnsupportedCompileSdk=36` to `gradle.properties`. Not a lint-suppression / not a hidden error — the AGP team documents this exact flag as the way to acknowledge running compileSdk=36 on AGP 8.7.3 until we bump AGP. Cross-references D-013 (why we're on compileSdk 36).
- 2026-08-07 — Follow-up `./gradlew :app:clean :app:assembleDebug :app:lintDebug`: 40 s wall, 49 actionable tasks (27 executed, 22 from cache). One benign `stripDebugDebugSymbols` note about `libandroidx.graphics.path.so` — comes from an androidx transitive dep, harmless for debug builds.
- 2026-08-07 — Lint: **0 errors**, 13 warnings (all `GradleDependency` — newer versions of core-ktx / activity-compose / lifecycle-* available). Acceptance criteria met.
- 2026-08-07 — During task 09 the user pushed `d636a01` (Phase 7 partial delivery: `SessionState`, `FrameChangeDetector`, `SessionSummary` UI, backend scenario branching). Rebased my `gradle.properties` change on top cleanly; `./gradlew :app:assembleDebug` still `BUILD SUCCESSFUL` (12 s incremental) so the toolchain verification still holds. Follow-up tasks 11, 12, 14 partially delivered by that commit — status update belongs to those tasks, not this one.

## Toolchain snapshot
- Gradle 8.9 (via committed wrapper — D-016)
- AGP 8.7.3 + Kotlin 2.0.21 + Compose Compiler via `org.jetbrains.kotlin.plugin.compose` (D-013)
- compileSdk / targetSdk 36, minSdk 26
- JDK: OpenJDK 21.0.9 (JetBrains Runtime, `/usr/bin/java`)
- Android SDK: `/Users/gabrielfagundes/Library/Android/sdk`, `platforms/android-36`, `build-tools/{35.0.0,36.0.0,36.1.0}`
