# Repository Guidelines

## Project Structure & Module Organization
- `app/src/main/java/com/roubao/autopilot/`: Kotlin source, grouped by feature (`agent/`, `controller/`, `data/`, `skills/`, `tools/`, `ui/`, `vlm/`).
- `app/src/main/res/`: Android resources (themes, strings, drawables, XML configs).
- `app/src/main/assets/skills.json`: Skill definitions and app mappings.
- `docs/`: Demo GIFs, screenshots, and branding assets.
- Root: Gradle Kotlin DSL build files (`build.gradle.kts`, `app/build.gradle.kts`).

## Build, Test, and Development Commands
- `./gradlew assembleDebug`: Build a debug APK.
- `./gradlew assembleRelease`: Build a release APK.
- `./gradlew installDebug`: Install debug build to a connected device.
- `./gradlew clean`: Clean build outputs.
- `./gradlew tasks`: List available Gradle tasks.
- CLI builds require JDK 17. If `java` is missing on PATH, use:
  - `GRADLE_USER_HOME=.gradle-user JAVA_HOME=.jdk/jdk-17.0.17+10 ./gradlew assembleDebug`
  - `GRADLE_USER_HOME=.gradle-user JAVA_HOME=.jdk/jdk-17.0.17+10 ./gradlew assembleRelease`
- Convenience wrapper: `scripts/build.sh debug|release` (expects JDK 17 at `.jdk/jdk-17.0.17+10`).

## Architecture Overview
- Dual-layer agent design: Skills route intent to either DeepLink delegation or GUI automation loops.
- Tools provide atomic actions (search apps, open app, deep link, clipboard, shell, HTTP).
- Agent loop lives in `app/src/main/java/com/roubao/autopilot/agent/` and is coordinated by `MobileAgent.kt`.
- Device control is centralized in `app/src/main/java/com/roubao/autopilot/controller/DeviceController.kt` via Shizuku.

## Security & Permissions
- Shizuku is required for system-level commands (screenshots, taps, input); ensure it is running before testing.
- Required permissions include overlay (`SYSTEM_ALERT_WINDOW`) and app discovery (`QUERY_ALL_PACKAGES`).
- API keys are stored via encrypted preferences; confirm settings are populated before agent runs.

## Coding Style & Naming Conventions
- Kotlin + Jetpack Compose; follow Android Studio defaults (4-space indentation).
- Package naming follows `com.roubao.autopilot.<feature>`.
- Classes use `UpperCamelCase`, functions/vars `lowerCamelCase`.
- Resources use `snake_case` (e.g., `splash_icon.png`).
- Logging uses `println("[ComponentName] message")` for lightweight debug traces.

## Testing Guidelines
- No automated test suite yet; verify manually on device/emulator.
- Checklist: Shizuku running, API key configured, overlay/notification/query permissions granted, network available for VLM calls.

## Commit & Pull Request Guidelines
- Commit messages follow Conventional Commits: `feat: ...`, `fix: ...`, `docs: ...`, `chore: ...` (Chinese summaries are common in history).
- PRs should include: clear summary, steps to verify, and screenshots/GIFs for UI changes. Note the Android version/device used for manual testing.

## Configuration Tips
- VLM API setup is in the app settings; ensure base URL/model are valid before running agent flows.
