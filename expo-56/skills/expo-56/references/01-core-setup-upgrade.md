# Expo SDK 56 — Core Platform, Setup, Upgrade & New Architecture

> Knowledge-base reference compiled from official Expo documentation.
> Domain: Core platform, setup, upgrade & New Architecture.
> Captured: 2026-05-22. Authoritative version numbers below come from the SDK 56 changelog;
> where the general docs pages defaulted to SDK 55 numbers, the discrepancy is noted inline.

---

## 1. Core Platform Versions (SDK 56)

Source: https://expo.dev/changelog/sdk-56

| Component | Version (SDK 56) |
|-----------|------------------|
| React Native | **0.85** |
| React | **19.2** (versions index page lists `19.2.3` as the pinned patch) |
| React Native Web | 0.21.0 |
| React Native TV | 0.85-stable |
| Hermes | **v1 — now the default engine** |
| TypeScript | **6.0.3** |

### Hermes v1
- Hermes **v1** is the default JavaScript engine in SDK 56.
- Opt out of Hermes v1 via `expo-build-properties`:
  ```json
  {
    "plugins": [
      ["expo-build-properties", { "useHermesV1": false }]
    ]
  }
  ```

Source: https://docs.expo.dev/versions/v56.0.0/ (package/version index — note: this page renders the latest-stable numbers and does not display a full per-package list; it points to `/llms.txt` for the complete page index.)

---

## 2. Tooling & Environment Requirements (SDK 56)

Source (authoritative for SDK 56): https://expo.dev/changelog/sdk-56
Source (general env guide, defaults to current stable SDK 55): https://docs.expo.dev/get-started/set-up-your-environment/

### Hard minimums introduced/raised in SDK 56
- **Node.js**: drops support for versions before **v20.19.4** (i.e. `>= 20.19.4` required, per React Native 0.85).
- **Xcode**: **26.4 is required** (minimum).
- **iOS / tvOS**: minimum **16.4** (bumped from 15.1).
- **macOS**: minimum **13.4**.

> Discrepancy note: The general SDK index/setup pages (which default to SDK 55) showed
> Node.js 22.13.x, Xcode 26.2+, and React 19.2.3. The SDK 56 changelog values above
> (Node.js 20.19.4, Xcode 26.4, iOS/tvOS 16.4, macOS 13.4, React 19.2) are the
> SDK-56-specific figures and take precedence for this domain.

### General environment (from set-up-your-environment, applies broadly)
- **JDK**: Java **17** required on all platforms.
  - macOS: `brew install --cask zulu@17`
  - Windows: `choco install -y microsoft-openjdk17`
  - Linux: OpenJDK 17
  - `JAVA_HOME` must be configured.
- **Watchman** (macOS): `brew update && brew install watchman`
- **Android Studio** (latest) + Android SDK Platform (35 listed on the SDK-55-defaulting page; SDK 56 versions index lists `compileSdkVersion 36`, `targetSdkVersion 36`, Android 7+ runtime). `ANDROID_HOME` env var required; verify ADB with `adb --version`.
- **iOS**: Xcode with iOS Simulator; enable Developer Mode on physical devices; Apple Developer Program enrollment for device deployment.

---

## 3. New Architecture

Source: https://docs.expo.dev/guides/new-architecture/

- **Status in SDK 56**: The New Architecture is **always enabled and cannot be disabled** (this rule applies to SDK 55+, which includes SDK 56).
- **No opt-out** in SDK 55+. The `newArchEnabled: false` setting is only honored in **SDK 54 and earlier** and requires a development build.
- New projects have had it enabled by default since SDK 52.
- All `expo-*` packages support the New Architecture, including **Bridgeless mode**.
- Modules built with the Expo Modules API support it by default — no extra work.
- Expo Go only supports the New Architecture.
- ~83% of SDK 54 projects were on the New Architecture as of January 2026.

### Config key (legacy / SDK 54 and earlier only)
```json
{
  "expo": {
    "newArchEnabled": true
  }
}
```
Bare React Native apps (SDK 54 and earlier):
- Android: `newArchEnabled=true` in `gradle.properties`
- iOS: `newArchEnabled` set to `"true"` in `Podfile.properties.json`

### Validating third-party compatibility
```sh
npx expo-doctor@latest
```
Checks libraries against the React Native Directory for New Architecture support.

---

## 4. Project Creation

Source: https://docs.expo.dev/get-started/create-a-project/

### Create an SDK 56 project
```sh
npx create-expo-app@latest --template default@sdk-56
```
- Base command: `npx create-expo-app@latest`
- Template flag: `--template default@sdk-56`

> Transition-period note: During the SDK 56 rollout, running `create-expo-app@latest`
> **without** the template flag creates an **SDK 55** project. Use SDK 55 if you need to
> run via Expo Go on physical devices; otherwise use `--template default@sdk-56` for SDK 56.

### AI agent scaffolding (new in SDK 56)
Source: https://expo.dev/changelog/sdk-56

New projects include the following files generated with Expo-specific guidance for AI coding agents:
- **AGENTS.md**
- **CLAUDE.md**
- **`.claude/settings.json`**

### System requirements for project creation
- Node.js LTS
- macOS, Windows (PowerShell / WSL 2), or Linux

---

## 5. Expo CLI (install / version)

Source: https://docs.expo.dev/more/expo-cli/

- The Expo CLI ships **bundled with the `expo` package** (local CLI). There is no separate global `expo-cli` to install.
- Add it by installing the package: `yarn add expo` (or your package manager's equivalent).
- Invoke via `npx expo` (or `yarn expo`).
- Check version: `npx expo --version`
- Help: `npx expo -h`

Common commands:
- `npx expo start` — launch dev server
- `npx expo prebuild` — generate native directories
- `npx expo run:ios` / `npx expo run:android` — compile locally
- `npx expo install [package]` — install version-compatible packages

---

## 6. Upgrading to SDK 56

Source (general walkthrough, documents through SDK 55): https://docs.expo.dev/workflow/upgrading-expo-sdk-walkthrough/
Source (SDK 56 specific "Upgrading your app"): https://expo.dev/changelog/sdk-56

### Step-by-step (SDK 56)

1. **Upgrade the `expo` package and fix dependencies**
   ```sh
   npx expo install expo@^56.0.0 --fix
   ```
   (Equivalent manual install then fix per the general walkthrough:
   `npm install expo@^56.0.0` / `yarn add expo@^56.0.0` / `pnpm add expo@^56.0.0` /
   `bun install expo@^56.0.0`, followed by `npx expo install --fix`.)

2. **Run diagnostics**
   ```sh
   npx expo-doctor@latest
   ```

3. **Handle native projects**
   - With Continuous Native Generation (CNG): delete the `android` and `ios` directories — they regenerate on the next build.
   - Without CNG: run `npx pod-install` (for projects with an `ios` directory) and apply changes from the Native project upgrade helper.
   - If using `expo-dev-client`: create new development builds.

4. **Review the SDK 56 changelog** for breaking changes, deprecations, and version-specific instructions.

### Alternative: AI agent upgrade
The walkthrough notes you can use an AI coding agent with Expo Skills' **`upgrading-expo`** skill instead of the manual steps.

---

## 7. Breaking Changes & Codemods (SDK 56)

Source: https://expo.dev/changelog/sdk-56

1. **`expo/fetch` is now the default `globalThis.fetch`**
   - Automatically installed as the global fetch implementation.
   - Opt out by setting `EXPO_PUBLIC_USE_RN_FETCH=1` in `.env`.

2. **Async file operations in `expo-file-system`**
   - `copy()` and `move()` now return Promises.
   - Use `copySync()` / `moveSync()` for synchronous behavior.

3. **DOM components WebView default**
   - `@expo/dom-webview` replaces the `react-native-webview` dependency for DOM components.

4. **Vector icons removed from `expo`**
   - `@expo/vector-icons` is no longer a dependency of the `expo` package.
   - Migrate with:
     ```sh
     npx @react-native-vector-icons/codemod
     ```
   - Migrate to scoped `@react-native-vector-icons/*` packages.

5. **Expo Router forks React Navigation internals**
   - Existing `@react-navigation/*` imports may break.
   - Codemod:
     ```sh
     npx expo-codemod sdk-56-expo-router-react-navigation-replace [your-source-directory]
     ```

### Deprecations
- Original **Calendar / Contacts / MediaLibrary** APIs — superseded by redesigned stable (object-oriented, Builder-pattern) versions.
- `@expo/vector-icons` — migrate to `@react-native-vector-icons/*`.

---

## 8. Notable New Features (SDK 56)

Source: https://expo.dev/changelog/sdk-56

- **Expo UI** — production-ready universal components; drop-in replacements for community libraries.
- **Inline modules** — define Expo modules directly in project structure.
- **Type generation** (`expo-type-information`) — CLI commands: `module-interface`, `inline-modules-interface`, `short-module-interface`.
- **Faster builds** — precompiled XCFrameworks (iOS ~16% faster); Android `android.usePrecompiledHeaders` option (up to 2.81x in benchmarks):
  ```json
  { "plugins": [["expo-build-properties", { "android": { "usePrecompiledHeaders": true } }]] }
  ```
- **Expo Router** — Android toolbar (experimental), Stack v5 (Material-style headers), streaming SSR, `createStaticLoader` / `createServerLoader`, `SuspenseFallback` customization.
- **Brownfield** — `"multipleFrameworks": true` (iOS plugin) for multiple isolated apps; custom Turbo Modules; iOS prebuilt frameworks by default.
- **Status / Navigation bar** — React component APIs aligned to a shared prop surface (`style`, `hidden`).
- **iOS Widgets** — promoted to stable; full environment access without pre-rendering.
- **Convex integration** — `eas integrations:convex:connect` provisions backends.

### Other config keys captured
- Bytecode diffing opt-out: `"enableBsdiffPatchSupport": false` (in the updates block).
- On-demand filesystem disable: `"experiment.onDemandFilesystem": false` (app.json).

---

## Sources
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/
- https://docs.expo.dev/get-started/set-up-your-environment/
- https://docs.expo.dev/get-started/create-a-project/
- https://docs.expo.dev/workflow/upgrading-expo-sdk-walkthrough/
- https://docs.expo.dev/guides/new-architecture/
- https://docs.expo.dev/more/expo-cli/
