# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Roubao (肉包) is an open-source AI phone automation assistant built natively for Android. It's a Kotlin rewrite of Alibaba's MobileAgent framework, running entirely on-device without requiring a PC connection. The app uses Vision Language Models (VLM) to understand screen content and execute complex automation tasks through natural language commands.

**Key Differentiator**: Unlike Python-based solutions that require a computer and ADB connection, Roubao runs natively on Android using Shizuku for system-level permissions.

## Build Commands

```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK
./gradlew assembleRelease

# Install debug build to connected device
./gradlew installDebug

# Clean build
./gradlew clean

# Check dependencies
./gradlew dependencies
```

### Firebase Configuration (Optional)

Firebase Crashlytics is used for optional crash reporting. For local development:

**Option 1: Disable Firebase (Recommended)**
- Firebase plugins and dependencies are already commented out in `app/build.gradle.kts`
- Firebase imports and calls are commented out in `App.kt` and `SettingsScreen.kt`
- Local crash logs are still captured via `CrashHandler`

**Option 2: Enable Firebase**
1. Download `google-services.json` from Firebase Console
2. Place it in `app/` directory
3. Uncomment Firebase dependencies in `app/build.gradle.kts`:
   - `id("com.google.gms.google-services")`
   - `id("com.google.firebase.crashlytics")`
   - Firebase BOM and crashlytics dependencies
4. Uncomment Firebase imports and code in `App.kt` and `SettingsScreen.kt`

## Architecture

### Three-Layer Agent Framework

Roubao implements a **Tools + Skills + Agent** architecture inspired by Claude Code:

```
User Input → Skills Layer → Tools Layer → Agent Layer → Shizuku → System
```

**1. Skills Layer** (`app/src/main/java/com/roubao/autopilot/skills/`)
- High-level user intent mapping
- Configuration-driven (see `app/src/main/assets/skills.json`)
- Two execution modes:
  - **Delegation**: Direct DeepLink to AI-capable apps (fast path)
  - **GUI Automation**: Traditional screenshot-analyze-operate loop with context guidance
- `SkillManager`: Intent recognition using LLM semantic matching
- `SkillRegistry`: Loads and manages skill definitions from JSON

**2. Tools Layer** (`app/src/main/java/com/roubao/autopilot/tools/`)
- Atomic operations exposed to LLM via Tool definitions
- Each Tool has a name, description, and typed parameters
- Available Tools:
  - `search_apps`: Multi-dimensional app search (pinyin, semantic, category)
  - `open_app`: Launch applications
  - `deep_link`: Navigate to specific app screens via URI
  - `clipboard`: Read/write clipboard
  - `shell`: Execute shell commands via Shizuku
  - `http`: HTTP requests for external API integration
- `ToolManager`: Central registry and execution dispatcher
- Tools are registered in `ToolRegistry` for LLM discovery

**3. Agent Layer** (`app/src/main/java/com/roubao/autopilot/agent/`)
- Ported from Alibaba's MobileAgent-v3
- Multi-agent collaboration pattern:
  - `Manager`: Task planning and decomposition
  - `Executor`: Action decision-making
  - `ActionReflector`: Operation outcome evaluation
  - `Notetaker`: Execution history tracking
- `MobileAgent`: Main execution loop coordinator
- `InfoPool`: Shared state container across agents
- `ConversationMemory`: Maintains multi-turn dialogue context with LLM

### Execution Flow

1. **Skill Matching Phase**:
   - User input → SkillManager analyzes intent using LLM
   - Checks installed apps and selects best match
   - High-confidence delegation skill → Direct DeepLink execution → Done
   - Otherwise → Proceed to Agent loop with skill context

2. **Agent Loop** (up to 25 steps by default):
   - Screenshot via Shizuku
   - Manager plans next action based on VLM analysis
   - Executor decides specific operation (tap/swipe/type/tool call)
   - Execute via DeviceController
   - Reflector evaluates success
   - Update InfoPool and repeat

3. **Safety Mechanisms**:
   - Auto-stop on sensitive screens (payment, passwords)
   - Manual stop via overlay UI
   - Step count limit (default: 25)

### Critical Components

**DeviceController** (`controller/DeviceController.kt`)
- Shizuku UserService integration via AIDL (`IShellService.aidl`)
- System-level command execution:
  - Screenshot: `screencap -p`
  - Tap: `input tap x y`
  - Swipe: `input swipe x1 y1 x2 y2 duration`
  - Type: `input text`
  - App launch: `am start`
- Coordinates normalization for different screen densities
- Root mode detection and su command support

**VLMClient** (`vlm/VLMClient.kt`)
- OpenAI-compatible API client
- Supports multiple VLM providers (Qwen-VL, GPT-4V, Claude)
- Image compression and base64 encoding
- Dynamic model fetching from `/models` endpoint
- Retry logic with exponential backoff

**AppScanner** (`controller/AppScanner.kt`)
- Intelligent app search across multiple dimensions:
  - Package name
  - App name (with pinyin support for Chinese)
  - Semantic similarity
  - App category
- Fuzzy matching with configurable thresholds
- Results cached and ranked by relevance score

### Data Flow

- **Settings**: Encrypted SharedPreferences (`SettingsManager.kt`) using AES-256-GCM for API keys
- **Execution History**: In-memory state flows, persisted as logs
- **Skills Configuration**: JSON asset loaded at runtime
- **UI State**: Jetpack Compose StateFlow pattern

## Key Dependencies

- **Shizuku** (13.1.5): ADB-level permissions without root
- **Jetpack Compose**: Modern Android UI with Material 3
- **OkHttp** (4.12.0): HTTP client for VLM API calls
- **Firebase Crashlytics**: Optional crash reporting (can be disabled)
- **AndroidX Security**: Encrypted key storage
- **Kotlin Coroutines**: Asynchronous execution

## Development Notes

### Adding New Tools

1. Create new Tool class implementing `Tool` interface in `tools/`
2. Define `name`, `description`, `parameterSchema`
3. Implement `suspend fun execute(params: Map<String, Any?>): ToolResult`
4. Register in `ToolManager.initialize()`

### Adding New Skills

1. Add skill definition to `app/src/main/assets/skills.json`:
```json
{
  "id": "unique_id",
  "name": "Display Name",
  "description": "What this skill does",
  "keywords": ["trigger", "words"],
  "category": "Category",
  "related_apps": [
    {
      "package": "com.example.app",
      "name": "App Name",
      "type": "delegation" | "gui_automation",
      "deep_link": "scheme://path" (for delegation),
      "steps": ["Step 1", "Step 2"] (for gui_automation),
      "priority": 100
    }
  ]
}
```
2. Skills are auto-loaded by `SkillRegistry` on app start

### Shizuku Integration

- Service must be running before app use
- Check status: `Shizuku.pingBinder()`
- Request permission: `Shizuku.requestPermission()`
- AIDL service defined in `app/src/main/aidl/com/roubao/autopilot/IShellService.aidl`
- Implementation: `service/ShellService.kt`

### VLM Prompting

The LLM receives:
- System prompt with task description
- Current screenshot (base64 encoded)
- Available tools/skills context
- Conversation history
- Screen dimensions

Response format expected: JSON with action type and parameters

### Testing

Currently no automated test suite. Manual testing workflow:
1. Install Shizuku and grant permissions
2. Configure VLM API key in Settings
3. Test simple commands first ("open Settings")
4. Check logs via Settings → Export Logs

## Project Structure

```
app/src/main/java/com/roubao/autopilot/
├── agent/           # Multi-agent orchestration (Manager, Executor, Reflector, Notetaker)
├── controller/      # Device control (Shizuku, AppScanner)
├── data/            # Settings management, encryption
├── service/         # Shizuku UserService AIDL implementation
├── skills/          # High-level intent handling
├── tools/           # Atomic operations for LLM
├── ui/              # Jetpack Compose screens + overlay service
├── utils/           # Crash handler, utilities
├── vlm/             # VLM API client
└── App.kt           # Application entry point

app/src/main/assets/
└── skills.json      # Skill definitions (declarative configuration)

app/src/main/aidl/
└── IShellService.aidl  # Shizuku service interface
```

## Common Issues

- **Shizuku not connected**: Restart Shizuku service, ensure wireless debugging is enabled (Android 11+)
- **Screenshot fails**: Check `/data/local/tmp` permissions
- **VLM timeout**: Increase timeout in `VLMClient`, check network/API quota
- **App search returns no results**: Rebuild app scanner cache, check pinyin library
- **Coordination errors**: Different devices report different screen sizes; coordinates may need density adjustment

## Version Information

- Current Version: v1.4.2
- Min SDK: 26 (Android 8.0)
- Target SDK: 34
- Kotlin: 1.9.20
- Gradle: 8.2.0
- Java: 17

## Roadmap

**v2.0 (In Development on `roubao2.0+AccessibilityService` branch)**:
- AccessibilityService integration for element-based clicking (more reliable than coordinates)
- UI tree awareness for structured context
- Macro script recording and playback
- No Root requirement for basic operations
