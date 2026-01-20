# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Roubao (肉包) is an open-source AI-powered Android automation assistant that runs entirely on-device without requiring a PC connection. It uses Vision Language Models (VLM) to understand screen content and autonomously operate the phone to complete user tasks.

**Key Differentiator**: Native Kotlin implementation (ported from Python-based MobileAgent-v3) that runs directly on Android using Shizuku for system-level permissions.

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

# List all tasks
./gradlew tasks
```

## Development Environment

- **Language**: Kotlin
- **Min SDK**: 26 (Android 8.0)
- **Target SDK**: 34
- **Compile SDK**: 34
- **JDK**: 17
- **Android Studio**: Hedgehog or later
- **UI Framework**: Jetpack Compose with Material 3
- **Build System**: Gradle with Kotlin DSL

## Core Architecture

Roubao implements a **dual-layer agent framework** inspired by Claude Code:

### 1. Skills Layer (User Intent)
- **Purpose**: Maps natural language to high-level tasks
- **Location**: `app/src/main/java/com/roubao/autopilot/skills/`
- **Configuration**: `app/src/main/assets/skills.json`
- **Key Components**:
  - `SkillManager.kt` - Intent recognition and skill routing
  - `SkillRegistry.kt` - Skill configuration and app matching
  - `Skill.kt` - Skill interface definition

**Two Execution Modes**:
1. **Delegation Mode**: High-confidence matches directly open AI-capable apps via DeepLink (e.g., opening Xiaomei AI for food ordering)
2. **GUI Automation Mode**: Traditional screen analysis loop for non-AI apps

### 2. Tools Layer (Atomic Capabilities)
- **Purpose**: Provides primitive operations that agents can use
- **Location**: `app/src/main/java/com/roubao/autopilot/tools/`
- **Key Components**:
  - `ToolManager.kt` - Tool registry and execution
  - `Tool.kt` - Tool interface definition

**Available Tools**:
- `SearchAppsTool` - Smart app search (pinyin, semantic matching)
- `OpenAppTool` - Launch applications
- `DeepLinkTool` - Direct navigation via DeepLinks
- `ClipboardTool` - Read/write clipboard
- `ShellTool` - Execute shell commands via Shizuku
- `HttpTool` - HTTP requests to external APIs

### 3. Agent Layer (GUI Automation)
- **Purpose**: Multi-agent system for visual reasoning and action planning
- **Location**: `app/src/main/java/com/roubao/autopilot/agent/`
- **Based on**: Alibaba MobileAgent-v3 (ported from Python to Kotlin)

**Agent Components**:
- `MobileAgent.kt` - Main execution loop coordinator
- `Manager.kt` - High-level planning agent (analyzes goals, decomposes tasks)
- `Executor.kt` - Decision agent (determines next action from screen state)
- `ActionReflector.kt` - Reflection agent (evaluates action outcomes)
- `Notetaker.kt` - Memory agent (records progress and context)
- `InfoPool.kt` - Shared state pool across agents
- `ConversationMemory.kt` - Maintains conversation history for LLM context

**Execution Flow**:
```
User Input → Skill Matching
  ├─ High Confidence Delegation Skill → DeepLink → Done
  └─ Standard Agent Loop:
      1. Screenshot (Shizuku)
      2. Manager: Plan strategy
      3. Executor: Decide next action
      4. Execute: tap/swipe/type/open_app
      5. Reflector: Evaluate outcome
      6. Loop until complete or safety limit (max 25 steps)
```

### 4. Device Control Layer
- **Location**: `app/src/main/java/com/roubao/autopilot/controller/`
- **Key Components**:
  - `DeviceController.kt` - Shizuku integration for system commands (screenshot, tap, swipe, input text, etc.)
  - `AppScanner.kt` - Multi-dimensional app search (package name, app name, pinyin, category, semantic similarity)

### 5. VLM Integration
- **Location**: `app/src/main/java/com/roubao/autopilot/vlm/`
- **Component**: `VLMClient.kt`
- **Protocol**: OpenAI-compatible API (supports GPT-4V, Qwen-VL, Claude, etc.)
- **Features**:
  - Vision + text multimodal requests
  - Dynamic model discovery via `/models` endpoint
  - Retry logic with exponential backoff
  - Base64 image encoding for screenshots

### 6. UI Layer
- **Location**: `app/src/main/java/com/roubao/autopilot/ui/`
- **Framework**: Jetpack Compose with Material 3
- **Key Screens**:
  - `HomeScreen.kt` - Main task input interface
  - `SettingsScreen.kt` - API configuration, model selection, safety settings
  - `HistoryScreen.kt` - Execution history with step-by-step logs
  - `CapabilitiesScreen.kt` - Available skills display
  - `OnboardingScreen.kt` - First-time user guide
  - `OverlayService.kt` - Floating window during task execution

### 7. Data Layer
- **Location**: `app/src/main/java/com/roubao/autopilot/data/`
- **Component**: `SettingsManager.kt`
- **Storage**: EncryptedSharedPreferences (AES-256-GCM)
- **Security**: API keys are encrypted at rest

## Key Dependencies

- **Shizuku** (`dev.rikka.shizuku:api:13.1.5`) - System-level permissions without root
- **OkHttp** (`com.squareup.okhttp3:okhttp:4.12.0`) - VLM API communication
- **Jetpack Compose** - Declarative UI framework
- **Coroutines** (`kotlinx-coroutines-android:1.7.3`) - Async operations
- **Security Crypto** (`androidx.security:security-crypto`) - Encrypted storage
- **Firebase Crashlytics** - Optional crash reporting (user-configurable)

## Shizuku Integration

**Critical**: This app requires Shizuku for all device control operations. Shizuku provides ADB-level permissions without root access.

**Required Permissions**:
- `moe.shizuku.manager.permission.API_V23` - Shizuku API access
- `QUERY_ALL_PACKAGES` - App discovery
- `SYSTEM_ALERT_WINDOW` - Overlay during execution

**Shell Commands Used** (via Shizuku):
- `screencap` - Capture screenshots for VLM analysis
- `input tap X Y` - Simulate touch events
- `input swipe X1 Y1 X2 Y2 duration` - Simulate gestures
- `input text "..."` - Input text to fields
- `am start -n package/activity` - Launch apps
- `am start -a VIEW -d "deeplink://..."` - DeepLink navigation
- `su -c` (Root mode only) - Execute privileged commands

## Adding New Skills

Skills are defined in `app/src/main/assets/skills.json`. Each skill specifies:

```json
{
  "id": "unique_id",
  "name": "Display Name",
  "description": "What this skill does",
  "category": "Category",
  "keywords": ["keyword1", "keyword2"],
  "params": [
    {
      "name": "param_name",
      "type": "string",
      "description": "Parameter description",
      "required": false
    }
  ],
  "related_apps": [
    {
      "package": "com.example.app",
      "name": "App Name",
      "type": "delegation|gui_automation",
      "deep_link": "scheme://path?param={param_name}",
      "priority": 100,
      "steps": ["Step 1", "Step 2"]
    }
  ]
}
```

**Skill Matching**:
1. Basic: Keyword matching against user input
2. Advanced: LLM-based semantic intent matching (when VLM client configured)
3. App availability: Only matches skills with installed apps
4. Priority: Higher priority apps preferred when multiple matches exist

## Adding New Tools

Tools should implement the `Tool` interface in `app/src/main/java/com/roubao/autopilot/tools/Tool.kt`:

```kotlin
interface Tool {
    val name: String
    val description: String
    val parameters: List<ToolParameter>
    suspend fun execute(params: Map<String, Any?>): ToolResult
}
```

Register new tools in `ToolManager.initialize()`.

## Safety Mechanisms

- **Sensitive Screen Detection**: Automatically stops when detecting payment/password pages
- **Max Step Limit**: Tasks limited to 25 agent loop iterations
- **Manual Stop**: User can halt execution via overlay controls
- **Screen Awake**: Device kept awake during execution to prevent interruption
- **Root Mode Guard**: su commands only available when Shizuku runs with root permissions

## Code Style Notes

- **Package Structure**: Feature-based organization under `com.roubao.autopilot`
- **Coroutines**: Suspend functions for async operations, use `withContext(Dispatchers.IO)` for I/O
- **Error Handling**: Graceful degradation - continue with reduced functionality if optional components fail
- **Logging**: Use println with `[ComponentName]` prefix for debug output
- **Chinese Comments**: Many comments are in Chinese (original language of development team)

## Testing Notes

This project does not currently have automated tests. When testing:

1. **Shizuku Connection**: Verify Shizuku is running before any device control operations
2. **API Configuration**: Ensure valid VLM API key is configured in settings
3. **App Permissions**: Check overlay, notification, and query permissions are granted
4. **Network**: VLM API calls require network connectivity
5. **Device Compatibility**: Test on different Android versions and custom ROMs (MIUI, ColorOS, etc.)

## Important Architectural Decisions

1. **No Python Dependency**: Complete Kotlin rewrite allows on-device execution
2. **Shizuku over Root**: Provides system access without requiring device rooting
3. **Skills before Agents**: Skills layer attempts fast path (DeepLink delegation) before expensive VLM-based GUI automation
4. **Multi-Agent Design**: Specialized agents (Manager, Executor, Reflector) mirror human task execution patterns
5. **OpenAI-Compatible API**: Vendor-agnostic VLM integration supports multiple providers

## Development Roadmap Context

**v2.0 (In Progress)** on branch `roubao2.0+AccessibilityService`:
- AccessibilityService integration for precise UI element interaction
- UI tree awareness for structured element identification
- Macro script recording and playback system
- Hybrid execution mode (accessibility + coordinates)

This context helps understand why certain features may be partially implemented or have architectural hooks for future enhancement.

## Related Resources

- Original MobileAgent Paper: Multi-agent system for mobile device control
- Shizuku Documentation: https://github.com/RikkaApps/Shizuku
- Chinese User Community: Primary user base is Chinese-speaking
