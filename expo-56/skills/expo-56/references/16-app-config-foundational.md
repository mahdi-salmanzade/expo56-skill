# Expo SDK 56 — App Configuration, Icons/Splash & Foundational Packages

Knowledge base reference compiled from official Expo SDK 56 documentation. Captured 2026-05-22.
Domain: app configuration (app.json / app.config.js / app.config.ts), config plugins, app icons & splash, and foundational packages (expo-build-properties, expo-constants, expo-system-ui, expo-splash-screen, expo-font, expo-linking, expo-localization).

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
- TypeScript support (nullish coalescing, optional chaining).
- Re-evaluated when Metro reloads.
- **Cannot use Promises** — the final config cannot contain promises.

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
Use `tsx` to import TypeScript files and enable `import` syntax for local config plugins.

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
| `plugins` | array | Config plugins | — |
| `buildCacheProvider` | undefined | Remote build cache downloading | — |
| `experiments` | object | Unstable experimental features | — |
| `_internal` | object | Internal developer tool properties | — |

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

`onDemandFilesystem`, `autolinkingModuleResolution`, `baseUrl`, `supportsTVOnly`, `tsconfigPaths`, `typedRoutes`, `turboModules`, `reactCanary`, `reactCompiler`, `reactServerComponentRoutes`, `reactServerFunctions`, `inlineModules` (`{ watchedDirectories: array }`).

### `_internal`
- `pluginHistory` (object): plugins already run on config.

---

## 3. Config Plugins

Sources: https://docs.expo.dev/config-plugins/introduction/ , https://docs.expo.dev/workflow/configuration/

Config plugins extend Expo's default app config to modify native Android/iOS projects during the prebuild process (Continuous Native Generation / CNG) without editing native files directly. "A config plugin is a top-level custom configuration point that is not built into the app config."

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
| `usePrecompiledModules` | boolean | `false` | Use precompiled Expo modules (XCFrameworks) |

### Shared (Android & iOS) — SDK 56 highlights

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `buildReactNativeFromSource` | boolean | `false` | Build RN from source; on iOS controls precompiled xcframework usage |
| `reactNativeReleaseLevel` | `'stable'` \| `'canary'` \| `'experimental'` | `'stable'` | RN release level for feature flags |
| `useHermesV1` | boolean | `false` | Enable experimental Hermes V1 engine (requires `buildReactNativeFromSource` on RN 0.83) |

> **Note on `usePrecompiledHeaders`:** the SDK 56 build-properties reference page did not surface a key by that exact name. The closest precompilation-related keys present are iOS `usePrecompiledModules` (precompiled Expo modules / XCFrameworks) and the shared `buildReactNativeFromSource` (which controls iOS precompiled xcframework usage). Verify against the live page if a literal `usePrecompiledHeaders` key is required.

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
- **Session & runtime:** `sessionId` (unique per app session), `executionEnvironment`, `debugMode` (`__DEV__`), `isHeadless`.
- **Version:** `expoVersion` (null in bare/web), `expoRuntimeVersion` (nullable on web).

### Methods
- `getWebViewUserAgentAsync()` → `Promise<string | null>`.

### Enums
- `ExecutionEnvironment`: `Bare`, `Standalone`, `StoreClient`.
- `UserInterfaceIdiom`: `Handset`, `Tablet`, `Desktop`, `TV`, `Unsupported`.

### SDK 56 notes
- `appOwnership` deprecated in favor of `executionEnvironment`.
- iOS properties `model`, `platform`, `systemVersion`, `userInterfaceIdiom`, and `deviceYearClass` moved to **expo-device**.

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

Config plugin (see also Section 4 for examples):
```json
{
  "expo": {
    "plugins": [
      [
        "expo-splash-screen",
        {
          "backgroundColor": "#232323",
          "image": "./assets/splash-icon.png",
          "dark": {
            "image": "./assets/splash-icon-dark.png",
            "backgroundColor": "#000000"
          },
          "imageWidth": 200
        }
      ]
    ]
  }
}
```

### Plugin properties

| Property | Default | Purpose |
|----------|---------|---------|
| `backgroundColor` | `#ffffff` | Hex color for splash background |
| `image` | `undefined` | Path to app icon/logo image |
| `imageWidth` | `100` | Image display width |
| `resizeMode` | `undefined` | `contain`, `cover`, or `native` |
| `dark` | `undefined` | Dark-mode configuration object |
| `enableFullScreenImage_legacy` | `false` | iOS only; legacy full-screen image support |
| `android` | `undefined` | Android-specific settings |
| `ios` | `undefined` | iOS-specific settings |

### API methods
- `preventAutoHideAsync()` — keep splash visible until manually hidden. `Promise<boolean>`.
- `hide()` — immediately hide splash. `void`.
- `hideAsync()` — async variant (backwards compatibility). `Promise<void>`.
- `setOptions(options)` — animation behavior; `duration` (ms) and `fade` (iOS only, boolean).

### SDK 56 note
Since SDK 52, due to the latest Android splash screen API changes, Expo Go and development builds cannot fully replicate the standalone-app splash experience.

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
- `fonts` — array of font file paths (relative to project root).
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
- `isLoaded(fontFamily)` — synchronous boolean.
- `getLoadedFonts()` — all loaded fonts (incl. build-time embedded); string array usable as `fontFamily`.

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
- `getInitialURL()` — URL that launched the app; `Promise<string | null>`.
- `openSettings()` — opens OS settings for the app.

### Hooks
- `useURL()` and `useLinkingURL()` (deprecated) — both return `string | null` (initial URL + subsequent changes).

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

Config plugin:
```json
{
  "expo": {
    "plugins": ["expo-localization"]
  }
}
```
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
