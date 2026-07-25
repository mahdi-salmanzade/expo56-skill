# Expo SDK 56 — App Configuration, Icons/Splash & Foundational Packages

Knowledge base reference compiled from official Expo SDK 56 documentation and the expo/expo source tree. Captured 2026-05-22, revised 2026-07-25 against the `origin/sdk-56` / `origin/sdk-57` release branches and published npm tarballs (`expo@56.0.17`, `expo@57.0.8`).
Domain: app configuration (app.json / app.config.js / app.config.ts), config plugins, app icons & splash, and foundational packages (expo-build-properties, expo-constants, expo-system-ui, expo-splash-screen, expo-font, expo-linking, expo-localization).

Sections 1–11 describe **SDK 56**. Everything that changed in SDK 57 lives in the trailing **SDK 57 delta** section; inline `(SDK 57: …)` markers point there.

### What models get wrong from memory

- **There is no `splash` key in the app config.** No top-level `expo.splash`, no `ios.splash`, no `android.splash`. Splash configuration lives **only** in the `expo-splash-screen` config plugin (Section 4/8). (`web.splash` does exist, for PWAs.) Verified against `docs/public/static/schemas/v56.0.0/app-config-schema.json` — the SDK 56 splash-screen doc page still links to dead `../config/app/#splash` anchors.
- **`Linking.useURL()` is the deprecated one**; `Linking.useLinkingURL()` is current. Models routinely state this backwards.
- **`ios.usePrecompiledModules` and `useHermesV1` default to `true`**, not `false` (Section 5). The SDK 56 docs page prints `false` for both — it is stale; the source is authoritative.
- **`Constants.manifest` is deprecated and hidden** (still typed as `EmbeddedManifest | null`, returns `null` whenever `manifest2` is set). Use `Constants.expoConfig`.
- `app.config.ts` supports ESM `import` **without** `tsx`; `tsx` is only needed to import other *TypeScript* files (Section 1).

---

## 1. App Configuration (app.json / app.config.js / app.config.ts)

Source: https://docs.expo.dev/workflow/configuration/

The app config files (**app.json**, **app.config.js**, **app.config.ts**) configure Expo Prebuild generation, project loading in Expo Go, and OTA update manifests. They must live at the project root alongside **package.json**.

Minimal example:
```json
{
  "name": "My app",
  "slug": "my-app"
}
```

### Static vs Dynamic config

**Static** (`app.json` / `app.config.json`):
- Can be automatically updated/modified by CLI tools.

**Dynamic** (`app.config.js` / `app.config.ts`):
- Requires manual developer updates.
- Supports comments, variables, single quotes, and `require()` for Node-compatible files.
- ESM `import` syntax works in **both** `app.config.js` and `app.config.ts`, including importing other JavaScript files. `tsx` is only required to import other *TypeScript* files or to customize language features.
- TypeScript support (nullish coalescing, optional chaining).
- Re-evaluated when Metro reloads.
- **Cannot use Promises** — the final config cannot contain promises.

Additional discovered filenames: `app.config.mts`, `app.config.cts`, `app.config.mjs`, `app.config.cjs`. The resolver order is `.ts, .mts, .cts, .mjs, .cjs, .js` (`packages/@expo/config/src/Config.ts` → `DYNAMIC_CONFIG_EXTS`). By default the config is transpiled to CommonJS and `.js`/`.ts` may mix ESM and CJS syntax; when that mix causes import/require errors, use an explicit extension to lock the module format. Available since **SDK 55** (`@expo/config` 55.0.7, #43242) — `origin/sdk-55:packages/@expo/config/src/Config.ts` already carries the identical `DYNAMIC_CONFIG_EXTS`, so this is not something to upgrade for.

### Configuration resolution order

1. Static config read if **app.config.json** exists (otherwise **app.json**).
2. Dynamic config loads if **app.config.ts** or **app.config.js** exist (TypeScript prioritized).
3. If dynamic config exports a function, the static config is passed as `({ config }) => {}`.
4. The final config cannot contain promises.
5. All functions are evaluated and serialized before use.
6. If a top-level `expo: {}` object exists, it replaces the root.

View the resolved config with `npx expo config`. Verify the public config with `npx expo config --type public`.

### Dynamic config examples

Basic dynamic config:
```js
const myValue = 'My App';

module.exports = {
  name: myValue,
  version: process.env.MY_CUSTOM_PROJECT_VERSION || '1.0.0',
  extra: {
    fact: 'kittens are cool',
  },
};
```

Function-based config that extends static config:
```js
module.exports = ({ config }) => {
  console.log(config.name); // prints 'My App'
  return {
    ...config,
  };
};
```

Environment-based switching:
```js
module.exports = () => {
  if (process.env.MY_ENVIRONMENT === 'production') {
    return {
      /* your production config */
    };
  } else {
    return {
      /* your development config */
    };
  }
};
```

Set environment variables per command:
```sh
MY_ENVIRONMENT=production eas update
```
On Windows: `npx cross-env MY_ENVIRONMENT=production eas update`.

TypeScript config (`app.config.ts`):
```ts
import { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  slug: 'my-app',
  name: 'My App',
});
```
`import` syntax already works here without extra tooling. Add `tsx` only to import other TypeScript files (e.g. local config plugins written in TS) or to customize language features.

### Accessing config at runtime
```js
import Constants from 'expo-constants';

Constants.expoConfig.extra.fact === 'kittens are cool';
```
Avoid importing **app.json** / **app.config.js** directly; use `Constants.expoConfig`.

### Fields filtered from public config / `Constants.expoConfig`
- `hooks`
- `ios.config`
- `android.config`
- `updates.codeSigningCertificate`
- `updates.codeSigningMetadata`

---

## 2. App Config Field Reference

Source: https://docs.expo.dev/versions/v56.0.0/config/app/

### Top-level fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `name` | string | App name as shown in Expo Go and on the home screen | — |
| `description` | string | Brief explanation of app purpose | — |
| `slug` | string | URL-friendly, account-unique project identifier | — |
| `owner` | string | Expo account name for team collaboration | Current user |
| `currentFullName` | string | Auto-generated display name (`@username/slug`); read-only | — |
| `originalFullName` | string | Auto-generated for services; persists across transfers; read-only | — |
| `sdkVersion` | string | Expo SDK version matching package.json | — |
| `runtimeVersion` | string OR object | Native/OTA update compatibility | — |
| `version` | string | App version (iOS: CFBundleShortVersionString) | — |
| `platforms` | array | Supported platforms | `["ios", "android"]` |
| `githubUrl` | string | Repository link | — |
| `orientation` | enum | `default`, `portrait`, `landscape` | `default` |
| `userInterfaceStyle` | enum | `light`, `dark`, `automatic` | `light` |
| `backgroundColor` | string | Root view background (6-char hex) | `#ffffff` |
| `primaryColor` | string | Android multitasker color (6-char hex) | — |
| `icon` | string | App icon path (1024x1024 PNG recommended) | — |
| `androidStatusBar` | object | Android status bar config (deprecated; use `expo-status-bar`) | — |
| `developmentClient` | object | Development client settings | — |
| `scheme` | string OR array | URL scheme(s) for deep linking | — |
| `extra` | object | Custom fields via `Constants.expoConfig.extra` | — |
| `updates` | object | expo-updates configuration | — |
| `locales` | object | Per-locale system dialog strings | — |
| `assetBundlePatterns` | array | **DEPRECATED** — glob patterns for assets bundled into the binary. Use EAS Update asset selection instead | — |
| `plugins` | array | Config plugins | — |
| `buildCacheProvider` | `'eas'` OR `{ plugin: string; options?: object }` | Remote build cache downloading (`plugin` required on the object form) | — |
| `experiments` | object | Unstable experimental features | — |
| `_internal` | object | Internal developer tool properties | — |

There is **no top-level `splash` key** (and none under `ios`/`android`) — see the note at the top of this file.

Read-only / legacy rows you will never author by hand: `currentFullName`, `originalFullName`, `_internal.pluginHistory`, `ios.publishManifestPath`, `ios.publishBundlePath`, `android.publishManifestPath`, `android.publishBundlePath`, `ios.bitcode`, `githubUrl`, `primaryColor`.

### `androidStatusBar` (deprecated; use `expo-status-bar` plugin)

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `barStyle` | enum | `light-content`, `dark-content` | `dark-content` |
| `backgroundColor` | string | 6 or 8-char hex | `#00000000` (transparent) |
| `hidden` | boolean | Visibility toggle | `false` |
| `translucent` | boolean | Float above content | `true` |

### `developmentClient`

| Field | Type | Description |
|-------|------|-------------|
| `silentLaunch` | boolean | Launch without dialogs / progress indicators |

### `updates`

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `enabled` | boolean | Activate updates system | `true` |
| `checkAutomatically` | enum | `ON_LOAD`, `ON_ERROR_RECOVERY`, `WIFI_ONLY`, `NEVER` | `ON_LOAD` |
| `useEmbeddedUpdate` | boolean | Load bundled update | `true` |
| `fallbackToCacheTimeout` | number | Wait time for update check (ms, 0-300000) | `0` |
| `url` | string | Manifest fetch endpoint | — |
| `codeSigningCertificate` | string | Local PEM certificate path | — |
| `codeSigningMetadata` | object | `{ alg: "rsa-v1_5-sha256", keyid: string }` | — |
| `requestHeaders` | object | Extra HTTP headers | — |
| `assetPatternsToBeBundled` | array | Glob patterns for included assets | All assets |
| `disableAntiBrickingMeasures` | boolean | Override anti-bricking protections | `false` |
| `useNativeDebug` | boolean | Enable native code debugging | `false` |
| `enableBsdiffPatchSupport` | boolean | Support bundle diffs via bsdiff | `true` |

### `ios`

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `appleTeamId` | string | Apple Developer team ID | — |
| `publishManifestPath` | string | Manifest output path | — |
| `publishBundlePath` | string | Bundle output path | — |
| `bundleIdentifier` | string | Unique App Store identifier (e.g. `host.exp.expo`) | — |
| `buildNumber` | string | Build version (CFBundleVersion) | — |
| `deploymentTarget` | string | Minimum iOS version (e.g. `"18.6"` or `"26"`) | — |
| `backgroundColor` | string | App background (6-char hex) | — |
| `scheme` | string OR array | iOS-specific URL scheme(s) | — |
| `icon` | string OR object | Icon path or appearance object (light/dark/tinted) | — |
| `appStoreUrl` | string | App Store listing URL | — |
| `bitcode` | undefined | Enable Bitcode optimization | — |
| `config` | object | Private API keys (not in production manifest) | — |
| `googleServicesFile` | string | GoogleService-Info.plist path | — |
| `supportsTablet` | boolean | Support iPad sizes | `false` |
| `isTabletOnly` | boolean | iPad-only app | — |
| `requireFullScreen` | boolean | Disable Split View / Slide Over | `false` |
| `userInterfaceStyle` | enum | `light`, `dark`, `automatic` | `light` |
| `infoPlist` | object | Arbitrary Info.plist additions | — |
| `entitlements` | object | Entitlements additions | — |
| `privacyManifests` | object | PrivacyInfo.xcprivacy definitions | — |
| `associatedDomains` | array | Associated Domains capability entries | — |
| `usesIcloudStorage` | boolean | iCloud Storage for DocumentPicker | — |
| `usesAppleSignIn` | boolean | Enable Apple Sign-In | — |
| `usesBroadcastPushNotifications` | boolean | Push Notifications Broadcast | — |
| `accessesContactNotes` | boolean | Access contact notes (requires Apple permission) | — |
| `runtimeVersion` | string OR object | iOS-specific OTA compatibility | — |
| `version` | string | iOS version (overrides root) | — |

- `ios.icon` (object): `light`, `dark`, `tinted` (each a string path).
- `ios.config`: `usesNonExemptEncryption` (boolean), `googleMapsApiKey` (string).
- `ios.privacyManifests`: `NSPrivacyAccessedAPITypes`, `NSPrivacyTrackingDomains`, `NSPrivacyTracking`, `NSPrivacyCollectedDataTypes`.
- `ios.runtimeVersion.policy`: `nativeVersion`, `sdkVersion`, `appVersion`, `fingerprint`.

### `android`

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `publishManifestPath` | string | Manifest output path | — |
| `publishBundlePath` | string | Bundle output path | — |
| `package` | string | Unique Play Store identifier (e.g. `com.example.app`) | — |
| `versionCode` | integer | Play Store version (increment per release) | — |
| `backgroundColor` | string | App background (6-char hex) | — |
| `userInterfaceStyle` | enum | `light`, `dark`, `automatic` | `light` |
| `scheme` | string OR array | Android-specific URL scheme(s) | — |
| `icon` | string | App icon path (1024x1024 PNG) | — |
| `adaptiveIcon` | object | Adaptive launcher icon config | — |
| `playStoreUrl` | string | Play Store listing URL | — |
| `permissions` | array | Manifest permissions to add | — |
| `blockedPermissions` | array | Permissions to remove from merged manifest | — |
| `googleServicesFile` | string | google-services.json path | — |
| `config` | object | Private API keys (not in production manifest) | — |
| `intentFilters` | array | Custom intent filters | — |
| `allowBackup` | boolean | Enable Google Drive app data backup | `true` |
| `softwareKeyboardLayoutMode` | enum | `resize`, `pan` | `resize` |
| `runtimeVersion` | string OR object | Android-specific OTA compatibility | — |
| `version` | string | Android version (overrides root) | — |
| `predictiveBackGestureEnabled` | boolean | Predictive back gesture (API 33+) | `false` |

- `android.adaptiveIcon`: `foregroundImage`, `monochromeImage` (Android 13+), `backgroundImage`, `backgroundColor` (default `#FFFFFF`).
- `android.config`: `googleMaps.apiKey` (string).
- `android.intentFilters[]`: `autoVerify` (boolean), `action` (string, e.g. `VIEW`), `data` (object), `category` (array).
- `android.runtimeVersion.policy`: `nativeVersion`, `sdkVersion`, `appVersion`, `fingerprint`.

intentFilters example:
```json
[{
  "autoVerify": true,
  "action": "VIEW",
  "data": { "scheme": "https", "host": "*.example.com" },
  "category": ["BROWSABLE", "DEFAULT"]
}]
```

### `web`

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `output` | enum | `single` (SPA), `static` (router), `server` (Node.js) | `single` |
| `favicon` | string | Favicon image relative path | — |
| `name` | string | Document title | Root `name` |
| `shortName` | string | App launcher/tab name (≤12 chars) | `name` |
| `lang` | string | Primary language tag | — |
| `scope` | string | Navigation scope URL | — |
| `themeColor` | string | Android toolbar color (6-char hex) | — |
| `description` | string | Website purpose | — |
| `dir` | enum | `auto`, `ltr`, `rtl` | — |
| `display` | enum | `fullscreen`, `standalone`, `minimal-ui`, `browser` | — |
| `startUrl` | string | Launch URL (relative to manifest) | — |
| `orientation` | enum | `any`, `natural`, `landscape`, `portrait`, `*-primary`, `*-secondary` | — |
| `backgroundColor` | string | Background (6-char hex) | — |
| `barStyle` | enum | `default`, `black`, `black-translucent` | — |
| `preferRelatedApplications` | boolean | Promote native apps over website | — |
| `dangerous` | object | Experimental features | — |
| `splash` | object | PWA splash (`backgroundColor`, `resizeMode` cover/contain default `contain`, `image`) | — |
| `config` | object | Firebase web config (apiKey, authDomain, databaseURL, projectId, storageBucket, messagingSenderId, appId, measurementId) | — |
| `bundler` | enum | `webpack`, `metro` | Auto-detected |

### `experiments` (unstable)

`outOfTreePlatforms` (boolean, fallback `false` — enable select out-of-tree platforms when their support packages are installed), `onDemandFilesystem`, `autolinkingModuleResolution`, `baseUrl`, `buildCacheProvider` (**deprecated** — use the top-level `buildCacheProvider`), `supportsTVOnly`, `tsconfigPaths`, `typedRoutes`, `turboModules`, `reactCanary`, `reactCompiler`, `reactServerComponentRoutes`, `reactServerFunctions`, `inlineModules` (`{ watchedDirectories: array }`).

### `_internal`
- `pluginHistory` (object): plugins already run on config.

---

## 3. Config Plugins

Sources: https://docs.expo.dev/config-plugins/introduction/ , https://docs.expo.dev/workflow/configuration/

Config plugins extend Expo's default app config to modify native Android/iOS projects during the prebuild process (Continuous Native Generation / CNG) without editing native files directly. "A config plugin is a top-level custom configuration point that is not built into the app config."

(SDK 57: `npx expo prebuild` now **cleans** by default — see **SDK 57 delta**.)

### Plugin structure / glossary
- **Plugin** — entry referenced in the `plugins` array; conventionally named `with<PluginName>` (e.g. `withMyPlugin`).
- **Plugin Functions** — platform-specific wrapper functions inside a plugin that perform modifications.
- **Mod Plugin Functions** — wrapper utilities from `expo/config-plugins` that safely modify native files via underlying mods.
- **Mods** — platform-specific modifiers (`mods.android.manifest`, `mods.ios.infoplist`) that directly alter native project files during prebuild.

### Key characteristics
- "Plugins are synchronous functions that accept an ExpoConfig and return a modified ExpoConfig."
- Naming convention: `with<Plugin Functionality>`.
- Return values should be serializable (excluding `mods`).
- Mods execute only during the prebuild syncing phase.
- App config modifications must occur outside mods for non-prebuild scenarios.

### Usage syntax in `plugins` array
- Plain string: `"expo-system-ui"`.
- Array with props: `["expo-splash-screen", { /* props */ }]`.
- Local plugin: a path such as `"./plugins/withMyPlugin"`. Resolution accepts `.js`, `.cjs`, `.mjs`, `.ts`, `.cts`, `.mts` (aligned with app-config resolution; `@expo/config-plugins` 56.0.7, PR #45989).

### Import surface (`expo/config-plugins`)

The exports a plugin actually reaches for:

| Export | Use |
|--------|-----|
| `withPlugins(config, [pluginA, [pluginB, props]])` | Compose several plugins in order |
| `withInfoPlist` / `withEntitlementsPlist` / `withExpoPlist` | iOS plist mods |
| `withAppDelegate` / `withXcodeProject` / `withPodfile` / `withPodfileProperties` | iOS project mods |
| `withAndroidManifest` / `withStringsXml` / `withAndroidColors` / `withAndroidColorsNight` / `withAndroidStyles` | Android resource + manifest mods |
| `withMainActivity` / `withMainApplication` / `withAppBuildGradle` / `withProjectBuildGradle` / `withSettingsGradle` / `withGradleProperties` | Android project mods |
| `withDangerousMod(config, ['ios' \| 'android', fn])` | Arbitrary filesystem access during prebuild; last resort |
| `withFinalizedMod`, `withMod`, `withBaseMod`, `withStaticPlugin` | Lower-level mod plumbing |
| `withRunOnce`, `createRunOncePlugin` | Idempotency guard for plugins applied more than once |
| `AndroidConfig`, `IOSConfig` | Namespaced helpers (e.g. `AndroidConfig.Strings.setStringItem`) |
| `CodeGenerator.mergeContents` | Idempotent insertion of a tagged block into an existing native file |
| `WarningAggregator` | Emit prebuild warnings instead of throwing |
| `Updates`, `History`, `XML`, `BaseMods`, `PluginError` | Utilities |

Minimal plugin:
```ts
import type { ConfigPlugin } from 'expo/config-plugins';
import { withAndroidManifest, withInfoPlist, withPlugins } from 'expo/config-plugins';

const withAndroidBits: ConfigPlugin<{ label?: string }> = (config, props = {}) =>
  withAndroidManifest(config, (config) => {
    // config.modResults is the parsed AndroidManifest.xml
    return config;
  });

const withIosBits: ConfigPlugin<{ label?: string }> = (config, props = {}) =>
  withInfoPlist(config, (config) => {
    config.modResults.CFBundleDisplayName = props.label ?? config.modResults.CFBundleDisplayName;
    return config;
  });

const withMyPlugin: ConfigPlugin<{ label?: string }> = (config, props = {}) =>
  withPlugins(config, [
    [withAndroidBits, props],
    [withIosBits, props],
  ]);

export default withMyPlugin;
```

For a published npm package, `app.plugin.js` at the package root is the entry point the `plugins` array resolves to. It must **default-export a `ConfigPlugin`** — `(config, props) => config`. `resolveConfigPluginExport` (`@expo/config-plugins/src/utils/plugin-resolver.ts`) unwraps `.default` and then calls the export as `plugin(config, props)`. Example: `expo-localization/app.plugin.js` is one line — `module.exports = require('./plugin/build/withExpoLocalization')` — and `withExpoLocalization.ts` ends with `export default withExpoLocalization`.

Separately, a package may ship an optional **typed-plugin helper** — `(props) => [name, props]` — imported from `app.config.ts` for type-checked plugin entries (`expo-localization/plugin/src/index.ts`: `export default (props = {}) => ['expo-localization', props]`). That is *not* an `app.plugin.js` entry point; wiring it there returns a tuple where prebuild expects a config and breaks the build.

---

## 4. App Icons & Splash Screen

Source: https://docs.expo.dev/develop/user-interface/splash-screen-and-app-icon/

### Splash screen image requirements
- 1024x1024 image, `.png` file, transparent background.
- Default file: `assets/images/splash-icon.png`.
- Testing: build for internal distribution or production. **"Do not use Expo Go or a development build to test your splash screen."**

Plugin config:
```json
{
  "expo": {
    "plugins": [
      [
        "expo-splash-screen",
        {
          "backgroundColor": "#232323",
          "image": "./assets/images/splash-icon.png",
          "dark": {
            "image": "./assets/images/splash-icon-dark.png",
            "backgroundColor": "#000000"
          },
          "imageWidth": 200
        }
      ]
    ]
  }
}
```

Platform-specific:
```json
{
  "expo": {
    "plugins": [
      [
        "expo-splash-screen",
        {
          "ios": {
            "backgroundColor": "#ffffff",
            "image": "./assets/images/splash-icon.png",
            "resizeMode": "cover"
          },
          "android": {
            "backgroundColor": "#0c7cff",
            "image": "./assets/images/splash-android-icon.png",
            "imageWidth": 150
          }
        }
      ]
    ]
  }
}
```

### App icon
Basic single icon — 1024x1024 PNG, exactly square, fills the whole square with no rounded corners. Default `assets/images/icon.png`:
```json
{
  "icon": "./assets/images/icon.png"
}
```

iOS — Icon Composer `.icon` directory (supported SDK 54+):
```json
{
  "expo": {
    "ios": {
      "icon": "./assets/app.icon"
    }
  }
}
```

iOS — PNG variants (light / dark / tinted):
```json
{
  "expo": {
    "ios": {
      "icon": {
        "dark": "./assets/images/ios-dark.png",
        "light": "./assets/images/ios-light.png",
        "tinted": "./assets/images/ios-tinted.png"
      }
    }
  }
}
```

Android — adaptive icon (foreground / background / monochrome):
```json
{
  "expo": {
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/images/foreground.png",
        "backgroundColor": "#FFFFFF",
        "monochromeImage": "./assets/images/monochrome.png",
        "backgroundImage": "./assets/images/background.png"
      }
    }
  }
}
```
A legacy `android.icon` may be provided for older non-adaptive devices.

---

## 5. expo-build-properties

Source: https://docs.expo.dev/versions/v56.0.0/sdk/build-properties/

Config plugin to customize native build properties during Prebuild for Android, iOS, and tvOS.

```sh
npx expo install expo-build-properties
```

Example:
```json
{
  "expo": {
    "plugins": [
      [
        "expo-build-properties",
        {
          "android": {
            "compileSdkVersion": 36,
            "targetSdkVersion": 36,
            "buildToolsVersion": "36.0.0"
          },
          "ios": {
            "deploymentTarget": "16.4"
          }
        }
      ]
    ]
  }
}
```

### Android properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `compileSdkVersion` | number | — | Override compile SDK version in build.gradle |
| `targetSdkVersion` | number | — | Override target SDK version |
| `minSdkVersion` | number | — | Override minimum SDK version |
| `buildToolsVersion` | string | — | Override build tools version |
| `kotlinVersion` | string | — | Override Kotlin version |
| `buildArchs` | string[] | `["armeabi-v7a", "arm64-v8a", "x86", "x86_64"]` | Override ABI architectures |
| `enableMinifyInReleaseBuilds` | boolean | — | Enable R8 obfuscation in release builds |
| `enableShrinkResourcesInReleaseBuilds` | boolean | — | Enable resource shrinking in release builds |
| `enablePngCrunchInReleaseBuilds` | boolean | `true` | Enable PNG optimization in release builds |
| `enableBundleCompression` | boolean | `false` | Enable JS bundle compression |
| `useLegacyPackaging` | boolean | `false` | Use legacy native library compression in APK |
| `usesCleartextTraffic` | boolean | Platform-dependent | Allow cleartext network traffic |
| `useDayNightTheme` | boolean | — | Apply DayNight theme variant for dark mode |
| `networkInspector` | boolean | `true` | Enable Network Inspector |
| `extraProguardRules` | string | — | Append custom Proguard rules to proguard-rules.pro |
| `extraMavenRepos` | (string \| AndroidMavenRepository)[] | — | Add Maven repos with optional credentials |
| `exclusiveMavenMirror` | string | — | Single Maven repository as exclusive mirror |
| `packagingOptions` | object | — | Native library packaging (exclude, merge, pickFirst, doNotStrip) |
| `manifestQueries` | object | — | Packages, intents, providers the app interacts with |
| `buildReactNativeFromSource` | boolean | `false` | Build React Native from source (increases build time) |
| `buildFromSource` | boolean | `false` | Deprecated: use `buildReactNativeFromSource` |
| `usePrecompiledHeaders` | boolean | `false` | **@experimental.** Generates a custom CMakeLists.txt with PCH support for all autolinked native libraries, pre-compiling common RN headers to speed up C++ compilation. Also togglable via `EXPO_USE_ANDROID_PRECOMPILED_HEADERS=1`. May not work with all native libraries |

### iOS properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `deploymentTarget` | string | — | Override iOS deployment target (deprecated: use built-in `ios.deploymentTarget`) |
| `useFrameworks` | `'static'` \| `'dynamic'` | — | Enable `use_frameworks!` for static/dynamic linking |
| `forceStaticLinking` | string[] | — | CocoaPods to link statically instead of frameworks |
| `extraPods` | ExtraIosPodDependency[] | — | Add extra CocoaPods to Podfile |
| `networkInspector` | boolean | `true` | Enable Network Inspector |
| `privacyManifestAggregationEnabled` | boolean | — | Aggregate Privacy Manifests from CocoaPods |
| `ccacheEnabled` | boolean | — | Enable C++ compiler cache for faster builds |
| `usePrecompiledModules` | boolean | `true` | Use precompiled Expo modules (XCFrameworks) instead of building from source; sets `EXPO_USE_PRECOMPILED_MODULES=1` during `pod install`. Set `false` to build modules from source |

### Shared (Android & iOS) — SDK 56 highlights

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `buildReactNativeFromSource` | boolean | `false` | Build RN from source; on iOS controls precompiled xcframework usage |
| `reactNativeReleaseLevel` | `'stable'` \| `'canary'` \| `'experimental'` | `'stable'` | RN release level for feature flags |
| `useHermesV1` | boolean | `true` | Hermes V1 is the **default** JS engine since SDK 56 / RN 0.84. Setting `false` falls back to legacy Hermes and requires **both** `buildReactNativeFromSource: true` (per platform) **and** pinning the legacy hermes-compiler version via package.json `resolutions` — the plugin throws otherwise |

> **Docs-vs-source drift in SDK 56.** The SDK 56 reference page prints `false` for both `usePrecompiledModules` and `useHermesV1`, and omits `android.usePrecompiledHeaders` entirely. All three rows above are taken from `packages/expo-build-properties/src/pluginConfig.ts` + `src/ios.ts` and are what the plugin actually does. `usePrecompiledModules` was defaulted to `true` in 56.0.14 (PR #46159); the `useHermesV1` default was documented as `true` in 56.0.15 (PR #46211); `usePrecompiledHeaders` first shipped in **56.0.10** (published 2026-05-19) but was only surfaced on the docs page from SDK 57. Note the trap: PR #45922 merged 2026-05-18 and its in-tree `package.json` read `56.0.9`, because 56.0.9 had already been published on 2026-05-15 — in-tree versions name the *previous* publish, not the release that contains the change.

### API methods
- `BuildProperties.resolveConfigValue(config, platform, key)` — resolves shared config values; platform-specific overrides take precedence.
- `BuildProperties.withBuildProperties(config, props)` — main config plugin.

---

## 6. expo-constants

Source: https://docs.expo.dev/versions/v56.0.0/sdk/constants/

System information that stays constant across an app installation lifecycle. Supported: Android, iOS, tvOS, Web, Expo Go.

```sh
npx expo install expo-constants
```
```js
import Constants from 'expo-constants';
```

### Constants object properties
- **Config:** `expoConfig` (resolved Expo config from app.json/app.config.js), `expoGoConfig` (when running in Expo Go), `manifest2` (modern Expo Updates manifest for EAS Update), `platform` (platform-specific manifest object).
- **Device:** `deviceName`, `statusBarHeight`, `systemFonts` (array of available system font names).
- **Session & runtime:** `sessionId` (unique per app session), `executionEnvironment`, `debugMode` (`__DEV__`), `isHeadless`, `linkingUri: string`, `experienceUrl`.
- **Version:** `expoVersion` (null in bare/web), `expoRuntimeVersion` (nullable on web).
- **Deprecated but still present:** `manifest: EmbeddedManifest | null` (hidden; `null` whenever `manifest2` is set — use `expoConfig`), `deviceYearClass: number | null` (use `Device.deviceYearClass`), `appOwnership`.

### Methods
- `getWebViewUserAgentAsync()` → `Promise<string | null>`.

### Enums
- `ExecutionEnvironment`: `Bare`, `Standalone`, `StoreClient`.
- `UserInterfaceIdiom`: `Handset`, `Tablet`, `Desktop`, `TV`, `Unsupported`.

### SDK 56 notes
- `appOwnership` deprecated in favor of `executionEnvironment`.
- iOS properties `model`, `platform`, `systemVersion`, `userInterfaceIdiom`, and `deviceYearClass` moved to **expo-device**. `deviceYearClass` still exists on the `Constants` object (typed `number | null`) but is deprecated — prefer `Device.deviceYearClass`.

---

## 7. expo-system-ui

Source: https://docs.expo.dev/versions/v56.0.0/sdk/system-ui/

Controls system UI elements outside the React tree — root view background color and (Android) locking interface style.

```sh
npx expo install expo-system-ui
```

Config plugin:
```json
{
  "expo": {
    "backgroundColor": "#ffffff",
    "userInterfaceStyle": "light",
    "plugins": ["expo-system-ui"]
  }
}
```

Manual native setup (non-CNG):
- Android — `android/app/src/main/res/values/strings.xml`:
  ```xml
  <string name="expo_system_ui_user_interface_style">light</string>
  ```
- iOS — `ios/your-app/Info.plist`:
  ```xml
  <key>UIUserInterfaceStyle</key>
  <string>Light</string>
  ```

### API
- `getBackgroundColorAsync()` — gets root view background color; returns hex color or null. (Android, iOS, tvOS, Web)
- `setBackgroundColorAsync(color)` — sets root view background color; accepts CSS3/SVG color values, e.g. `SystemUI.setBackgroundColorAsync("black")`. (Android, iOS, tvOS, Web)

---

## 8. expo-splash-screen

Source: https://docs.expo.dev/versions/v56.0.0/sdk/splash-screen/

```sh
npx expo install expo-splash-screen
```

Config plugin JSON examples live in **Section 4** (shared + platform-specific). Reminder: there is no `expo.splash` app-config key — this plugin is the only supported surface.

### Plugin properties

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `backgroundColor` | string | `#ffffff` | Hex color for splash background |
| `image` | string | `undefined` | Path to app icon/logo image |
| `imageWidth` | number | `100` | Image display width |
| `resizeMode` | `'contain'` \| `'cover'` \| `'native'` | `'contain'` | How the image is scaled (docs page prints `undefined`; source TSDoc says `contain`) |
| `dark` | `{ image?, backgroundColor? }` | `undefined` | Dark-mode overrides |
| `enableFullScreenImage_legacy` | boolean | `false` | iOS only; legacy full-screen image support |
| `android` | `Partial<AndroidSplashConfig>` | `undefined` | Android-specific settings (below) |
| `ios` | `Partial<IOSSplashConfig>` | `undefined` | iOS-specific settings (below) |

Platform sub-shapes (`packages/expo-splash-screen/plugin/src/types.ts`):

```ts
type BaseAndroidSplashConfig = {
  backgroundColor?: string; image?: string;
  mdpi?: string; hdpi?: string; xhdpi?: string; xxhdpi?: string; xxxhdpi?: string;
};
type AndroidSplashConfig = BaseAndroidSplashConfig & {
  backgroundColor: string;
  drawable?: { icon: string; darkIcon?: string };
  imageWidth: number;
  resizeMode: 'contain' | 'cover' | 'native';
  dark?: BaseAndroidSplashConfig;
};

type BaseIOSSplashConfig = {
  backgroundColor?: string; image?: string;
  tabletBackgroundColor?: string; tabletImage?: string;
};
type IOSSplashConfig = BaseIOSSplashConfig & {
  backgroundColor: string;
  enableFullScreenImage_legacy: boolean;
  imageWidth: number;
  resizeMode: 'cover' | 'contain'; // no 'native' on iOS
  dark?: BaseIOSSplashConfig;
};
```

### API methods
- `preventAutoHideAsync()` — keep splash visible until manually hidden. `Promise<boolean>`.
- `hide()` — immediately hide splash. `void`.
- `hideAsync()` — async variant (backwards compatibility). `Promise<void>`.
- `setOptions(options)` — animation behavior; `duration` (ms) and `fade` (iOS only, boolean).

### SDK 56 notes
- Since SDK 52, due to the latest Android splash screen API changes, Expo Go and development builds cannot fully replicate the standalone-app splash experience.
- **Patch trap: pin ≥ 56.0.7.** On 56.0.0–56.0.6 the package `exports` field resolved to the *web* stubs on native, silently turning `preventAutoHideAsync()` (and friends) into no-ops. Fixed in 56.0.7 (PR #45798). The SDK 56 pin is `~56.0.14` (`bundledNativeModules.json` in `expo@56.0.17`), so a fresh `expo install` is already past it.

---

## 9. expo-font

Source: https://docs.expo.dev/versions/v56.0.0/sdk/font/

```sh
npx expo install expo-font
```

### Config plugin (recommended — embeds fonts at build time)
```json
{
  "expo": {
    "plugins": [
      [
        "expo-font",
        {
          "fonts": ["./path/to/file.ttf"],
          "android": {
            "fonts": [{
              "fontFamily": "Source Serif 4",
              "fontDefinitions": [{
                "path": "./path/to/SourceSerif4-ExtraBold.ttf",
                "weight": 800
              }]
            }]
          }
        }
      ]
    ]
  }
}
```
- `fonts` — array of font file paths (relative to project root). Package-style paths such as `@expo-google-fonts/inter/400Regular/Inter_400Regular.ttf` also resolve on **expo-font ≥ 56.0.7** (#46784) — `resolveFontPaths()` catches `ENOENT` from `fs.stat(path.resolve(projectRoot, p))` and retries with `Module.createRequire(...).resolve(p)`. The SDK 56 pin is `~56.0.7`, so this is not an SDK 57 feature.
- `android` — platform-specific definitions with custom family names.
- `ios` — array of font file paths; family names drawn directly from font files.

### Runtime loading with useFonts
```tsx
const [loaded, error] = useFonts({
  'Inter-Black': require('./assets/fonts/Inter-Black.otf'),
});
```
Wrap with `SplashScreen` to prevent early rendering before fonts load.

### API
- `loadAsync(fontFamilyOrFontMap, source)` — load fonts from static/remote resources; `Promise`.
- `isLoaded(fontFamily: string): boolean` — synchronous.
- `isLoading(fontFamily: string): boolean` — synchronous; `true` while a `loadAsync` for that family is still in flight.
- `getLoadedFonts()` — all loaded fonts (incl. build-time embedded); string array usable as `fontFamily`.
- `renderToImageAsync(glyphs: string, options?: RenderToImageOptions): Promise<RenderToImageResult>` — rasterize glyphs (e.g. an icon font) to an image usable as an `Image` `source`.

Supported: Android, iOS, tvOS, Web, Expo Go.

---

## 10. expo-linking (+ deep linking)

Sources: https://docs.expo.dev/versions/v56.0.0/sdk/linking/ , https://docs.expo.dev/linking/into-your-app/

```sh
npx expo install expo-linking
```
```js
import * as Linking from 'expo-linking';
```

### Methods
- `createURL(path, namedParameters)` — builds deep links with optional query params; supports double or triple slashes. Schemes from `expo.scheme` or platform-specific `expo.{android,ios}.scheme` take precedence. Returns a URL string.
- `parse(url)` — parses deep link info; returns `ParsedURL` (hostname, path, scheme, queryParams).
- `openURL(url)` — open URL with an installed app; `Promise<boolean>` (rejects if no handler).
- `canOpenURL(url)` — whether an installed app can handle the URL; always `true` on web; may reject on Android/iOS without proper config.
- `getInitialURL()` — URL that launched the app; `Promise<string | null>`. (Not deprecated, but `getLinkingURL()` is the synchronous equivalent.)
- `getLinkingURL(): string | null` — synchronous; the URL that launched the app, or `null`.
- `parseInitialURLAsync(): Promise<ParsedURL>` — `getInitialURL()` + `parse()` in one call.
- `sendIntent(action: string, extras?: SendIntentExtras[]): Promise<void>` — Android only; launch an Android activity intent.
- `openSettings()` — opens OS settings for the app.
- `resolveScheme(options: { scheme?: string; isSilent?: boolean }): string` — pick the scheme used by `createURL`. When several schemes are declared and none is passed it **warns and returns the first manifest scheme** (`Linking found multiple possible URI schemes … Using '<first>'. Ignoring: …`); `isSilent` suppresses that warning, it does not change the return value. It **throws** when no scheme is configured at all, and when expo-constants exposes no manifest — neither throw is gated on `isSilent`. In Expo Go it returns `'exp'`.
- `collectManifestSchemes(): string[]` — every scheme declared in the manifest.
- `hasCustomScheme(): boolean`, `hasConstantsManifest(): boolean` — environment probes used by the above.
- `clearInitialURL(): void` — clears the cached launch URL so subsequent `getLinkingURL()` calls return `null` until a new deep link arrives. No-op on web; `@platform android`, `@platform ios`. Added in `expo-linking` **56.0.13** (#46265); the SDK 56 pin is `~56.0.16`, so it is available in SDK 56 projects — verified in the published `expo-linking@56.0.16` tarball (`build/Linking.d.ts` → `export declare function clearInitialURL(): void;`). It is **not** an SDK 57 feature; the frozen v56 docs JSON that omits it is a release-time snapshot that lags patch releases.

### Hooks
- `useLinkingURL(): string | null` — **current hook.** Returns the linking URL and any subsequent changes; the initial URL is available immediately on first render.
- `useURL(): string | null` — **DEPRECATED**, use `useLinkingURL()`. Same return type, but resolves the initial URL asynchronously (first render is `null`). The deprecation direction is frequently stated backwards — `useURL` is the old one.

### Events
- `addEventListener('url', handler)` — listen for URL changes (hooks preferred).

### Types
- `ParsedURL`: hostname, path, scheme, queryParams.
- `CreateURLOptions`: isTripleSlashed, queryParams, scheme.
- `QueryParams`: `Record<string, undefined | string | string[]>`.

### Deep linking configuration
```json
{
  "expo": {
    "scheme": "myapp"
  }
}
```
After configuring, create a new development build. If no custom scheme is defined, the system uses `android.package` and `ios.bundleIdentifier` as fallback schemes.

Handle/parse incoming URLs:
```tsx
import * as Linking from 'expo-linking';

const url = Linking.useLinkingURL();
const { hostname, path, queryParams } = Linking.parse(url);
```

Test with `uri-scheme`:
```sh
npx uri-scheme open myapp://somepath/details --ios
npx uri-scheme open com.example.app://somepath/details --android
# Expo Go:
npx uri-scheme open exp://127.0.0.1:8081/--/somepath/details --ios
```
Android App Links / iOS Universal Links handle HTTP(S) links and fall back to your website when the app is not installed.

Supported: Android, iOS, tvOS, Web, Expo Go (limitations for published updates).

---

## 11. expo-localization

Source: https://docs.expo.dev/versions/v56.0.0/sdk/localization/

```sh
npx expo install expo-localization
```

Config plugin — the docs page shows only the bare string form, but the plugin accepts four props (`packages/expo-localization/plugin/src/withExpoLocalization.ts`):

```ts
type ConfigPluginProps = {
  supportsRTL?: boolean;
  forcesRTL?: boolean;
  allowDynamicLocaleChangesAndroid?: boolean;
  supportedLocales?: string[] | { ios?: string[]; android?: string[] };
};
```
```json
{
  "expo": {
    "plugins": [
      ["expo-localization", { "supportsRTL": true, "supportedLocales": ["en-US", "ar-EG"] }]
    ]
  }
}
```
Since 56.0.5 (PR #45888) every `supportedLocales` entry is prevalidated as a BCP-47 tag via `new Intl.Locale(tag)`; an invalid tag throws at prebuild with `Invalid supportedLocales entry …: must be a BCP-47 locale tag.`

```js
import { getLocales, getCalendars } from 'expo-localization';
```
Both methods are synchronous. On iOS, results stay constant during runtime. On Android, user locale changes apply immediately without app restart — use `AppState` to refresh on foreground.

### Hooks
- `useLocales()` — array of `Locale` (min 1), ordered by device preference; rerenders on OS setting changes. Web returns `null` for currency and measurement systems.
- `useCalendars()` — array of `Calendar` (min 1; currently single element); rerenders on OS setting changes.

### Methods
- `getLocales()` → `[Locale, ...Locale[]]`.
- `getCalendars()` → `[Calendar, ...Calendar[]]`.

### Locale fields
`languageTag` (e.g. `'en-US'`), `languageCode` (e.g. `'en'`), `regionCode`, `textDirection` (`'ltr'`/`'rtl'`), `currencyCode`, `currencySymbol`, `decimalSeparator`, `digitGroupingSeparator`, `measurementSystem` (`'metric'`/`'us'`/`'uk'`/null), `temperatureUnit` (`'celsius'`/`'fahrenheit'`/null), `languageScriptCode` (ISO 15924).

### Calendar fields
`calendar` (Unicode calendar type), `timeZone` (e.g. `'Europe/Warsaw'`), `uses24hourClock` (boolean or null), `firstWeekday` (1–7 Sunday–Saturday, or null).

Supported: Android, iOS, tvOS, Web, Expo Go.

---

## Source URL index

Substitute `v57.0.0` for `v56.0.0` in any versioned URL below for the SDK 57 page.

- App config guide: https://docs.expo.dev/workflow/configuration/
- App config field reference (SDK 56): https://docs.expo.dev/versions/v56.0.0/config/app/
- Config plugins introduction: https://docs.expo.dev/config-plugins/introduction/
- Splash screen & app icon guide: https://docs.expo.dev/develop/user-interface/splash-screen-and-app-icon/
- expo-build-properties (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/build-properties/
- expo-constants (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/constants/
- expo-system-ui (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/system-ui/
- expo-splash-screen (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/splash-screen/
- expo-font (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/font/
- expo-linking (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/linking/
- Deep linking guide: https://docs.expo.dev/linking/into-your-app/
- expo-localization (SDK 56): https://docs.expo.dev/versions/v56.0.0/sdk/localization/

---

## SDK 57 delta

Verified against `origin/sdk-56` / `origin/sdk-57` and published npm tarballs, **not** against `main` (which is SDK 58 in progress) and not against the frozen docs schemas (which lag patch releases).

The app config **schema is unchanged**. In this whole domain there are exactly **three** real 56 → 57 changes: two breaking (`expo prebuild` cleans by default; `expo-font` server context) and one additive (`android.cmakeVersion`). Sections 1, 2, 6, 7, 8, 10 and 11 are unchanged. Everything else that looks like a 57 feature in this domain was **also backported into the SDK 56 patch line** — see *Already in SDK 56* below before you plan an upgrade around it.

### Breaking

**`npx expo prebuild` now cleans by default.** It clears and regenerates `ios/` and `android/` instead of applying changes additively. Pass `--no-clean` to restore the SDK 56 behaviour (`--clean` still parses, but it is now the default). Any hand-edits inside the native folders, and any workflow that relied on incremental prebuild, break silently.
Confirmed 57-only: `origin/sdk-57:packages/@expo/cli/src/prebuild/index.ts` has `'--no-clean'` and `clean: !args['--no-clean']`; `origin/sdk-56` has neither (`clean: args['--clean']`, opt-in). #47209 appears in the sdk-57 `@expo/cli` CHANGELOG and **not** in the sdk-56 one.

**`expo-font` server context (Section 9, server-side only).** `Server.resetServerContext()` was **removed**; the replacement is `Server.withServerContext(callback)`, which scopes server-side font state per render via `AsyncLocalStorage`. Server-side font loading outside that callback throws `expo-font server context accessed outside of withServerContext(). Wrap your server-side font usage in withServerContext(() => /* server code */).` (verbatim from `serverContext.web.ts` → `requireStore()`). This lives on the `expo-font/build/server` subpath export — confirmed in the sdk-57 `package.json` `exports` map — not the public SDK surface; see reference 05 for SSR details.
Confirmed 57-only: `origin/sdk-57:packages/expo-font/src/server.ts` re-exports `withServerContext` and has no `resetServerContext`; `origin/sdk-56:packages/expo-font/src/server.ts` still exports `resetServerContext()`. #46669 is in the sdk-57 `expo-font` CHANGELOG only.

### New in 57

- **`android.cmakeVersion`** (Section 5) — new `expo-build-properties` key overriding the CMake version used to build native code. Shipped in `expo-build-properties` **57.0.2** (2026-06-30, #47377), before 57.0.0 (2026-07-08), so the SDK 57 pin resolves to a version that has it. Genuinely 57-only: `origin/sdk-56:packages/expo-build-properties/src/pluginConfig.ts` has no `cmakeVersion` and the sdk-56 CHANGELOG has no #47377. Source: `origin/sdk-57:packages/expo-build-properties/{CHANGELOG.md,src/pluginConfig.ts}` (`cmakeVersion?: string`, `cmakeVersion: { type: 'string', nullable: true }`).

That is the entire additive surface for this domain in SDK 57.

### Already in SDK 56 — do NOT upgrade for these

Each of these was backported to the SDK 56 patch line. Bump the patch instead; upgrading to 57 buys nothing here.

- **Package-style font paths in the `expo-font` config plugin** (Section 9) — needs **expo-font ≥ 56.0.7** (#46784, 2026-06-15). `origin/sdk-56:packages/expo-font/plugin/src/utils.ts` already has the `ENOENT` catch plus `Module.createRequire(...).resolve(p)` fallback, confirmed in the published `expo-font@56.0.7` tarball. The SDK 56 pin is `~56.0.7`, so a fresh install already has it.
- **`android.usePrecompiledHeaders`** (Section 5) — needs **expo-build-properties ≥ 56.0.10** (#45922, merged 2026-05-18, first published 2026-05-19). Same key, same semantics in both SDKs (`@default false`, `@experimental`, `EXPO_USE_ANDROID_PRECOMPILED_HEADERS=1`). SDK 57 only added it to the docs page; there is no behavioural change.
- **`Linking.clearInitialURL()`** (Section 10) — needs **expo-linking ≥ 56.0.13** (#46265). Present in the published `expo-linking@56.0.16` `.d.ts`. The v56 docs-data JSON that omits it is a stale release-time snapshot.
- **`@expo/config-plugins` `Updates.getAppVersion(config, platform?)` / `getNativeVersion` honouring `ios.version` / `android.version`** for the `appVersion` runtime-version policy (#47416) — needs **@expo/config-plugins ≥ 56.0.12** (2026-07-07). The identical entry appears in both the sdk-56 (56.0.12) and sdk-57 (57.0.3) CHANGELOGs.
- **Docs-only corrections** to the `expo-build-properties` reference page: `useHermesV1` and `ios.usePrecompiledModules` print `true` from SDK 57 onward. Both already defaulted to `true` at runtime in SDK 56 — this reference states the corrected values inline (Section 5). Nothing to do.

### Provably unchanged in 57

- **App config schema (Section 2).** A flattened key diff between `origin/sdk-56:docs/public/static/schemas/v56.0.0/app-config-schema.json` and the SDK 57 schema on its own release branch (`origin/sdk-57:docs/public/static/schemas/unversioned/app-config-schema.json` — the sdk-57 branch has no `v57.0.0/` directory) shows **0 keys added, 0 removed**. Every field in Section 2 is valid in both SDKs.
- **expo-system-ui (7), expo-localization (11).** `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-system-ui/src packages/expo-localization/src packages/expo-localization/plugin/src` is **empty**. Identical source, including the four config-plugin props.
- **expo-constants (6).** The only diff in `packages/expo-constants/src` is two TSDoc prose strings ("bare workflow" → "existing React Native project"). No API change.
- **expo-splash-screen (8).** Plugin props and API (`hide`, `hideAsync`, `preventAutoHideAsync`, `setOptions`, `SplashScreenOptions`) unchanged; the only source diff is an internal signature tightening in `plugin/src/InterfaceBuilder.ts` (two optional params made required at a private call site).
- **expo-linking (10).** The only diff in `packages/expo-linking/src` is one TSDoc prose string in `Schemes.ts`. Every method and hook in Section 10 behaves identically in both SDKs.

### Not in 57 — do not attribute

- **iOS UIKit scene-based life cycle** (`SceneDelegate.swift`, `ExpoAppSceneDelegate`, `Info.plist` → `UIApplicationSceneManifest`) — adopted on `main` 2026-06-10 (#46734) but **never merged into the SDK 57 release branch**: `git merge-base --is-ancestor d5395300b4d origin/sdk-57` is false. On `origin/sdk-57` the bare template has no `SceneDelegate.swift`, `packages/expo/ios/AppDelegates/` has no `ExpoAppSceneDelegate.swift`, the `Info.plist` has zero `UIApplicationSceneManifest` entries, and `AppDelegate: ExpoAppDelegate` still creates `window = UIWindow(frame: UIScreen.main.bounds)`, calls `factory.startReactNative(...)` and keeps both `RCTLinkingManager` overrides. Unreleased `main` / SDK 58 material — **AppDelegate-patching config plugins keep working unchanged in SDK 57**.
- The `expo-localization` RTL rework — #48086 (iOS RTL no longer force-enabled from device locale, follows `I18nManager`), #48080 (`android:supportsRtl` written from the `supportsRTL` prop), #48092 (`resourceConfigurations` deduplication on repeat `prebuild --no-clean`) all merged 2026-07-23, **after** 57.0.0 shipped. None of the three PR numbers appears in the `expo-localization` CHANGELOG on `origin/sdk-56` or `origin/sdk-57`, and `packages/expo-localization/plugin/src` is byte-identical across the two branches. Landed after the 57 cut.
- The `@expo/schemer` ajv → `@expo/schema-utils` swap that changes app-config validation error text (#47340) — merged on `main` but not on either release branch; `origin/sdk-57:packages/@expo/schemer/package.json` still depends on `ajv ^8.1.0` / `ajv-formats ^2.0.2`. Landed after the 57 cut.
- **Node `engines` bump** (`^22.13.0 || ^24.3.0 || …`) — not in SDK 57. `origin/sdk-57:packages/expo/package.json` has no `engines` field at all; Node ≥ 20.19.4 still works. Landed after the 57 cut.
- **Android `proguard-android-optimize.txt` / R8-on-by-default** — not in SDK 57. `origin/sdk-57:templates/expo-template-bare-minimum/android/app/build.gradle` still uses `proguard-android.txt`, and `android.enableMinifyInReleaseBuilds` (Section 5) remains opt-in with no default. Landed after the 57 cut.

### Version pins (56 → 57)

From `packages/expo/bundledNativeModules.json` on the release branches — the file that ships inside the `expo` package and that `expo install` actually resolves against. Cross-checked against the published `expo@56.0.17` and `expo@57.0.8` tarballs, which match exactly.

Do **not** read pins from `docs/public/static/schemas/v5x.0.0/native-modules.json`; those are release-time snapshots and are stale (they still say `expo-linking ~56.0.13`, `expo-font ~56.0.5`, `react-native-screens 4.25.2`).

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-build-properties` | `~56.0.24` | `~57.0.7` |
| `expo-constants` | `~56.0.22` | `~57.0.7` |
| `expo-linking` | `~56.0.16` | `~57.0.4` |
| `expo-splash-screen` | `~56.0.14` | `~57.0.5` |
| `expo-font` | `~56.0.7` | `~57.0.1` |
| `expo-system-ui` | `~56.0.5` | `~57.0.1` |
| `expo-localization` | `~56.0.6` | `~57.0.1` |
| `react-native` | `0.85.3` | `0.86.0` |

Note the shape: first-party `expo-*` packages are **not** a flat `~57.0.0` — each has its own patch line. The SDK 56 column also keeps climbing (SDK 56 is still actively patched), so treat these as "as of `expo@56.0.17` / `expo@57.0.8`" and re-read `bundledNativeModules.json` rather than assuming.

`react-native-screens` is `~4.26.0` in **both** SDKs — it did not move in 56 → 57, despite what the frozen schemas say.

The RN bump matters here only as context for `expo-build-properties` deployment targets — see references 01 and 17.
