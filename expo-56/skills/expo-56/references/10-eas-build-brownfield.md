# Expo SDK 56 — EAS Build, Build Performance, Brownfield & AI Scaffolding

> Knowledge-base reference compiled from official Expo SDK 56 documentation and changelog.
> Captured: 2026-05-22. SDK 56.

---

## 1. iOS Build Performance

Source: <https://expo.dev/changelog/sdk-56>

- **Precompiled XCFrameworks** reduce the **median clean iOS build time by ~16% (~1 minute saved)**.
- Enabled **by default**, both **locally and on EAS Build**.
- **Disable** via the environment variable:
  ```sh
  EXPO_USE_PRECOMPILED_MODULES=0
  ```
- React Native core and Expo modules are shipped as prebuilt frameworks rather than being compiled from source on every clean build.
- See also "Precompiled artifacts for community libraries" under EAS Build (Section 3), which add a further ~1 minute / ~20% reduction on iOS clean builds.

---

## 2. Android Build Performance

Source: <https://expo.dev/changelog/sdk-56>

### 2.1 `android.usePrecompiledHeaders` (expo-build-properties)

- New config option in **`expo-build-properties`** that applies **CMake precompiled headers** to C++ codegen output.
- **Benchmark:** the `:app:buildCMakeDebug` task dropped from **17m 10s → 6m 06s — a 2.81x speedup**.
- **Default projects** see approximately **1.3x faster builds**.
- Configure in **app.json**:
  ```json
  {
    "plugins": [
      ["expo-build-properties", { "android": { "usePrecompiledHeaders": true } }]
    ]
  }
  ```

### 2.2 Kotlin Compiler Plugin

- Replaces runtime **reflection** with **build-time code generation**.
- Delivers roughly **40% faster cold starts** and **33% faster first render** times.
- **No app-side modifications required** — works automatically.

---

## 3. EAS Build Enhancements

Sources: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/build/introduction/>, <https://docs.expo.dev/build/setup/>

### 3.1 Per-step timing statistics

- EAS Build now surfaces **per-step timing for `xcodebuild` and Gradle tasks** in the build dashboard, making it easier to spot which native steps dominate build time.

### 3.2 Precompiled artifacts for community libraries

- Popular community libraries — notably **`react-native-reanimated`** and **`react-native-screens`** — are now shipped as **precompiled artifacts** on EAS Build.
- Cuts the **median iOS clean build time by an additional ~1 minute (~20%)**, stacking on top of the precompiled XCFrameworks gain (Section 1).

### 3.3 EAS Observe (coming soon)

- **Forthcoming** production performance-monitoring service that tracks **real-world** runtime metrics from shipped apps.
- Status at SDK 56: announced / not yet generally available.

### 3.4 EAS Build basics

- **What it is:** a hosted Expo Application Services platform that produces standalone app binaries for Expo and React Native projects, for **Android and iOS**.
- Key features: cloud builds with consistent environments, automatic (or bring-your-own) signing credential management, URL-based sharing of internal distribution builds, named build profiles in **`eas.json`**, native `expo-updates` support with per-profile channels, dependency caching, reusable dev builds via fingerprint matching, device installs via Expo Orbit.

### 3.5 Setup commands

Source: <https://docs.expo.dev/build/setup/>

```sh
npm install -g eas-cli          # Install EAS CLI
eas login                       # Authenticate (verify with: eas whoami)
eas build:configure            # Configure Android/iOS project for EAS Build
eas build --platform android    # Build for Android
eas build --platform ios        # Build for iOS
eas build --platform all        # Build for both
```

- Optional flag: `--message "description"` to attach build notes.
- **Android** requires Google Play Developer membership ($25 one-time). **iOS** requires Apple Developer Program membership ($99/yr). Credentials can be auto-managed by EAS or supplied manually.

### 3.6 `eas.json` configuration

Sources: <https://docs.expo.dev/build-reference/build-configuration/>, <https://docs.expo.dev/build/eas-json/>

Top-level fields: **`cli`**, **`build`** (and **`submit`** for EAS Submit).

**`cli` object:**
- `version` — semver range for the EAS CLI version
- `requireCommit` — boolean; require a git commit before building
- `appVersionSource` — where the app version is sourced from
- `promptToConfigurePushNotifications` — boolean

**Build profile fields (common):**
- `extends` — inherit from another profile (max nesting depth: 5)
- `distribution` — e.g. `"internal"` or store distribution
- `developmentClient` — boolean (true for dev builds)
- `node` — Node.js version string
- `env` — object of environment variables
- `resourceClass` — e.g. `"medium"` or `"large"`
- `channel` — EAS Update channel
- `android` / `ios` — platform-specific config objects

**Platform objects (`android` / `ios`):**
- `image` — base build image identifier
- `resourceClass`, `env`, `distribution`
- `simulator` — boolean (iOS)
- `buildType` — `"apk"` or `"aab"` (Android)

> "You can specify common properties both in the platform-specific configuration object or at the profile's root. The platform-specific options take precedence over globally-defined ones."

**Default generated profiles:**
```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

---

## 4. Brownfield / Native Integration

Sources: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/brownfield/overview/>, <https://docs.expo.dev/brownfield/isolated-approach/>, <https://docs.expo.dev/versions/latest/sdk/brownfield/>

### 4.1 SDK 56 brownfield improvements (changelog)

- **Multiple isolated apps in a single host:** set **`"multipleFrameworks": true`** on the **iOS plugin config** to run multiple inner `expo-brownfield` apps inside one host app **without symbol collisions**.
- **Custom Turbo Modules from the host app:** host apps register Turbo Module classes via **`ReactNativeHostManager.initialize`**, passing a **`turboModuleClasses`** dictionary.
- **iOS prebuilds by default:** `expo-brownfield` now uses **prebuilt React Native frameworks by default** on iOS; opt out with the **`buildReactNativeFromSource`** plugin option.

### 4.2 What brownfield means

- A **brownfield** app is an existing native app (built with another technology) whose **main entry point is NOT a React Native view**. Greenfield apps use Expo/React Native as the entry point from the start.
- Brownfield support for integrating Expo modules into existing native projects is in **alpha** at SDK 56.

**Supported tooling in brownfield:** Expo SDK ✓, Expo Modules API ✓, Expo Router ✓, Expo CLI ✓, EAS Build ✓, EAS Submit ✓, EAS Update ✓. **Not supported:** Expo Dev Client ✗.

### 4.3 Two integration approaches

1. **Integrated** — React Native code lives inside the existing native project (tight coupling). Setup involves Application class init, a `ReactActivity` subclass, and Gradle/AndroidManifest configuration.
2. **Isolated** — React Native developed separately and packaged as a native library: **AAR for Android, XCFramework for iOS**, then consumed like any other dependency.

### 4.4 `expo-brownfield` library

Install:
```sh
npx expo install expo-brownfield
```

**Config plugin options (app.json):**

iOS:
- `ios.targetName` — Xcode target name (default: `"<scheme>brownfield"` / `"<slug>brownfield"`)
- `ios.bundleIdentifier` — bundle identifier for the brownfield target
- `ios.buildReactNativeFromSource` — boolean, default **`false`** (i.e. uses prebuilt RN frameworks by default; set true to build from source)
- `ios.multipleFrameworks` — set **`true`** to allow multiple isolated brownfield apps in one host without symbol collisions (SDK 56)

Android:
- `android.group` — Maven group ID
- `android.libraryName` — generated Android library module name (default `"brownfield"`)
- `android.package` — Java/Kotlin package name
- `android.version` — version string (default `"1.0.0"`)
- `android.publishing` — publishing configuration array

Example:
```json
{
  "expo": {
    "plugins": [
      [
        "expo-brownfield",
        {
          "ios": {
            "targetName": "MyBrownfield",
            "bundleIdentifier": "com.example.mybrownfield"
          },
          "android": {
            "libraryName": "mybrownfield",
            "group": "com.example",
            "package": "com.example.mybrownfield",
            "version": "1.0.0"
          }
        }
      ]
    ]
  }
}
```

**CLI commands:**
- `npx expo-brownfield build:android` — builds and publishes the AAR to Maven (default local `~/.m2`)
- `build:ios` — builds the XCFramework and copies the Hermes XCFramework
- `tasks:android` — lists available publish tasks and Maven repositories
- `npx expo prebuild` — generates native projects with brownfield targets (for native debugging)

**Architecture / API surface:**
- **Android module:** `ReactNativeHostManager`, `BrownfieldActivity`, `ReactNativeFragment`, `ReactNativeViewFactory`, `BrownfieldMessaging`. `BrownfieldActivity` exposes a `showReactNativeFragment()` extension.
- **iOS framework target:** `ReactNativeHostManager`, `ReactNativeViewController`, `ReactNativeView` (SwiftUI), `BrownfieldMessaging`, `ReactNativeDelegate`.

**JS-side core API (`Brownfield`):**
- `Brownfield.sendMessage(message)`
- `Brownfield.addMessageListener(listener)`
- `Brownfield.popToNative(animated)`
- `Brownfield.setNativeBackEnabled(enabled)`
- `Brownfield.getSharedStateValue(key)` / `setSharedStateValue(key, value)`
- `Brownfield.useSharedState(key, initialValue)` (hook)

**Testing:** `npx expo start` for Metro + hot reload during development; in production the JS bundle is embedded in the AAR/XCFramework so no Metro server is needed.

---

## 5. AI-Friendly Scaffolding (SDK 56)

Sources: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/skills/>, <https://docs.expo.dev/more/create-expo/>

### 5.1 Generated agent files

- New projects from **`create-expo-app`** generate, **by default**:
  - **`AGENTS.md`**
  - **`CLAUDE.md`**
  - **`.claude/settings.json`**
- Purpose: give AI coding agents (e.g. Claude Code) Expo-specific context and the **`expo` skills plugin configured automatically**.
- **`AGENTS.md` points to the versioned Expo docs matching the project's SDK version.**
- **Opt out** with:
  ```sh
  npx create-expo-app --no-agents-md
  ```
  This "Skips generating **AGENTS.md**, **CLAUDE.md**, and **.claude/settings.json**."

### 5.2 Official Expo Skills

- **Expo Skills** are structured instruction files that teach AI agents how to build, deploy, and debug Expo and React Native apps.
- Work with **Claude Code, Cursor, Codex**, and other AI agents. Source lives in the **`expo/skills`** GitHub repo; skills are fine-tuned for Opus models.

**Install in Claude Code:**
```text
/plugin marketplace add expo/skills
/plugin install expo
```

**Install for Codex / Cursor / other agents:**
```sh
npx skills add expo/skills
```

**Sample skills available:** `building-native-ui`, `eas-update-insights`, `expo-deployment`, `expo-module`, `native-data-fetching`, `upgrading-expo`, plus others (14 total at time of capture).

### 5.3 Related AI tooling

- **Expo MCP Server** — gives AI agents direct access to Expo/EAS services (see <https://docs.expo.dev/eas/ai/mcp/>).
- **`/llms.txt`** — documentation index listing all available doc pages for LLMs (see <https://docs.expo.dev/llms/>).

---

## Source URLs

- Changelog (primary SDK 56 source): <https://expo.dev/changelog/sdk-56>
- EAS Build introduction: <https://docs.expo.dev/build/introduction/>
- EAS Build setup: <https://docs.expo.dev/build/setup/>
- Build configuration reference: <https://docs.expo.dev/build-reference/build-configuration/>
- eas.json reference: <https://docs.expo.dev/build/eas-json/>
- Adopting prebuild: <https://docs.expo.dev/guides/adopting-prebuild/>
- Brownfield overview: <https://docs.expo.dev/brownfield/overview/>
- Brownfield isolated approach: <https://docs.expo.dev/brownfield/isolated-approach/>
- Brownfield integrated approach: <https://docs.expo.dev/brownfield/integrated-approach/>
- expo-brownfield SDK reference: <https://docs.expo.dev/versions/latest/sdk/brownfield/>
- Expo Skills for AI agents: <https://docs.expo.dev/skills/>
- create-expo-app: <https://docs.expo.dev/more/create-expo/>
- Expo MCP server: <https://docs.expo.dev/eas/ai/mcp/>
