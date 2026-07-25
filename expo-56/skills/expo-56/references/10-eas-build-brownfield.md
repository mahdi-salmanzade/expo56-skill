# Expo SDK 56 — EAS Build, Build Performance, Brownfield & AI Scaffolding

> Knowledge-base reference compiled from official Expo SDK 56 documentation and changelog.
> Captured: 2026-05-22. SDK 56. Re-verified 2026-07-25 against `expo/expo` `origin/sdk-56`
> (`expo-brownfield` 56.0.25, `expo-build-properties` 56.0.24, `expo-modules-autolinking` 56.0.21)
> and `origin/sdk-57`. SDK 57 deltas are collected in a single trailing section.
>
> **Scope note:** EAS Build, `eas.json`, and the agent-scaffolding material below are **not
> SDK-versioned** — EAS CLI, `create-expo`, and the unversioned docs guides ship on their own
> cadence. Only Sections 1, 2, 4 and the EAS image aliases are genuinely SDK-scoped.

## What models get wrong

- Precompiled Expo modules are **ON by default** on iOS from SDK 56 (Android since SDK 53). You do **not** opt in with `EXPO_USE_PRECOMPILED_MODULES=1` — that was the SDK 55 local-build opt-in.
- The preferred opt-out is the `expo-build-properties` `ios.usePrecompiledModules: false` plugin option, **not** the env var. `ios.usePrecompiledModules` defaults to **`true`** in SDK 56 source today (`origin/sdk-56:packages/expo-build-properties/src/pluginConfig.ts:448`), but only since `expo-build-properties` **56.0.14** (#46159, 2026-05-23). The frozen v56 docs page renders `@default false` because it was cut at 56.0.0, where that was correct — on 56.0.0–56.0.13 merely listing the plugin silently disabled precompiled modules. The SDK 56 pin `~56.0.17` resolves above the fix.
- `react-native-reanimated` and `react-native-worklets` must be opted out of precompilation **together** (SDK 56). Doing only one produces an unresolvable mixed precompiled/source linkage.
- `expo-brownfield` has no `Brownfield` **value** export — it exports bare named functions. The official docs use the namespace-import form (`import * as Brownfield from 'expo-brownfield'` → `Brownfield.sendMessage(...)`), which is valid; direct named imports are equivalent. Do not reject either form.
- `ReactNativeHostManager.initialize(turboModuleClasses:)` is **iOS-only**; the Android overload takes `(application, additionalPackages)`.
- Expo Dev Client is the one Expo tool that does **not** work in brownfield.

---

## 1. iOS Build Performance — precompiled Expo modules

Sources: <https://expo.dev/changelog/sdk-56>, `docs/pages/guides/prebuilt-expo-modules.mdx`

- **Precompiled XCFrameworks** reduce the **median clean iOS build time by ~16% (~1 minute saved)**. *(Blog benchmark; not reproducible from the monorepo — treat as marketing, not an engineering guarantee.)*
- React Native core and Expo modules ship as prebuilt binaries rather than being compiled from source on every clean build. Android ships **.aar** files linked through Gradle; iOS ships **XCFrameworks** linked through CocoaPods. Packages that aren't precompiled fall back to building from source automatically — precompiled and source-built modules coexist in one project.
- **Enabled by default:** Android since **SDK 53**; iOS since **SDK 56** (in SDK 55, iOS was default only on EAS Build, with `EXPO_USE_PRECOMPILED_MODULES=1` as the local opt-in). Applies both locally and on EAS Build.

### 1.1 Turning precompilation off

**Preferred — `expo-build-properties` (applies to local *and* EAS Build; takes effect on the next `npx expo prebuild`):**
```json app.json
{
  "expo": {
    "plugins": [
      ["expo-build-properties", { "ios": { "usePrecompiledModules": false } }]
    ]
  }
}
```
`ios.usePrecompiledModules` defaults to **`true`** in SDK 56 and 57 today (`packages/expo-build-properties/src/pluginConfig.ts`). The frozen v56 docs data reflects 56.0.0, where the default really was `false`. #46159 changed it to `true` in `expo-build-properties` **56.0.14 (2026-05-23)** so the plugin matches the Podfile default. On 56.0.0–56.0.13 adding the plugin silently disabled precompiled modules (it wrote `"EXPO_USE_PRECOMPILED_MODULES": "false"` into `Podfile.properties.json`); the SDK 56 pin `~56.0.17` resolves above that, so today the effective default is `true`.

**Env var** (read during `pod install`):
```sh
export EXPO_USE_PRECOMPILED_MODULES=0     # local: before pod install / npx expo run:ios
eas env:set --name EXPO_USE_PRECOMPILED_MODULES --value 0 --visibility plaintext   # EAS
```

**Per-package opt-out** via Expo Autolinking in **package.json** (`".*"` = everything; same key for `android` and `ios`):
```json package.json
{
  "expo": {
    "autolinking": {
      "ios": { "buildFromSource": ["react-native-reanimated", "react-native-worklets"] }
    }
  }
}
```

### 1.2 Known failure modes (SDK 56)

- **`react-native-reanimated` + `react-native-worklets` are tightly coupled.** Source-build **both** or neither. Source-building only one yields a mixed precompiled/source linkage that fails to resolve the matching framework at runtime.
- **`worklets.staticFeatureFlags` / `reanimated.staticFeatureFlags` overrides are ignored by precompiled binaries** — flag values are baked in at build time. To apply them, disable precompiled modules entirely.
- Symptom: `Unable to recognize flag: <NAME>` at runtime **on EAS Build but not locally** — the precompiled artifact's flag list doesn't match your pinned package version. Local `pod install` doesn't fetch third-party precompiled XCFrameworks, so mismatches only surface on EAS.

See also "Precompiled artifacts for community libraries" under EAS Build (Section 3).

---

## 2. Android Build Performance

Source: <https://expo.dev/changelog/sdk-56>

### 2.1 `android.usePrecompiledHeaders` (expo-build-properties)

- Config option in **`expo-build-properties`** that applies **CMake precompiled headers (PCH)** to all autolinked native libraries by generating a custom `CMakeLists.txt`.
- `@default false`, `@experimental` — "might not work with all native libraries."
- Can also be enabled with the env var `EXPO_USE_ANDROID_PRECOMPILED_HEADERS=1`.
- **Benchmark:** the `:app:buildCMakeDebug` task dropped from **17m 10s → 6m 06s — a 2.81x speedup**; default projects ~**1.3x** faster. *(Blog benchmark, not reproducible from the monorepo.)*
- Configure in **app.json**:
  ```json
  {
    "plugins": [
      ["expo-build-properties", { "android": { "usePrecompiledHeaders": true } }]
    ]
  }
  ```
- **Docs caveat:** the option is real in SDK 56 (`origin/sdk-56:packages/expo-build-properties/src/pluginConfig.ts:246`) but is **absent from `docs/public/static/data/v56.0.0/expo-build-properties.json`**, so it does not render on the SDK 56 build-properties docs page. It is present in the v57 and unversioned data. Source of truth is `pluginConfig.ts`.

### 2.2 Kotlin Compiler Plugin

> **Unverified.** The following is reported only by the SDK 56 changelog blog post and could not be
> confirmed anywhere in the `expo/expo` monorepo: there is no `kotlinCompilerPluginClasspath`,
> `annotationProcessor`, or Expo-authored compiler-plugin wiring in `expo-modules-core`'s Android
> build (the only KSP reference in `ExpoModulesCorePlugin.gradle` is a version map + a JVM-toolchain
> helper). Do not present the numbers below as engineering facts.

- Reportedly replaces runtime **reflection** with **build-time code generation**.
- Claimed roughly **40% faster cold starts** and **33% faster first render**.
- **No app-side modifications required** — works automatically.

---

## 3. EAS Build Enhancements

Sources: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/build/introduction/>, <https://docs.expo.dev/build/setup/>

### 3.1 Per-step timing statistics

- EAS Build now surfaces **per-step timing for `xcodebuild` and Gradle tasks** in the build dashboard, making it easier to spot which native steps dominate build time.

### 3.2 Precompiled artifacts for community libraries

- **Seven** community libraries ship precompiled iOS configs, identically in SDK 56 and SDK 57 (`packages/expo-modules-autolinking/external-configs/ios/*/spm.config.json`):
  `@react-native-async-storage/async-storage`, `@shopify/react-native-skia`, `react-native-reanimated`, `react-native-safe-area-context`, `react-native-screens`, `react-native-svg`, `react-native-worklets`.
- These are downloaded as precompiled XCFrameworks **on EAS Build only** — local `pod install` does not fetch them and builds them from source instead. That asymmetry is why feature-flag mismatches surface only on EAS (see §1.2).
- Cuts the **median iOS clean build time by an additional ~1 minute (~20%)**, stacking on top of the precompiled XCFrameworks gain (Section 1). *(Blog benchmark, not reproducible from the monorepo.)*

### 3.3 EAS Observe

- Production performance-monitoring service tracking **real-world** runtime metrics from shipped apps (cold/warm launch, TTR, TTI, navigation timings, update download).
- **Status: Open Beta.** First 10,000 monthly active users are free (`docs/pages/eas/observe/introduction.mdx`). It has shipped — the "coming soon" framing from the SDK 56 changelog is stale.
- Library pin: `expo-observe ~56.0.26` (SDK 56) / `~57.0.8` (SDK 57), per `packages/expo/bundledNativeModules.json` on the release branches — that file ships inside `expo` and is what `expo install` resolves against.
- Docs: `/eas/observe/{introduction,get-started,configuration,dashboard,events,eas-update,reference,integrations}`. There is also an official `eas-observe` skill.

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

Sources: <https://docs.expo.dev/eas/json/> (full property reference), <https://docs.expo.dev/build/eas-json/> (narrative guide). Not SDK-versioned — only an `unversioned` schema exists (`docs/public/static/schemas/unversioned/eas-json-build-*.js`).

Top-level fields: **`cli`**, **`build`** (and **`submit`** for EAS Submit).

**`cli` object:**
- `version` — required semver range for the EAS CLI version
- `requireCommit` — boolean; require all changes committed before a build. Defaults to `false`
- `appVersionSource` — `local` (default) or `remote`; `remote` makes EAS-server values win
- `promptToConfigurePushNotifications` — boolean; `false` skips push credentials setup. Defaults to `true`

**Build profile — common properties** (settable at the profile root *or* inside `android`/`ios`; platform-specific wins):
`withoutCredentials`, `extends`, `credentialsSource`, `releaseChannel`, `channel`, `distribution`, `developmentClient`, `resourceClass`, `prebuildCommand`, `buildArtifactPaths`, `node`, `corepack`, `yarn`, `pnpm`, `bun`, `expoCli`, `env`, `autoIncrement`, `cache` (`{ disabled, key, paths }`), `config`, `environment`.

- `extends` — inherit from another profile; chains up to **depth 5**, no circular references. Cannot be specified per platform.
- `resourceClass` — `'default' | 'medium' | 'large'`, **defaults to `medium`** on both platforms. `large` is **not available on the free plan**.

**Android-only:** `withoutCredentials`, `image`, `resourceClass`, `ndk`, `autoIncrement`, `buildType` (`"apk"` / `"aab"`), `gradleCommand`, `applicationArchivePath`, `config`.

**iOS-only:** `withoutCredentials`, `simulator`, `enterpriseProvisioning`, `autoIncrement`, `image`, `resourceClass`, `bundler`, `fastlane`, `cocoapods`, `scheme`, `buildConfiguration`, `applicationArchivePath`, `config`.

**Build image aliases** (`android.image` / `ios.image`): `auto` (default when `image` is omitted — chosen from SDK/RN version), `latest`, or a per-SDK alias such as `sdk-56` / `sdk-57`. SDK aliases update with every SDK release; `latest` updates with every image release.

| SDK | Android image | macOS image |
| --- | --- | --- |
| 56 | `ubuntu-26.04-jdk-17-ndk-r27b` (`sdk-56`) — Node.js 22.22.2, pnpm 10.33.3, NDK 27.1.12297006, Java 17, Maestro 2.5.1 | `macos-tahoe-26.4-xcode-26.4` (`sdk-56`) — macOS Tahoe 26.4.1, Xcode 26.4 (17E202), Node.js 22.22.2, fastlane 2.233.1, CocoaPods 1.16.2, Ruby 3.2 |

Source: `docs/pages/build-reference/infrastructure.mdx`.

> **Gradle heap trap:** EAS Build sets `org.gradle.jvmargs` via the `GRADLE_OPTS` env var on the worker, which **overrides any `org.gradle.jvmargs` in your project's `gradle.properties`**. `-Xmx` is `4g` on `medium`, `8g` on `large`. Override by setting `GRADLE_OPTS` under a build profile's `env` in **eas.json**, in a workflow file, or as an EAS environment variable.

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

Sources: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/brownfield/overview/>, <https://docs.expo.dev/brownfield/isolated-approach/>, <https://docs.expo.dev/versions/v56.0.0/sdk/brownfield/>

**Version pins:** `expo-brownfield ~56.0.25` (SDK 56) / `~57.0.7` (SDK 57), per `packages/expo/bundledNativeModules.json` on `origin/sdk-56` / `origin/sdk-57`. Do **not** read pins from `docs/public/static/schemas/v5{6,7}.0.0/native-modules.json` — those are frozen at the SDK cut and understate every package.

### 4.1 SDK 56 brownfield improvements (changelog)

- **Multiple isolated apps in a single host:** set **`"multipleFrameworks": true`** on the **iOS plugin config** to run multiple inner `expo-brownfield` apps inside one host app **without symbol collisions**.
- **Custom Turbo Modules from the host app (iOS only):**
  ```swift
  // iOS — plugin/templates/ios/ReactNativeHostManager.swift:30
  @objc public func initialize(turboModuleClasses: [String: AnyClass] = [:])
  ```
  The Android counterpart has **no** `turboModuleClasses` parameter:
  ```kotlin
  // Android — plugin/templates/android/ReactNativeHostManager.kt:27
  fun initialize(application: Application, additionalPackages: List<ReactPackage> = emptyList())
  ```
  On iOS, `ReactNativeHostManager` is also exposed to Objective-C since **56.0.15** (#46227).
- **iOS prebuilds by default:** `expo-brownfield` uses **prebuilt React Native frameworks by default** on iOS; opt out with the **`buildReactNativeFromSource`** plugin option.
- **Host iOS deployment target: `platform :ios, '16.4'`.** SDK 56 already required 16.4 at the package level (`ExpoModulesCore.podspec`), but the brownfield docs shipped `15.1` in the hand-written host Podfile snippet until the SDK 57 docs pass (commit `e40945de1be`). Use **16.4** for SDK 56 as well.

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
- `ios.targetName` — Xcode target name (default: `"<scheme>brownfield"` → `"<ios.scheme>brownfield"` → `"<slug>brownfield"`; non-alphanumerics stripped)
- `ios.bundleIdentifier` — bundle identifier for the brownfield target (default: `ios.bundleIdentifier` with its last component replaced by the target name, else `com.example.<targetName>`). Validated against Apple's identifier grammar
- `ios.buildReactNativeFromSource` — boolean, default **`false`** (i.e. uses prebuilt RN frameworks by default; set true to build from source)
- `ios.multipleFrameworks` — boolean, default **`false`**; set **`true`** to allow multiple isolated brownfield apps in one host without symbol collisions (SDK 56)
- `ios.hostProvidedFrameworks` — `string[]`, default `[]`. Framework names the host iOS app already provides; they are stripped from the produced artifact. Validated as an array of non-empty strings, de-duplicated. **Added in 56.0.18** (2026-06-05, #46355) and present in SDK 57. CLI equivalent: `build:ios --host-provided SDWebImage SDWebImageWebPCoder`

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

**CLI commands** (`expo-brownfield`, bin `./bin/cli.js`; `cli/src/index.ts`):

| Command | Flags |
| --- | --- |
| `build:android` — build & publish the AAR to Maven (default local `~/.m2`) | `-d, --debug` · `-r, --release` · `-a, --all` · `--fused` · `--verbose` · `-l, --library <library>` · `-t, --task <task...>` · `--repo, --repository <repository...>` · `--dry-run` |
| `build:ios` — build the XCFramework(s) and copy the Hermes XCFramework | `-d, --debug` · `-r, --release` · `--verbose` · `-s, --scheme <scheme>` · `-x, --xcworkspace <xcworkspace>` · `-a, --artifacts <artifacts>` · `--dry-run` · `-p, --package [package]` · `--host-provided <frameworks...>` |
| `tasks:android` — list publishing tasks and Maven repositories | `--verbose` · `-l, --library <library>` · `--dry-run` |

- `-p, --package [package]` bundles the iOS output as a **self-contained Swift Package** instead of loose **.xcframework** directories, addable to the host app as a local Xcode dependency. Name defaults to `<scheme>Artifacts` — derived from the resolved Xcode **scheme**, not the target name — plus a `-debug`/`-release` suffix when precompiled modules are bundled (the SDK 56 default), e.g. `MyAppbrownfieldArtifacts-release` (`cli/src/utils/config.ts:53-59`). Shipped in `expo-brownfield` **55.0.12** (2026-02-26, #43369), i.e. SDK 55; present unchanged in SDK 56 and 57.
- `--fused` publishes a **single fat AAR per variant via AGP Fused Library**. Available in **both** SDKs — `expo-brownfield` 56.0.25 (2026-07-23) and 57.0.7 (2026-07-22), both #47921. The plugin writes `android.experimental.fusedLibrarySupport` and `android.experimental.fusedLibrarySupport.publicationOnly=false` into **gradle.properties**.
- There is a hidden internal `mangle` command invoked by `scripts/ios/mangle.rb` during `pod install` — not user-facing.
- `npx expo prebuild` — generates native projects with brownfield targets (for native debugging). **SDK 57 changes this command's default; see the delta section.**

**Architecture / native API surface:**
- **Android** (`expo.modules.brownfield` + generated templates): `object BrownfieldMessaging` (`addListener`/`removeListener`/`sendMessage`), `object BrownfieldLifecycleDispatcher` (`onApplicationCreate`, `onConfigurationChanged`), `class ReactNativeHostManager`, `open class BrownfieldActivity : AppCompatActivity(), DefaultHardwareBackBtnHandler` with `open fun showReactNativeFragment(rootComponent: String = "main", additionalPackages: List<ReactPackage> = emptyList())`, `class ReactNativeFragment : Fragment()`, `object ReactNativeViewFactory`, `class SharedState(val key: String) : SharedObject()`.
- **iOS** (framework target): `public struct BrownfieldMessaging`, `public struct BrownfieldState` (`get`/`set`/`subscribe`/`delete`), `public class ReactNativeViewController: UIViewController`, `public struct ReactNativeView: View` (SwiftUI), `ReactNativeDelegate`, `@objc public class ReactNativeHostManager`.

**JS-side API — bare named exports from `expo-brownfield`.** There is no `Brownfield` *value* export, but the namespace-import form the docs use (`import * as Brownfield from 'expo-brownfield'`, then `Brownfield.sendMessage(...)`) is valid and equivalent to importing the names directly:
```ts
import {
  popToNative, setNativeBackEnabled,
  addMessageListener, sendMessage, removeMessageListener,
  removeAllMessageListeners, getMessageListenerCount,
  getSharedStateValue, setSharedStateValue, deleteSharedState,
  addSharedStateListener, useSharedState,
} from 'expo-brownfield';
```

| Function | Signature |
| --- | --- |
| `popToNative` | `(animated: boolean = false) => void` — `animated` is iOS-only |
| `setNativeBackEnabled` | `(enabled: boolean) => void` |
| `addMessageListener` | `(listener: Listener<MessageEvent>) => EventSubscription` |
| `sendMessage` | `(message: Record<string, any>) => void` |
| `removeMessageListener` | `(listener: Listener<MessageEvent>) => void` |
| `removeAllMessageListeners` | `() => void` |
| `getMessageListenerCount` | `() => number` |
| `getSharedStateValue` | `<T = any>(key: string) => T \| undefined` |
| `setSharedStateValue` | `<T = any>(key: string, value: T) => void` |
| `deleteSharedState` | `(key: string) => void` |
| `addSharedStateListener` | `<T = any>(key: string, callback: (event: SharedStateChangeEvent<T> \| undefined) => void) => EventSubscription` |
| `useSharedState` | `<T = any>(key: string, initialValue?: T) => [T \| undefined, (value: T \| ((prev: T \| undefined) => T)) => void]` |

> **Trap — the published SDK 56 docs are wrong here.** `addSharedStateListener`'s callback receives an **event object** of type `SharedStateChangeEvent<T> = { value: T | undefined }`, not the raw value — read `event?.value`. This was fixed in **`expo-brownfield` 56.0.11** (#44401), but `docs/public/static/data/v56.0.0/expo-brownfield.json` was cut before the fix and still renders `(value: T | undefined) => void`. The `origin/sdk-56` source is correct.

**Testing:** `npx expo start` for Metro + hot reload during development; in production the JS bundle is embedded in the AAR/XCFramework so no Metro server is needed.

### 4.5 Lifecycle listeners (both approaches)

Missing lifecycle forwarding is the most common cause of "deep links / push notifications never fire in my host app." Source: `docs/pages/brownfield/lifecycle-listeners.mdx`.

- **Android:** forward `onCreate()` and `onConfigurationChanged()` from your `Application` class to **`ApplicationLifecycleDispatcher`** (`BrownfieldLifecycleDispatcher` wraps this for brownfield targets). `ReactActivityHandler` forwards `Activity` events. Modules register `ReactActivityLifecycleListener` / `ApplicationLifecycleListener` implementations through a `Package` class.
- **iOS:** forward the relevant `AppDelegate` calls to **`ExpoAppDelegateSubscriberManager`**, or — if your `AppDelegate` doesn't already extend another class — inherit from **`ExpoAppDelegate`**, which forwards automatically. Modules register an `ExpoAppDelegateSubscriber`. Not every `UIApplicationDelegate` method is forwarded; check `ExpoAppDelegate.swift` for the list.
- **Verify:** `npx expo install expo-linking`, add a `Linking` listener, and open a deep link.

---

## 5. AI-Friendly Scaffolding

> **Not SDK-versioned.** `create-expo` and the skills/MCP docs ship independently of the SDK; this
> section decayed entirely between 2026-05-22 and 2026-07-25. Re-verify against current docs.

Sources: <https://docs.expo.dev/skills/>, <https://docs.expo.dev/more/create-expo/>, `packages/create-expo/src/generateAgentFiles.ts`

### 5.1 Generated agent files

New projects from **`create-expo-app`** generate:

| File | When |
| --- | --- |
| `AGENTS.md` | **Always** |
| `CLAUDE.md` | **Always** |
| `.claude/settings.json` | **Always** |

- A change to generate `CLAUDE.md` / `.claude/settings.json` only when Claude Code is detected (`~/.claude.json` or `~/.claude`) has landed on `main` (#46666, 2026-06-10) but is **not yet published** — the shipping `create-expo` 5.0.0 still generates all three unconditionally, matching the docs page (`more/create-expo.mdx`).
- `CLAUDE.md` is a **one-line file** whose entire content is `@AGENTS.md`.
- `.claude/settings.json` is exactly:
  ```json
  { "enabledPlugins": { "expo@claude-plugins-official": true } }
  ```
- Existing files are never overwritten (each target is skipped when `fs.existsSync(filePath)`).
- In the shipping `create-expo` 5.0.0, `AGENTS.md` and `CLAUDE.md` are generated from **inline constants** in `generateAgentFiles.ts` (`getAgentsMdContent()` / `CLAUDE_MD_CONTENT`); `AGENTS.md` is a two-line pointer to `https://docs.expo.dev/versions/v<sdk>.0.0/`. Sourcing them from `expo/llm-configs` via `scripts/sync-agent-templates.js` (#46968) is on `main` but **not yet released**.
- **`AGENTS.md` points to the versioned Expo docs matching the project's SDK version.**
- **Opt out** with `npx create-expo-app --no-agents-md` — "Skips generating **AGENTS.md**, **CLAUDE.md**, and **.claude/settings.json**."

### 5.2 Official Expo Skills

- **Expo Skills** are structured instruction files that teach AI agents how to build, deploy, and debug Expo and React Native apps. Source lives in the **`expo/skills`** GitHub repo.

| Agent | Install |
| --- | --- |
| Claude Code | `/plugin install expo@claude-plugins-official` |
| Codex | `codex plugin add expo@openai-curated` (or `/plugins` → `openai-curated` → `expo`) |
| Cursor | Auto-imports skills already installed for Claude Code/Codex — **Settings › Rules, Skills, Subagents › "Include third-party Plugins, Skills, and other configs"** (on by default). Otherwise `npx skills add expo/skills` |
| Any other agent | `npx skills add expo/skills` |

> Skills in Cursor do **not** appear in the slash (`/`) menu — they work by auto-discovery only.

**21 skills** ship in the `expo` plugin, named `expo-*` (framework) or `eas-*` (EAS services):

- **EAS (6):** `eas-app-stores`, `eas-hosting`, `eas-observe`, `eas-simulator`, `eas-update-insights`, `eas-workflows`
- **Framework (15):** `expo-app-clip`, `expo-brownfield`, `expo-data-fetching`, `expo-dev-client`, `expo-dom`, `expo-examples`, `expo-module`, `expo-native-ui`, `expo-project-structure`, `expo-router`, `expo-skill-feedback`, `expo-tailwind-setup`, `expo-ui`, `expo-upgrade`, `expo-web-to-native`

The older names quoted at capture time (`building-native-ui`, `expo-deployment`, `native-data-fetching`, `upgrading-expo`) were renamed. Relevant to this file: an official **`expo-brownfield`** skill now exists. Source: `docs/ui/components/ExpoSkillsTable/data/expo-skills.json` (fetched 2026-07-23).

### 5.3 Related AI tooling

- **Expo MCP Server** — gives AI agents direct access to Expo/EAS services. Now at <https://docs.expo.dev/mcp/>; the old `/eas/ai/mcp/` URL 301-redirects there (`docs/public/_redirects:628-629`), so links in older material still work.
- **AI-agent quick-start guides** at `/agents/`: `claude`, `codex`, `cursor`, `argent`, `agent-device`.
- **`/llms.txt`** — documentation index listing all available doc pages for LLMs (see <https://docs.expo.dev/llms/>).

---

## Source URLs

> Do **not** use `/versions/latest/` URLs — `latest` now resolves to SDK 57 and will silently
> mislead SDK 56 readers. Always pin `/versions/v56.0.0/` or `/versions/v57.0.0/`.

- Changelog (primary SDK 56 source): <https://expo.dev/changelog/sdk-56>
- Precompiled Expo Modules: <https://docs.expo.dev/guides/prebuilt-expo-modules/> (`docs/pages/guides/prebuilt-expo-modules.mdx`)
- EAS Build introduction: <https://docs.expo.dev/build/introduction/>
- EAS Build setup: <https://docs.expo.dev/build/setup/>
- Build server infrastructure (image tables, Gradle JVM args): <https://docs.expo.dev/build-reference/infrastructure/>
- **eas.json full property reference:** <https://docs.expo.dev/eas/json/> — schemas at `docs/public/static/schemas/unversioned/eas-json-build-{common,android,ios}-schema.js`
- eas.json narrative guide: <https://docs.expo.dev/build/eas-json/>
- Adopting prebuild: <https://docs.expo.dev/guides/adopting-prebuild/>
- Brownfield overview: <https://docs.expo.dev/brownfield/overview/>
- Brownfield isolated approach: <https://docs.expo.dev/brownfield/isolated-approach/>
- Brownfield integrated approach: <https://docs.expo.dev/brownfield/integrated-approach/>
- Brownfield lifecycle listeners: <https://docs.expo.dev/brownfield/lifecycle-listeners/>
- expo-brownfield SDK reference: <https://docs.expo.dev/versions/v56.0.0/sdk/brownfield/> · SDK 57: <https://docs.expo.dev/versions/v57.0.0/sdk/brownfield/>
- EAS Observe: <https://docs.expo.dev/eas/observe/introduction/>
- Expo Skills for AI agents: <https://docs.expo.dev/skills/>
- create-expo-app: <https://docs.expo.dev/more/create-expo/>
- Expo MCP server: <https://docs.expo.dev/mcp/> · AI agent guides: <https://docs.expo.dev/agents/>

**Monorepo anchors (most reliable):** `packages/expo-brownfield/{src,cli/src/index.ts,plugin/src,plugin/templates}`, `packages/expo-build-properties/src/pluginConfig.ts`, `packages/expo-modules-autolinking/{CHANGELOG.md,external-configs/ios}`, `packages/create-expo/src/generateAgentFiles.ts`.

---

## SDK 57 delta

Brownfield is essentially unchanged; the real SDK 57 movement is in `npx expo prebuild`, the precompiled-modules pipeline, and the EAS build images.

### Breaking

- **`npx expo prebuild` now CLEANS by default.** It clears and regenerates `android/` and `ios/` on every run. Pass **`--no-clean`** to apply changes to the existing folders (the SDK 56 behaviour). This matters for brownfield: any hand-edits to the generated native projects are wiped unless you pass `--no-clean`. Source: `origin/sdk-57:packages/@expo/cli/CHANGELOG.md` → `## 57.0.0` breaking change (#47209). Not on `origin/sdk-56`.

### New in 57

- **`android.cmakeVersion?: string`** (`expo-build-properties`) — "Override the CMake version, applied to the app and all autolinked native modules." Shipped in `expo-build-properties` 57.0.2 / `expo-modules-autolinking` 57.0.2 (#47377). **Not backported** — absent from `origin/sdk-56:packages/expo-build-properties/src/pluginConfig.ts`. Caveat: also absent from the frozen `docs/public/static/data/v57.0.0/expo-build-properties.json` (cut at 57.0.0, before 57.0.2); it only renders on the unversioned docs page.
- **`buildFromSource` now propagates across the precompiled dependency graph** (iOS). Source-building `react-native-worklets` automatically forces its dependents (`react-native-reanimated`) to build from source, instead of linking a prebuilt XCFramework against a source-built dependency. `expo-modules-autolinking` 57.0.9 (#48041). This is the mechanical fix for the manual "opt out of both together" workaround in §1.2 — **that workaround is still required on SDK 56** (top of `origin/sdk-56` is 56.0.21, no such entry).
- **Missing `ENABLE_CROSS_RUNTIME_STACK_TRACES` flag added to the `react-native-worklets` precompile config**, so its prebuilt XCFramework matches the package's `staticFlags.json`. `expo-modules-autolinking` 57.0.4 (#47478). Directly addresses the `Unable to recognize flag: <NAME>` failure mode in §1.2. Not on `origin/sdk-56`.
- **Resource-bundle targets are raised to their pod's effective deployment target**, fixing Xcode 27 build failures for pods whose podspec declares a deployment target below iOS 15.0 (e.g. `ReachabilitySwift`). `expo-modules-autolinking` 57.0.5 (#47562). Not on `origin/sdk-56`.
- **`react-native-reanimated` precompile config realigned** with `reanimated@4.5.0`'s generated component/native-view sources (`expo-modules-autolinking` 57.0.0, #47201). SDK 56 stays aligned to `reanimated@4.3.1` / `worklets@0.8.3` (56.0.13, #46221).
- **EAS Build images** — `docs/pages/build-reference/infrastructure.mdx`:

  | | SDK 56 | SDK 57 |
  | --- | --- | --- |
  | Android image | `ubuntu-26.04-jdk-17-ndk-r27b` (`sdk-56`) | `ubuntu-26.04-jdk-17-ndk-r27b-sdk-57` (`latest`, `sdk-57`) |
  | GCE image | `ubuntu-2604-resolute-amd64-v20260505` | `ubuntu-2604-resolute-amd64-v20260624` |
  | Node.js | 22.22.2 | 22.23.1 |
  | pnpm / npm | 10.33.3 / 10.9.4 | 11.9.0 / 10.9.8 |
  | Bun / node-gyp / Maestro | 1.3.13 / 12.3.0 / 2.5.1 | 1.3.14 / 13.0.0 / 2.6.1 |
  | NDK / Java | 27.1.12297006 / 17 | 27.1.12297006 / 17 (unchanged) |
  | macOS image | `macos-tahoe-26.4-xcode-26.4` (`sdk-56`) | `macos-tahoe-26.5-xcode-26.6` (`latest`, `sdk-57`) |
  | macOS / Xcode | Tahoe 26.4.1 / 26.4 (17E202) | Tahoe 26.5.2 / 26.6 (17F113) |
  | fastlane / CocoaPods / Ruby | 2.233.1 / 1.16.2 / 3.2 | 2.236.1 / 1.16.2 / 3.2 |

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 |
| --- | --- | --- |
| `expo-brownfield` | `~56.0.25` | `~57.0.7` |
| `expo-build-properties` | `~56.0.24` | `~57.0.7` |
| `expo-observe` | `~56.0.26` | `~57.0.8` |

Source: `git show origin/sdk-56:packages/expo/bundledNativeModules.json` / `origin/sdk-57:...`. First-party `expo-*` pins are **not** flat `~57.0.0` — each package has its own patch level. Ignore `docs/public/static/schemas/{v56.0.0,v57.0.0}/native-modules.json`; it is frozen at the SDK cut and understates these.

Of the seven precompiled community libraries in §3.2, only two moved in 56 → 57: `react-native-reanimated` `4.3.1` → `4.5.0` and `react-native-worklets` `0.8.3` → `0.10.0` — which is exactly why the reanimated precompile config was realigned (#47201) and the missing worklets flag added (#47478). The other five are **identical on both branches**: `react-native-screens` **`~4.26.0`** (the frozen schema's `4.25.2` is stale — do not chase a screens precompiled-artifact mismatch that does not exist), `react-native-safe-area-context ~5.7.0`, `react-native-svg 15.15.4`, `@shopify/react-native-skia 2.6.2`, `@react-native-async-storage/async-storage 2.2.0`.

### Verified unchanged (do not go hunting)

- **`expo-brownfield` 57.0.0 introduced zero user-facing changes** (`origin/sdk-57:packages/expo-brownfield/CHANGELOG.md` → "_This version does not introduce any user-facing changes._"). The CLI (`cli/src/index.ts`) is **byte-identical** between `origin/sdk-56` and `origin/sdk-57`. `docs/pages/versions/v56.0.0/sdk/brownfield.mdx` vs `v57.0.0` differs **only** in `sourceCodeUrl`.
- `ios.hostProvidedFrameworks` and `build:android --fused` are **not** SDK-57-only — both were backported. Do not upgrade for them; just bump the patch: `hostProvidedFrameworks` needs `expo-brownfield >= 56.0.18` (#46355), `--fused` needs `>= 56.0.25` (#47921, the same PR that shipped as 57.0.7). The SDK 56 pin `~56.0.25` already covers both.
- `ios.usePrecompiledModules: false` being honoured when `EXPO_USE_PRECOMPILED_MODULES` is already set in the environment (e.g. on EAS Build) is likewise a backport, not a 57 feature — needs `expo-modules-autolinking >= 56.0.17` (#46983, shipped as 57.0.3 on the other line).
- **No Node `engines` bump in SDK 57.** `origin/sdk-57:packages/expo/package.json` has **no `engines` field** at all; Node >= 20.19.4 still works. The 22.22.2 → 22.23.1 move in the EAS image table above is just the build worker's Node, not a requirement on your machine. (The `^22.13.0 || ^24.3.0` engines range is SDK 58 work on `main`.)
- The precompiled community-library set is the **same seven packages** in both SDKs.
- `ios.usePrecompiledModules` defaults to **`true`** in both SDKs; the Podfile logic that makes precompiled modules unconditionally default is identical on both branches (`diff origin/sdk-56 origin/sdk-57 -- templates/expo-template-bare-minimum/ios/Podfile` is empty).
- **No `proguard-android-optimize.txt` in SDK 57.** `origin/sdk-57:templates/expo-template-bare-minimum/android/app/build.gradle` still uses `getDefaultProguardFile("proguard-android.txt")`, identical to `sdk-56`. R8 full-mode optimization is **not** on by default in 57.
- **No iOS scene-based lifecycle on the SDK 57 release branch.** `origin/sdk-57` has no `SceneDelegate.swift` in `templates/expo-template-bare-minimum/ios/HelloWorld/` (only an unrelated `expo-updates` e2e fixture matches repo-wide), the template `Info.plist` has no `UIApplicationSceneManifest`, and `AppDelegate.swift` is **byte-identical** between `origin/sdk-56` and `origin/sdk-57`. `UIWindowSceneDelegate` appears only on `main` (`packages/expo/ios/AppDelegates/ExpoAppSceneDelegate.swift`), i.e. the SDK 58 track. Brownfield integrated-approach guidance about patching `AppDelegate` is therefore valid unchanged for both 56 and 57. *(If you have a source saying scenes shipped in 57, re-check it against `origin/sdk-57` before acting.)*
- Host Podfile `platform :ios, '16.4'` — the SDK 57 docs bump (#47405) was a catch-up; `ExpoModulesCore.podspec` already declares `ios => '16.4'` on **both** branches.
- EAS Build, `eas.json`, `create-expo` agent scaffolding and Expo Skills are **not SDK-versioned** — nothing in §3.4–3.6 or §5 changes because of SDK 57.

Packages checked for this file: `expo-brownfield`, `expo-build-properties`, `expo-modules-autolinking`, `@expo/cli`, `expo-observe`, `create-expo`.
