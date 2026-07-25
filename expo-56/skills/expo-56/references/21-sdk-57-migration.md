# Expo SDK 56 → 57 Migration

> Knowledge-base reference compiled from the **release branches** `origin/sdk-56` and `origin/sdk-57`
> of `expo/expo`, cross-checked against published npm tarballs (2026-07-25).
> Domain: taking a working SDK 56 app to SDK 57, and debugging the fallout.
> `expo@57.0.0` published **2026-06-30** (root `CHANGELOG.md` dates the SDK 57 release 2026-07-08);
> latest patch **57.0.8** (2026-07-22, dist-tag `latest`).
> **SDK 56 is still actively patched** — `expo@56.0.17` shipped **2026-07-23**, one day *after* 57.0.8.
> Support window is SDK 54–57.
>
> **Provenance rules.** Version pins come *only* from
> `git show origin/sdk-5{6,7}:packages/expo/bundledNativeModules.json` — that file ships inside the
> `expo` package and is what `expo install` resolves against. The frozen docs schemas
> (`docs/public/static/schemas/v5{6,7}.0.0/native-modules.json`) are **stale and must not be used**;
> they disagree with the shipped pins (e.g. they say `react-native-screens 4.25.2` and
> `expo-router ~56.2.9`; the shipped values are `~4.26.0` and `~56.2.16`).
> Behaviour claims come from `packages/*/CHANGELOG.md` **on the release branches**, from source on
> those branches, and from published tarballs. The monorepo's `main` branch is **SDK 58 in progress**
> and is never evidence for SDK 57; neither is any `## Unpublished` changelog section.

| Claim | Truth |
|-------|-------|
| SDK 57 raises the Node floor | **No.** `expo@57.0.8` ships **no `engines` field at all**. Node ≥ 20.19.4 still works (§1) |
| `npx expo prebuild` is additive | Only through SDK 56. **57 cleans by default**; `--no-clean` restores it (§2, §6) |
| React was bumped with RN 0.86 | No. React stays **19.2.3**, TypeScript stays **~6.0.3** (§1) |
| SDK 57 moves iOS to the UIKit scene lifecycle | **No.** No `SceneDelegate`, no `UIApplicationSceneManifest` in the 57 templates. `AppDelegate` still owns `RCTLinkingManager`. Expect it after 57 (§5) |
| `react-native-screens` moved in 57 | No. **`~4.26.0` on both** SDK 56 and 57 (§4) |
| Everything under "new in 57" needs an upgrade to get | **No.** Most of it was backported into the SDK 56 patch line. Check §3b before planning an upgrade around a feature |
| OTA updates keep matching after upgrade | Not guaranteed. `fingerprint` and `sdkVersion` policies always change (§7) |

---

## 1. Before you start — environment gates

| Gate | SDK 56 | SDK 57 | Notes |
|------|--------|--------|-------|
| **Node.js** | `>=20.19.4` | **unchanged** | Neither `expo@56` nor `expo@57` declares `engines`. Verified: `npm view expo@57.0.8 engines` → empty. |
| React Native | 0.85.3 | **0.86.0** | `bundledNativeModules.json`. |
| React / react-dom | 19.2.3 | **19.2.3 (unchanged)** | Do not bump. |
| TypeScript | ~6.0.3 | **~6.0.3 (unchanged)** | `@types/react` stays `~19.2.2`. |
| iOS / tvOS minimum | 16.4 | **16.4 (unchanged)** | macOS 13.4 unchanged. `ExpoModulesCore.podspec:53-56` is byte-identical on both branches. |
| Xcode (EAS image) | 26.4 | **26.6** | Alias `macos-tahoe-26.4-xcode-26.4` (`sdk-56`) → `macos-tahoe-26.5-xcode-26.6` (`latest`, `sdk-57`). |
| EAS Android image | `ubuntu-26.04-jdk-17-ndk-r27b` (`sdk-56`) | `ubuntu-26.04-jdk-17-ndk-r27b-sdk-57` | Same JDK 17 / NDK r27b. |
| EAS image Node | 22.22.2 | 22.23.1 | Both images already ship Node 22; Node is not a differentiator. |
| JS engine | Hermes V1, New Arch only | unchanged | |

Sources: `git show origin/sdk-5{6,7}:packages/expo/package.json` (no `engines`);
`packages/expo-modules-core/ExpoModulesCore.podspec`; `bundledNativeModules.json`;
EAS image table from the **live** `docs/pages/build-reference/infrastructure.mdx` (that page is a
service-status page, not SDK-frozen content — the copy on the `sdk-57` branch is stale and stops at
`sdk-55`).

> **There is no Node gate.** A previous version of this guide claimed SDK 57 raised `engines.node` to
> `^22.13.0 || ^24.3.0 || …`. That is **SDK 58 work on `main`**; it is not in any shipped 57 package.
> Do not block or rewrite CI on it. (Raising Node anyway is harmless — SDK 56 runs fine on 22.x.)

---

## 2. The upgrade command sequence

```sh
# 1. Commit / branch. §6 rewrites ios/ and android/.
git status --porcelain     # must be clean

# 2. Bump the SDK package.
npm install expo@^57.0.0        # or: yarn add / pnpm add / bun install

# 3. Align every other dependency to the SDK 57 pin set.
npx expo install --fix

# 4. Diagnose.
npx expo-doctor@latest

# 5. Regenerate native projects (CNG projects).
rm -rf ios android
npx expo prebuild --clean

# 6. Rebuild. Dev clients MUST be rebuilt — the fingerprint changed (§7).
npx expo run:ios
npx expo run:android
```

| Step | What to check after |
|------|---------------------|
| 2 | `expo` resolves to `57.x`; lockfile updated. |
| 3 | Diff the lockfile against §4. `react` must still be `19.2.3`; `react-native-screens` must still be `~4.26.0`. |
| 4 | expo-doctor should be clean, or every remaining finding understood. |
| 5 | `ios/` and `android/` regenerate; every config-plugin edit is still present (§6). |
| 6 | OTA runtime version (§7), Android release build, camera output (§9). |

There is **no 56→57 codemod**; the only codemod Expo ships is
`sdk-56-expo-router-react-navigation-replace`, an SDK 56 concern. For bare (non-CNG) projects,
replace step 5 with `npx pod-install`. **No manual iOS AppDelegate work is required for 57** — the
generated `AppDelegate.swift` is unchanged between 56 and 57 (§5).

---

## 3. Breaking changes that are genuinely new in SDK 57

The root `CHANGELOG.md` `## 57.0.0` "🛠 Breaking changes" section on `origin/sdk-57` contains
**exactly four entries**. They are the first four rows below. The rest of this table is verified
package-by-package against both release branches.

| What changed | What to do |
|--------------|------------|
| **`@expo/cli`: `npx expo prebuild` clears and regenerates native folders by default.** Implementation: `clean: !args['--no-clean']` (`packages/@expo/cli/src/prebuild/index.ts:67`); SDK 56 has `clean: args['--clean']`. (#47209, `@expo/cli` 57.0.0) | Pass `--no-clean` to keep the SDK 56 additive behaviour, or move hand edits into config plugins. See §6. |
| **`expo-font` [web]: `Server.resetServerContext()` removed** (#46669, `expo-font` 57.0.0). Server font state is now scoped per render via `AsyncLocalStorage`. Verified: `src/server.ts` exports `resetServerContext` on `sdk-56` and `withServerContext` on `sdk-57`. | Replace with `Server.withServerContext(callback)`. `@expo/router-server` already wraps `getStaticContent`/`getStreamingContent`. Font APIs called outside the wrapper throw. |
| **`expo-modules-jsi` [iOS]: `JavaScriptError` is now a copyable class conforming to `Error`** (was a non-copyable struct), and **`JavaScriptValue` no longer conforms to `Error`** (#47154, `expo-modules-jsi` 57.0.0). | Swift code that caught a `JavaScriptValue` as an `Error`, or moved `JavaScriptError` as a non-copyable value, must be rewritten. New `JavaScriptError.init(_:value:)` wraps an arbitrary JS value while preserving its identity. |
| **`@expo/ui` [universal][android]: Compose `BasicTextField` instead of Filled Material `TextField`** (#46442). | **Also on the SDK 56 line — see §3b.** Listed here only because it is in the root 57.0.0 breaking-changes section. |
| **`expo-camera` [iOS]: default `pictureSize` `high` → `photo`** (#47173, `expo-camera` 57.0.1). Verified in source: `ios/CameraViewModule.swift` falls back to `.high` on `sdk-56` and `.photo` on `sdk-57`. Preview now hosted on the view's backing layer; `videoOrientation` replaced with `AVCaptureDevice.RotationCoordinator` (#47172). | Set `pictureSize="high"` explicitly to keep the old preset. Otherwise expect **larger images and different EXIF dimensions**. See §9. |
| **`expo-app-metrics`: development-only `triggerCrash` and `simulateCrashReport` removed** (#46924, 57.0.0). Verified: present in `src/` on `sdk-56`, absent on `sdk-57`. | Delete usage. No replacement. |
| **`expo-observe`: legacy non-OpenTelemetry dispatch path removed**; metrics and logs are always OTLP (#47030, 57.0.0). | Self-hosted / custom collectors must accept OTLP. |
| **`@expo/cli`: DevTools plugin WebSocket handlers receive a fetch `Request`** instead of a Node `IncomingMessage` (#47410, `@expo/cli` 57.0.5). | Custom DevTools plugins must read `request.url` / `request.headers` off a `Request`. |
| **`expo-modules-core` peer range for `react-native-worklets` widened** to `^0.7.4 \|\| ^0.8.0 \|\| ^0.9.0 \|\| ^0.10.0` (SDK 56: `^0.7.4 \|\| ^0.8.0`). | Consequence of the worklets `0.8.3 → 0.10.0` pin bump (§4). On SDK 56 you cannot install worklets 0.10 without a peer warning; on 57 you must. |

### Verified **not** breaking in 57 — do not act on these

These were asserted in an earlier revision of this guide and are **false for SDK 57**. Each was
checked against `origin/sdk-56` and `origin/sdk-57` source, not changelogs.

| Claimed change | Reality |
|----------------|---------|
| iOS UIKit scene lifecycle / `SceneDelegate` / `UIApplicationSceneManifest` | Not in 57. See §5. |
| `engines.node` raised to `^22.13.0 \|\| …` | No `engines` field in `expo@57.0.8`. See §1. |
| Android release builds switch to `proguard-android-optimize.txt` (R8 optimization on) | `templates/expo-template-bare-minimum/android/app/build.gradle:119` still reads `proguard-android.txt` on `origin/sdk-57`. See §8. |
| `expo-video` [iOS] default `audioMixingMode` `doNotMix` → `auto` | `ios/VideoPlayer.swift` declares `var audioMixingMode: AudioMixingMode = .doNotMix` on **both** branches. |
| `expo-cellular` `allowsVoipAsync` removed on Android | Still exported from `src/Cellular.ts` and implemented in `CellularModule.kt` on `sdk-57`. |
| `expo-modules-core` removed `AppContext.hostingRuntimeContext` / `AppContext.errorManager` | Both still declared in `AppContext.kt:58,271` on `sdk-57`. |
| `expo-modules-core` `ArrayBuffer` interface replaced with a class | `android/…/jni/ArrayBuffer.kt:9` still reads `interface ArrayBuffer` on `sdk-57`. `ArrayBuffer.withJSBytes` does not exist on either branch. |
| `@expo/schemer` dropped `validateProperty`/`validateName`/… and `ajv` | All five methods still at `src/index.ts:314-336`; `ajv`, `ajv-formats`, `json-schema-traverse` still in `dependencies` on `sdk-57`. |
| `expo login` defaults to browser login | `packages/@expo/cli/src/login/index.ts` is identical on both branches; `--browser` is still opt-in. |
| `expo-updates` [iOS] stops copying/hashing embedded assets on first launch | The whole `packages/expo-updates/ios` diff between the branches is logging-only. |
| `@expo/fingerprint` `SourceSkips.ExpoConfigVersions` now strips `ios.version`/`android.version` | `packages/@expo/fingerprint/src` is **byte-identical** between `sdk-56` (0.19.9) and `sdk-57` (0.20.6). See §7. |
| `MainActivity` `android:configChanges` gained `assetsPaths` | Present in the SDK **56** manifest template too. |
| `jest-expo` realigned Babel options via `resolveBabelOptions` | The only `jest-expo/src` diff between branches is `preset/moduleMocks/expoModules.js`. |
| `expo` dropped legacy-arch `RCTRootViewFactoryConfiguration` setup | Still referenced in `packages/expo/ios/AppDelegates/` on `sdk-57`. |
| `expo-network` removed the legacy `fetchNetworkState` path | No occurrence of `fetchNetworkState` in `src/` on **either** branch — nothing changed in 57. |

---

## 3b. Not a 57 delta — already available on the SDK 56 patch line

SDK 56 is still receiving features, not just fixes. Every item below is in **both** release
branches' changelogs. **Upgrading to 57 to get any of these is wasted work** — bump the package on
the 56 line instead. Minimum versions are the package's own version, resolvable with
`npx expo install <pkg>` on SDK 56 (`expo@56.0.17` pins the latest of each in its
`bundledNativeModules.json`).

| Change | Minimum on the SDK 56 line |
|--------|---------------------------|
| **`@expo/config-plugins`: `Updates.getAppVersion`, `Updates.getNativeVersion` and the `appVersion` runtime-version policy honour `ios.version`/`android.version`** (#47416). `src/utils/Updates.ts` is **byte-identical** on both branches. This is the OTA-affecting one — see §7. | `@expo/config-plugins` **56.0.12** (check with `npm ls @expo/config-plugins`) |
| `@expo/ui` [universal][android] Compose `BasicTextField` replaces Filled Material `TextField` (#46442) | `@expo/ui` **56.0.17** |
| `expo-widgets`: Android config plugin gated behind `enableAndroid` (#46463) | `expo-widgets` **56.0.17** |
| `expo-router` [Android]: navigation state restoration across activity recreation removed (#47422) | `expo-router` **56.2.14** |
| `expo-router`: `standard-navigation` integration, native tabs rewrite (#46456); `unstable_createStandardRouterNavigator` type inference fix (#46737) | `expo-router` **56.2.10** / **56.2.12** |
| `expo-router`: `unstable_nativeProps` on native Stack options (#47482); `Theme` exported from the root entry (#47476); `expo-router/drawer` re-exports (#46635) | `expo-router` **56.2.14** / **56.2.15** / **56.2.10** |
| `expo-modules-core` + `@expo/ui`: worklet UI runtime resolved without `react-native-reanimated`, via `getUIRuntimeHolder()` (#46922, #46935); actionable error when the worklets native adapter isn't linked (#46571) | `expo-modules-core` **56.0.18** / `@expo/ui` **56.0.19**; #46571 at `expo-modules-core` **56.0.15** |
| iOS Podfile: explicit `usePrecompiledModules: false` wins over a preset env var (#46983). The generated `Podfile` is **identical** on both branches, including the Hermes-V1 gating line. | `expo-modules-autolinking` **56.0.17** |
| `@expo/cli`: `expo customize metro.config.js` no longer installs `@expo/metro-config` (#46600) | `@expo/cli` **56.1.14** |
| `@expo/cli`: Device Hub as a Simulator replacement for Xcode 27+ (#46757) | `@expo/cli` **56.1.16** |
| `expo-doctor`: honours the RN Directory `new-arch-only` status so New-Arch-only libs aren't flagged (#46755) | `expo-doctor` **1.19.10** (independent version line; `npx expo-doctor@latest` always gets it) |
| `expo-linking`: `Linking.clearInitialURL()` (#46265) | `expo-linking` **56.0.13** |
| `expo-modules-core` [iOS]: `@Record` macro (#46547) | `expo-modules-core` **56.0.16** |
| `@expo/require-utils` (#47441) | **56.1.4** |
| `expo-media-library`: `isFavorite` filter, `Query.exeForMetadata()`, `getAssetContentUriAsync` | present in `src/` on both branches |
| `expo-contacts` [iOS]: `cancelButtonTitle` / `showsCancelButton` / `preventAnimation` on `presentCreateForm` (#46960) | present in `src/` on both branches |
| `@expo/ui` SwiftUI modifier additions (`accessibilityHidden`, `redacted`/`privacySensitive`, `seedColor`, `useNativeState().get()/.set()`, …) | present in `src/` on both branches |
| `babel-preset-expo`: `EXPO_PUBLIC_USE_RN_FETCH` inlined inside `node_modules` (#46986); per-platform preset overrides | `babel-preset-expo` **56.0.16** |
| `expo/fetch` correctness fixes (gzip/br/zstd on Android, `bodyUsed` on double-clone, empty body for body-less POST, `Response.blob()`) | present in `packages/expo/src/winter/fetch` on both branches |

---

## 4. Dependency pin diff (56 → 57)

Authoritative source: `git show origin/sdk-5{6,7}:packages/expo/bundledNativeModules.json`.
v56 lists 122 packages, v57 lists 123.

### Third-party pins that CHANGED — exactly seven

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `react-native` | `0.85.3` | **`0.86.0`** |
| `react-native-reanimated` | `4.3.1` | **`4.5.0`** |
| `react-native-worklets` | `0.8.3` | **`0.10.0`** |
| `react-native-gesture-handler` | `~2.31.1` | **`~2.32.0`** |
| `react-native-keyboard-controller` | `1.21.6` | **`1.21.9`** |
| `react-native-pager-view` | `8.0.1` | **`8.0.2`** |
| `lottie-react-native` | `~7.3.4` | **`~7.3.8`** |

`react-native-worklets` 0.8 → 0.10 is two minor releases and the largest third-party jump; it is
also what forces the widened `expo-modules-core` peer range (§3). Autolinking gained the worklets
`ENABLE_CROSS_RUNTIME_STACK_TRACES` flag on the 57 line only
(`packages/expo-modules-autolinking/external-configs/ios/react-native-worklets/spm.config.json`).

### Third-party pins that did NOT change — do not churn these

`react` `19.2.3` · `react-dom` `19.2.3` · **`react-native-screens` `~4.26.0`** (a stale docs schema
says `4.25.2`; it is wrong) · `react-native-safe-area-context` `~5.7.0` · `react-native-svg`
`15.15.4` · `react-native-webview` `13.16.1` · `@shopify/react-native-skia` `2.6.2` ·
`@shopify/flash-list` `2.0.2` · `react-native-web` `~0.21.0` · `react-native-maps` `1.27.2` ·
`@expo/vector-icons` `^15.0.2` · `@react-native-async-storage/async-storage` `2.2.0`.
`typescript` (`~6.0.3`) and `@types/react` (`~19.2.2`) are not in `bundledNativeModules.json` at
all — they come from the default template's devDependencies, unchanged.

### First-party pins are NOT flat `~57.0.0`

Every `expo-*` / `@expo/*` package moved to its own `57.0.x`. Notable, because the patch component
matters when you pin manually. Values below are read from the **published tarballs**
(`npm pack expo@56.0.17` / `expo@57.0.8` → `package/bundledNativeModules.json`), which is what
`expo install` actually resolves against:

| Package | 56 (`expo@56.0.17`) | 57 (`expo@57.0.8`) |
|---------|---------------------|--------------------|
| `expo` | `~56.0.17` | **`~57.0.8`** |
| `expo-router` | `~56.2.16` | **`~57.0.8`** |
| `expo-video` | `~56.1.4` | **`~57.0.2`** |
| `expo-dev-client` | `~56.0.24` | **`~57.0.8`** |
| `expo-updates` | `~56.0.23` | **`~57.0.9`** |
| `expo-modules-core` | `~56.0.22` | **`~57.0.7`** |
| `@expo/ui` | `~56.0.23` | **`~57.0.7`** |
| `@expo/fingerprint` | `~0.19.9` | **`~0.20.6`** (independent version line) |

> The published `expo@56.0.17` tarball's `bundledNativeModules.json` is byte-identical to
> `origin/sdk-56` HEAD. The `expo@57.0.8` tarball is *behind* `origin/sdk-57` HEAD on two entries —
> the branch already has `expo-updates ~57.0.10` and `expo-dev-client ~57.0.9` queued for the next
> `expo` patch. Take the tarball values as ground truth for what `expo install` gives you today.

**Newly listed in 57:** `expo-eas-client` at `~57.0.1` (absent from the SDK 56 list).
**Nothing was removed.** `@expo/dom-webview` is in **both** lists (`~56.0.6` → `~57.0.1`) — an
earlier revision of this guide claimed it was new in 57; it is not.

> **Template drift warning.** `templates/expo-template-default/package.json` on `origin/sdk-57`
> agrees with `bundledNativeModules.json` exactly. The same template on **`main`** has drifted to
> post-57 pins (`main` is SDK 58 in progress) — do not read it for a 57 upgrade.
> `npx expo install --fix` resolves against `bundledNativeModules.json` shipped inside your installed
> `expo` package. Trust that file.

---

## 5. iOS: what did **not** change

**SDK 57 does not adopt the UIKit scene lifecycle.** Verified on `origin/sdk-57`:

- `git grep -l SceneDelegate origin/sdk-57 -- templates/` → **nothing**. There is no
  `templates/expo-template-bare-minimum/ios/HelloWorld/SceneDelegate.swift`.
- The bare template's `Info.plist` contains **no** `UIApplicationSceneManifest` key.
- `templates/…/ios/HelloWorld/AppDelegate.swift` on `sdk-57` is unchanged from `sdk-56`: it is still
  `class AppDelegate: ExpoAppDelegate`, it still creates
  `window = UIWindow(frame: UIScreen.main.bounds)` and calls
  `factory.startReactNative(withModuleName:in:launchOptions:)`, and it still overrides
  `application(_:open:options:)` and `application(_:continue:restorationHandler:)` with
  `RCTLinkingManager`.
- There is no `ExpoAppSceneDelegate` and no `ExpoReactNativeFactoryProvider` protocol in
  `packages/expo/ios/AppDelegates/` on `sdk-57`.

Practical consequences for the 56 → 57 upgrade:

- **Config plugins that patch `AppDelegate.swift`** with `withAppDelegate`/`mergeContents` and anchor
  on `application(_:open:options:)` or `application(_:continue:restorationHandler:)` **keep working**.
  Nothing to retarget.
- `ExpoAppDelegateSubscriber`-based plugins keep working (they always did).
- No manual Xcode target / `project.pbxproj` work is needed for bare projects.

> **Not in 57; landed after the 57 cut.** The scene-lifecycle adoption (`SceneDelegate.swift`,
> `ExpoAppSceneDelegate`, `UIApplicationSceneManifest`, and the removal of the AppDelegate
> `RCTLinkingManager` overrides) exists only on `main`, i.e. SDK 58 in progress. It is easy to
> re-introduce into notes because it is prominent on `main` — it is **not** part of this upgrade.
> When it does ship, source-patching plugins that anchor on the AppDelegate linking overrides will
> break; nothing else in the list above changes.

---

## 6. prebuild semantics changed

| SDK 56 | SDK 57 |
|--------|--------|
| `npx expo prebuild` **applies changes on top of** existing `ios/` and `android/` | `npx expo prebuild` **clears and regenerates** them |
| `--clean` opted into wiping | `--no-clean` opts back into the additive behaviour |

Source: root `CHANGELOG.md` `## 57.0.0` → 🛠 Breaking changes (#47209);
`packages/@expo/cli/src/prebuild/index.ts:13` (`'--no-clean': Boolean`), `:67`
(`clean: !args['--no-clean']`), `:37` (help text: *"Apply changes to the existing native folders
instead of recreating them"*). Confirmed absent from `origin/sdk-56`, where `:66` reads
`clean: args['--clean']`.

**Why it matters.** Any project that ran `prebuild` and then hand-edited `ios/` or `android/` — a
custom entitlement, an extra Gradle dependency, a tweaked `AndroidManifest.xml`, a patched
`AppDelegate` — now loses those edits on the next `prebuild`, `run:ios`, `run:android` or EAS Build
that triggers prebuild. There is no prompt and no diff; the folders are simply regenerated.

**Do not reach for `--no-clean` as the fix** — it preserves the old footgun. In order of preference:
(1) move the edit into a config plugin (`withInfoPlist`, `withAndroidManifest`, `withAppBuildGradle`,
`withEntitlementsPlist`, `withXcodeProject`, `withAppDelegate`); (2) if it cannot be expressed as a
plugin, commit `ios/`+`android/` and stop running `prebuild` entirely (fully bare workflow);
(3) use `--no-clean` only as a temporary bridge. For the upgrade itself use the clean path:
```sh
rm -rf ios android && npx expo prebuild --clean
```

---

## 7. Runtime version & fingerprint

This section breaks **published updates**, not builds. It fails without an error message: existing
installs simply stop receiving updates because their embedded runtime version no longer matches what
the server now computes.

### Read this first — the `appVersion` change is *not* a 57 delta

`@expo/config-plugins` #47416 (`Updates.getAppVersion`, `Updates.getNativeVersion` and the
`appVersion` runtime-version policy honouring `ios.version` / `android.version`) landed on **both**
lines. `packages/@expo/config-plugins/src/utils/Updates.ts` is byte-identical on `origin/sdk-56` and
`origin/sdk-57`. It shipped in `@expo/config-plugins` **56.0.12** (2026-07-07) — the same day as **57.0.3**.

So: **if you are on a recent SDK 56 patch, this already happened to you.** Verbatim from the SDK 56
changelog:

> Honor `ios.version` and `android.version` in `Updates.getAppVersion`, `Updates.getNativeVersion`,
> and the `appVersion` runtime version policy. Previously the platform-specific overrides were
> ignored, so projects that used only `ios.version`/`android.version` (with no top-level `version` in
> `app.json`) received the `package.json` fallback (or `"1.0.0"`). `Updates.getAppVersion` gains an
> optional `platform` argument […] Also fixes `Updates.getNativeVersion` on Android, which previously
> used the iOS version for the `${version}` component.

Run `npm ls @expo/config-plugins` to see which side of 56.0.12 you are on. If you are already ≥
56.0.12, the 56 → 57 upgrade changes nothing here.

### `@expo/fingerprint` `~0.19.9` → `~0.20.6`

The version line moves, but **`packages/@expo/fingerprint/src` is byte-identical between the two
release branches** — 0.20.0 through 0.20.6 are all *"This version does not introduce any user-facing
changes"* except #47503 (more default `getConfig` exclusions), which is on the 56 line too.

Your fingerprint **will** still change on upgrade — but because the *inputs* changed (RN 0.86, new
first-party package versions, new native module sources), not because the algorithm changed. There
is no `SourceSkips` semantics change in 57.

### Who is affected

| Runtime version policy | Affected by 56 → 57? |
|------------------------|----------------------|
| `"runtimeVersion": "1.0.0"` (literal string) | No |
| `{"policy": "appVersion"}` | Only if you are crossing `@expo/config-plugins` 56.0.12 at the same time — an SDK 56 patch concern, not a 57 one |
| `{"policy": "nativeVersion"}` | Same as above |
| `{"policy": "fingerprint"}` | **Yes** — RN 0.86 and new package versions are hashed inputs |
| `{"policy": "sdkVersion"}` | Yes, trivially — `56.0.0` → `57.0.0` |

The `fingerprint` and `sdkVersion` policies always change, and both change *loudly*: dev clients
report a mismatch. The quiet failure mode is the `appVersion` one, and it is triggered by the
config-plugins patch bump, so it can bite you without ever touching SDK 57.

### Check-and-republish procedure

```sh
# 1. Record the CURRENT runtime version, on SDK 56, before touching anything.
npx expo config --type public | grep -i runtimeVersion
npx expo-updates runtimeversion:resolve --platform ios      > /tmp/rtv-ios-56.txt
npx expo-updates runtimeversion:resolve --platform android  > /tmp/rtv-android-56.txt

# 2. Upgrade (§2).

# 3. Recompute and diff.
npx expo-updates runtimeversion:resolve --platform ios      > /tmp/rtv-ios-57.txt
npx expo-updates runtimeversion:resolve --platform android  > /tmp/rtv-android-57.txt
diff /tmp/rtv-ios-56.txt /tmp/rtv-ios-57.txt
diff /tmp/rtv-android-56.txt /tmp/rtv-android-57.txt
```

**If the values differ:** (1) do **not** publish an SDK 57 update against the new runtime version and
expect old installs to get it — old binaries embed the old runtime version; (2) ship a **new native
build** carrying the new runtime version and publish SDK 57 updates against it; (3) keep the SDK 56
branch alive on the **old** runtime version (`eas update --branch <old-branch> --runtime-version
<old value>`) until adoption of the new build is acceptable; (4) or preserve the old value explicitly
— replace the `appVersion` policy with a literal `"runtimeVersion": "<old value>"`, or add the
matching top-level `version` to `app.json` — then re-verify with step 3 before publishing.

Also expect **EAS Build cache misses** on the first SDK 57 build, and **every existing dev client to
report a runtime mismatch** until rebuilt. Both are expected.

> Not in SDK 57: `@expo/fingerprint` presets (`strict`/`balanced`/`relaxed`), the `package` source
> type, and `react-native` name+version hashing are `main`-only (SDK 58 in progress). Do not
> configure `preset` on SDK 57.

---

## 8. Android changes

### R8 optimization is **not** turned on in 57

`templates/expo-template-bare-minimum/android/app/build.gradle:119` on `origin/sdk-57`:
```gradle
proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
```
Identical to `origin/sdk-56`. `packages/expo-modules-core/android/proguard-rules.pro` is also
byte-identical between the branches. **There is no ProGuard/R8 work in this upgrade.**

> Not in 57; landed after the 57 cut: the switch to
> `getDefaultProguardFile("proguard-android-optimize.txt")` is `main`-only. When it ships, the
> failure mode will be reflection-based code (Gson/Moshi/Jackson models, `Class.forName`, JNI
> lookups by name, `@Serializable` classes) crashing **only in release builds** — worth knowing, but
> do not budget for it in a 56 → 57 upgrade.

### Actual Android-side changes in 57

| Change | Detail |
|--------|--------|
| **`android.cmakeVersion` build property** | New in `expo-build-properties` **57.0.2** (#47377) — absent from `origin/sdk-56`. Applied to the app **and every autolinked native-module subproject**. |
| Everything else | `AndroidManifest.xml` (`configChanges` including `assetsPaths`), `build.gradle`, and the `expo-modules-core` Kotlin API surface are unchanged between the two branches. |

```json
{ "plugins": [["expo-build-properties", { "android": { "cmakeVersion": "3.31.6" } }]] }
```

---

## 9. Default-value changes that silently alter behaviour

No error, no warning, no type change. In SDK 57 there is **one** of these, and it is verified in
source on both branches.

| Package | Default in 56 | Default in 57 | Symptom | Restore old behaviour |
|---------|---------------|---------------|---------|-----------------------|
| `expo-camera` [iOS] | `pictureSize: 'high'` | **`'photo'`** (#47173, `expo-camera` 57.0.1) | Larger captured images, different EXIF dimensions, uploads over size limits | `<CameraView pictureSize="high" />` |

Previously listed here and **removed as false** (see §3): `expo-video` `audioMixingMode`
(`.doNotMix` on both branches), the iOS Podfile precompiled-modules and Hermes-V1 gating changes
(the generated `Podfile` is identical on both branches — see §3b for the SDK 56 patch that
introduced them), and `expo-widgets` undefined-prop fallback (no such entry in either changelog).

`expo-widgets`' Android plugin being opt-in behind `enableAndroid` is real but is **also on the 56
line** (`expo-widgets` 56.0.17) — §3b.

---

## 10. New in SDK 57 worth adopting

Opportunity, not obligation. **Only 57-only items are listed here**; anything that also exists on the
SDK 56 patch line has been moved to §3b, because "upgrade to 57 to get X" is bad advice when X is one
`npx expo install` away.

| Package | What's new (verified absent from `origin/sdk-56`) |
|---------|--------------------------------------------------|
| `expo-image` | `Image.writeToCacheAsync(source, cacheKey)` / `Image.readFromCacheAsync(cacheKey)` — seed and read the image cache by key (#46620, `expo-image` 57.0.0). |
| `expo-app-metrics` | `NetworkRequestObserver` class + `useNetworkRequestObserver` hook, with native-side host/method filtering via `setFilter` (#46475, #46775); unhandled JS errors captured through `global.ErrorUtils` as OTel `exception` log events, fatals written synchronously and ingested next launch (#46923); Android crash reports (#46869); `expo.memory.warning` on iOS (#47108). All at `expo-app-metrics` 57.0.0. |
| `expo-observe` | `<ObserveInteractiveMarker>` (#46909, 57.0.0). OTLP-spec retry on both platforms (#47159, #47160, 57.0.5). |
| `@expo/cli` | DevTools plugins can declare a `serverEntryPoint` (HTTP + WebSocket inside the CLI process). |
| `expo-build-properties` | `android.cmakeVersion` (#47377, 57.0.2) — §8. |
| `expo-modules-core` [iOS] | `@Event` macro (function-typed `var` → typed JS event) and `@ExpoModule` / `@ExpoModule("CustomName")` synthesizing the JS name so `Name(…)` is no longer required (#46938); `@SharedObject` binding `@JS` members onto the prototype (#47107, 57.0.4); `Module.emit` (#46555) and `SharedObject.native(from:)` (#47054). All 57.0.0 unless noted. |
| `expo-modules-jsi` [iOS] | `UnownedThisSyncFunctionClosure` overloads for `createFunction`/`setProperty` (#46949); `JavaScriptRuntime: Identifiable` (#47068). Both 57.0.0. |

Note that several items the SDK 57 root changelog advertises as new were in fact **already on the
56 line** when 57 shipped — the root changelog aggregates everything since the previous *SDK*, not
since the previous *patch*. Verified examples, all present in `origin/sdk-56` changelogs:
`pod-install` Bundler-managed CocoaPods (#43605, `pod-install` 1.0.19), `expo-modules-autolinking`
inline-module targets (#46698, 56.0.16) and experimental `tvos`/`macos` autolinking (#46344,
56.0.17), `expo-brownfield` `hostProvidedFrameworks` (#46355, 56.0.18), `expo-network` macOS support
(#46535, 56.0.5), and `expo-modules-jsi` `JavaScriptUnownedValue` / closure-taking
`JavaScriptObject.setProperty(_:function:)` (#46616, #46622, 56.0.9). See §3b for the general rule.

**Also new in 57 — `expo-modules-jsi` 57.0.2 (iOS), both absent from the SDK 56 line:**

| API | What it does | Source |
|---|---|---|
| `JavaScriptRef.withValue { }` | Non-consuming borrow accessor — reads the referenced value without taking it, so a long-lived ref can be read repeatedly. | `Runtime/JavaScriptRef.swift:63` (#47238) |
| `JavaScriptRuntime.longLivedObjects` | A `LongLivedObjectCollection` keeping `LongLivedObject`s (e.g. in-flight promises) alive across async boundaries, releasing stragglers at runtime teardown. | `Runtime/JavaScriptRuntime.swift:684` (#47511) |

`JavaScriptCodable` also gained a `Date` conformance in 57.0.2 (#47602): encodes to a JS `Date`, decodes from a JS `Date`, epoch milliseconds, or a string the JS engine's `Date` constructor accepts.

> **Verified absent from `origin/sdk-57` — do not write code against these:**
> `CameraView.scanDocumentAsync` / `isDocumentScannerAvailable`, `File.preview()` / `File.canPreview()`,
> `ArrayBuffer.withJSBytes`, `expo-clipboard` `{ android: { isSensitive: true } }`,
> `<VideoView controllerAutoShow={false} />`, `<ObserveErrorBoundary>`,
> `<AppMetricsErrorBoundary>`, `expo-widgets` `staleDate` / `initialProps`,
> `useReleasingSharedObjectWithLifecycle`, and the `expo-dev-client`
> `launchModeExperimental` → `launchMode` rename (both keys exist unchanged on both branches).
> Note `JavaScriptValue.getAny()` **does** exist (`Values/JavaScriptValue.swift:185`) on both lines,
> but is deprecated; 57.0.1 (#47381) changed it to return `NSNull` for unrepresentable values.

### New packages

- **`expo-eas-client`** `~57.0.1` — the only genuinely new entry in the SDK 57
  `bundledNativeModules.json`.
- `@expo/dom-webview` is **not** new to 57 — it is in the SDK 56 `bundledNativeModules.json` at
  `~56.0.6` (→ `~57.0.1`).
- `@expo/require-utils` is **not** new to 57 either. The root 57.0.0 changelog says *"Initial release
  of `@expo/require-utils`"*, but the package also ships on the SDK 56 line (`56.1.6` on
  `origin/sdk-56`, with #47441 in its 56.1.4 changelog entry). It is a CLI/dev-server module-eval
  helper, not an app-installable native module, and is not in `bundledNativeModules.json` on either
  line.

---

## 11. Rollback

SDK 56 is still supported and still shipping (56.0.17, 2026-07-23, dist-tag `sdk-56`), so rolling
back is viable.

```sh
git checkout <pre-upgrade-commit> -- package.json package-lock.json  # or yarn.lock / pnpm-lock.yaml
rm -rf node_modules && npm ci
npx expo config --type public | grep sdkVersion   # expect 56.x
rm -rf ios android && npx expo prebuild --clean   # on the 56 CLI --clean is explicit, not default
# then rebuild dev clients and any internal-distribution builds
```

Pin the EAS image explicitly while rolled back — `latest` now points at SDK 57 images:
`{ "build": { "production": { "image": "sdk-56" } } }`.

### What does NOT roll back cleanly

| Item | Why | Mitigation |
|------|-----|------------|
| **Published-update runtime versions** | Updates published under an SDK 57 runtime version (§7) stay on that runtime version. Rolling the source back does not re-target them. | Keep the SDK 56 update branch alive on the old runtime version; do not repoint channels at the 57 branch. |
| **Store-released binaries** | An SDK 57 build already in the App Store / Play Store embeds the 57 runtime version and cannot be served SDK 56 updates. | Ship a corrective build; keep serving the 57 branch until adoption drops. |
| **Prebuilt native dirs** | If you committed `ios/`/`android/` and let SDK 57 regenerate them, your uncommitted hand edits are gone (§6). The *generated* iOS and Android files themselves are effectively identical between 56 and 57, so the loss is your own patches, not Expo's scaffolding. | Restore from git, or `prebuild --clean` on the SDK 56 CLI (step 3 above). |
| **`@expo/fingerprint` hashes** | 0.20.x hashes differ from 0.19.x for the same tree because the *inputs* (RN version, package versions) differ. Any recorded fingerprint computed on 57 is not comparable. | Recompute after rollback; expect one more cache-miss build. |

Node version is **not** on this list — SDK 57 does not raise the floor (§1), so nothing needs
reverting.

---

## 12. Post-upgrade verification checklist

Run all of these before shipping. Each maps to a specific, verified SDK 57 change.

| # | Check | Command / procedure | Covers |
|---|-------|---------------------|--------|
| 1 | Dependency alignment | `npx expo install --fix` reports no changes on a second run; lockfile diff matches §4; `react` still `19.2.3`, `react-native-screens` still `~4.26.0` | §4 |
| 2 | Doctor | `npx expo-doctor@latest` — clean, or every remaining finding understood | §2 |
| 3 | Clean prebuild | `rm -rf ios android && npx expo prebuild --clean`, then `git diff` the regenerated `ios/`/`android/` and confirm every config-plugin-injected change is still present. Hand edits that were never in a plugin are gone — this is the main 57 footgun | §6 |
| 4 | iOS deep link — cold start | Kill the app fully, then `xcrun simctl openurl booted "myapp://some/path"`. Unchanged in 57, but worth a smoke test after any prebuild regeneration | §5, §6 |
| 5 | **OTA runtime version match** | `npx expo-updates runtimeversion:resolve --platform ios` / `--platform android`; diff against the pre-upgrade values recorded in §7. If changed, follow the republish procedure before publishing anything | §7 |
| 6 | **OTA end-to-end** | Publish to a test branch, install the **new** build, confirm the update is received. Then confirm an **old** SDK 56 install still receives updates from the old branch | §7 |
| 7 | Worklet / animation smoke test | Exercise every Reanimated worklet, gesture handler and `@expo/ui` worklet prop. `react-native-worklets` jumps 0.8.3 → 0.10.0 — the single largest third-party move in this upgrade | §3, §4 |
| 8 | **Camera output** | Capture a photo on iOS and check file size and EXIF dimensions against the SDK 56 baseline — `pictureSize` default changed `high` → `photo` | §9 |
| 9 | Web SSR fonts | If you SSR with `expo-font`, replace `Server.resetServerContext()` with `Server.withServerContext(cb)` and confirm no throw and no font bleed between concurrent renders | §3 |
| 10 | DevTools plugins | If you ship a custom DevTools plugin with a WebSocket handler, confirm it reads `request.url`/`request.headers` off a fetch `Request` | §3 |
| 11 | Native Swift modules | If you author iOS modules: rebuild and fix any `JavaScriptValue`-caught-as-`Error` or non-copyable `JavaScriptError` usage | §3 |
| 12 | Android release build | `npx expo run:android --variant release` (or an EAS `production` build). No ProGuard/R8 change in 57, but RN 0.86 is a new release-build surface | §8 |

Deliberately **not** on this list, because they are not part of the 56 → 57 upgrade: a Node version
gate, `SceneDelegate.swift` / `UIApplicationSceneManifest` existence checks, R8 reflection auditing,
and `expo-video` background-audio behaviour.

---

## Sources

- Release branches `origin/sdk-56` and `origin/sdk-57` of `expo/expo` — the shipped source for each
  SDK line. Every claim above was checked with `git show origin/sdk-5{6,7}:<path>` or
  `git grep <sym> origin/sdk-5{6,7} -- <path>`.
- `packages/expo/bundledNativeModules.json` on each release branch — the only authoritative pin
  source; it ships inside the `expo` package and is what `expo install` resolves against.
- Published npm tarballs / `npm view` — used to confirm the absence of `engines` on `expo@57.0.8`,
  the dist-tags (`latest` 57.0.8, `sdk-56` 56.0.17) and publish dates.
- Root `CHANGELOG.md` `## 57.0.0` on `origin/sdk-57` — its "🛠 Breaking changes" section has exactly
  four entries; it has no "3rd party library updates" subsection, so dependency bumps are not in it.
- `packages/*/CHANGELOG.md` on **both** release branches — used to separate real 57 deltas from
  SDK 56 backports (§3b).
- `docs/pages/build-reference/infrastructure.mdx` (live copy) for EAS image aliases — the copy on
  the `sdk-57` branch is stale.
- **Not used, and actively contradicted above:** the `main` branch (SDK 58 in progress), any
  `## Unpublished` changelog section, and
  `docs/public/static/schemas/v5{6,7}.0.0/native-modules.json` (stale pins).
