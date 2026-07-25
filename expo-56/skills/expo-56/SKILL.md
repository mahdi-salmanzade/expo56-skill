---
name: expo-56
description: >-
  Authoritative, version-pinned reference for Expo SDK 56 and SDK 57 — both
  postdate your training, so defer to this skill over memory for anything Expo.
  SDK 57 is the current release (2026-07-08, React Native 0.86); SDK 56 is the
  previous, still-supported release (React Native 0.85). Use it whenever the user
  is building, configuring, debugging, or upgrading an Expo app: expo-router file
  routes (e.g. `app/(tabs)/_layout.tsx`, protected/auth routes), @expo/ui, Expo
  Modules (Swift/Kotlin native modules, config plugins), EAS Build/Update/Submit
  (channels, OTA, profiles, credentials), app.json/app.config, or any `expo-*`
  package (camera, notifications, location, audio, video, sqlite, file-system,
  secure-store, maps, sensors, calendar, contacts, etc.). Trigger even when the
  user doesn't name a version but shows any Expo signal — `npx expo`, `expo
  install`, eas.json, an `expo-` import — or asks about the "latest/newest Expo
  SDK", or is upgrading an Expo app a full SDK version (55→56 or 56→57), or hits
  errors after `expo install expo@latest --fix`. Always determine which SDK the
  project is actually on before answering; the version a user names is often
  wrong. These releases changed many APIs, defaults and minimums (expo-av
  removed, file-system rewritten with async copy/move, expo/fetch as the default
  fetch, expo-router dropped React Navigation, @expo/vector-icons deprecated,
  and in 57: `prebuild` cleans by default, RN 0.86, and a changed `expo-camera`
  iOS capture default). Do NOT trigger for bare React Native projects without
  Expo, or for Flutter, native iOS/Android, or web-only work.
---

# Expo SDK 56 & 57

> **Audited against the `expo/expo` release branches `origin/sdk-56` and `origin/sdk-57` on 2026-07-25.** SDK 57 shipped 2026-07-08; its latest patch is **57.0.8** (published 2026-07-22, dist-tag `latest`). SDK 56 shipped 2026-06-01 and is **still actively patched** — latest **56.0.17** (published 2026-07-23, dist-tag `sdk-56`), i.e. *newer than the SDK 57 patch*. The supported window is SDK 54–57.
>
> **Sourcing rule — this matters more than it sounds.** Version pins here come from `packages/expo/bundledNativeModules.json` **on the release branch** (`git show origin/sdk-57:packages/expo/bundledNativeModules.json`), because that file ships inside the `expo` package and is what `expo install` resolves against. Two tempting sources are **wrong**:
> - `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json` is **stale** — it still lists `react-native-screens` 4.25.2 and `expo-router` ~56.2.9 when the shipped values are ~4.26.0 and ~56.2.16.
> - The **`main` branch contains post-57 work destined for SDK 58.** Anything under `## Unpublished` in a CHANGELOG, and any `packages/*/package.json` value on `main`, is **not** in SDK 57.
>
> Re-verify with `npx expo install --fix` after any patch bump; this is a point-in-time snapshot.

This skill is a **version-pinned API reference**, not a task playbook. It covers building on SDK 56 or 57, and the **55→56** and **56→57** migrations. Expo's own official skills are playbooks for the newest SDK and carry no version pins — see [Composing with the official Expo skills](#composing-with-the-official-expo-skills). Use both.

## Step 1: Establish which SDK the project is on

Do this **before** answering anything version-specific. A user saying "latest Expo" may be on either release. The genuine 56↔57 differences that silently break code are narrow: `prebuild` semantics, `expo-camera`'s iOS default `pictureSize`, and the RN 0.86 bump. Most other "SDK 57" changes were **also backported to the 56 line** — check the patch version before telling anyone to upgrade.

```bash
# Authoritative — the installed major decides everything else
cat package.json | grep '"expo"'      # → "expo": "~57.0.8"  or  "~56.0.17"
npx expo config --type public         # if you need the resolved config too
```

- Major `57` → use SDK 57 values; each reference's **`## SDK 57 delta`** section is the source of truth for what changed.
- Major `56` → use the main body of each reference as-is.
- No `expo` dependency → this is a bare React Native project. This skill does not apply.
- User is starting fresh → default to **SDK 57**.

When you are unsure, say which SDK you assumed. A correct answer for the wrong SDK is still a wrong answer.

## Quick facts

| | **SDK 57** (current) | **SDK 56** (previous, supported) |
|---|---|---|
| Released | 2026-07-08 | 2026-06-01 |
| Latest patch | 57.0.8 | 56.0.17 |
| React Native | **0.86.0** | 0.85.3 |
| React | 19.2.3 | 19.2.3 |
| JS engine | **Hermes v1** (default) | **Hermes v1** (default) |
| New Architecture | Mandatory | Mandatory |
| Node.js | No enforced floor — neither `expo@56` nor `expo@57` declares `engines` | Same |
| iOS / tvOS min | 16.4 (macOS 13.4) | 16.4 (macOS 13.4) |
| TypeScript | 6.0.3 | 6.0.3 |
| iOS app lifecycle | `AppDelegate` (scenes are **SDK 58**, not 57) | `AppDelegate` |
| `expo prebuild` | **Cleans by default** (`--no-clean` to opt out) | Additive by default |
| New project | `npx create-expo-app@latest --template default@sdk-57` | `... --template default@sdk-56` |
| Upgrade | `npx expo install expo@^57.0.0 --fix` | `npx expo install expo@^56.0.0 --fix` |

**Bundled dependency pins that differ between the two:**

Exactly **seven** third-party packages moved (`bundledNativeModules.json`, release branches):

| Package | SDK 56 | SDK 57 |
|---|---|---|
| `react-native` | 0.85.3 | **0.86.0** |
| `react-native-reanimated` | 4.3.1 | **4.5.0** |
| `react-native-worklets` | 0.8.3 | **0.10.0** |
| `react-native-gesture-handler` | ~2.31.1 | **~2.32.0** |
| `react-native-keyboard-controller` | 1.21.6 | 1.21.9 |
| `react-native-pager-view` | 8.0.1 | 8.0.2 |
| `lottie-react-native` | ~7.3.4 | ~7.3.8 |

**Unchanged across both** (state these confidently, and don't churn them during an upgrade): `react-native-screens` **~4.26.0**, `react-native-safe-area-context` ~5.7.0, `react-native-svg` 15.15.4, `react-native-webview` 13.16.1, `@shopify/react-native-skia` 2.6.2, `@shopify/flash-list` 2.0.2, `react-native-web` ~0.21.0, `react-native-maps` 1.27.2, `@expo/vector-icons` ^15.0.2, `@react-native-async-storage/async-storage` 2.2.0.

First-party `expo-*` packages track their SDK line, but **not at a flat `.0`** — take the exact pin from the release branch rather than assuming: `expo-router` ~56.2.16 → ~57.0.8, `expo-video` ~56.1.4 → ~57.0.2, `expo-dev-client` ~56.0.24 → ~57.0.9, `expo-updates` ~56.0.23 → ~57.0.10, `@expo/fingerprint` ~0.19.9 → ~0.20.6.

> These are branch-HEAD values. What `expo install` resolves today comes from the *published* `expo@57.0.8` tarball, which trails the branch by one patch on two packages (`expo-dev-client` ~57.0.8, `expo-updates` ~57.0.9). Either is defensible; don't treat a one-patch disagreement as an error.

> **Expo Go:** the App Store / Play Store build targets **SDK 54**. Neither SDK 56 nor 57 has a store build. Install a matching client with `npx expo-go download <android|ios> latest`, from expo.dev/go, TestFlight, or `eas go` — but note it works for Android devices, Android emulators and the iOS Simulator only; Apple policy blocks side-loading older builds onto a physical iPhone. For anything real, use a development build. See `references/19`.

## Why this matters

Several APIs across these releases are **new or breaking** versus what React Native knowledge assumes. If you write these from memory you will likely be wrong:

- `expo/fetch` is the default `globalThis.fetch` (56+).
- `expo-file-system` `copy()`/`move()` are **async**; the module is an object-oriented `File`/`Directory`/`Paths` API (56+).
- `expo-router` **no longer depends on React Navigation** (56+).
- `expo-av` is **removed** — use `expo-video` + `expo-audio` (56+).
- Calendar / Contacts / MediaLibrary were **redesigned**; old APIs are importable only from `/legacy` (56+).
- `@expo/vector-icons` is deprecated in favour of scoped `@react-native-vector-icons/*` (56+).
- `npx expo prebuild` **wipes and regenerates** `ios/`/`android/` by default (57).
- The `appVersion` runtime-version policy now honours `ios.version`/`android.version`, which silently changes runtime versions and can break OTA matching — but this is **not** a 57 feature: `@expo/config-plugins` shipped it to both lines on the same day (56.0.12 and 57.0.3, 2026-07-07).

## How to use this skill

1. Establish the SDK version (Step 1 above).
2. Identify the task's domain and open the matching reference from the routing table.
3. **Grep the file's `^## ` headings first and jump to the section you need** — several references run 700–1100 lines; do not read them top-to-bottom.
4. If the project is on **SDK 57**, also read that file's trailing **`## SDK 57 delta`** section. Every reference has one; where a domain was untouched by 57 it says so explicitly, which is itself useful — it means you can trust the main body as-is.
5. Write code using the verbatim signatures and import paths from the reference. Where a reference flags a discrepancy, prefer its note and verify against live docs if it's load-bearing.

## Composing with the official Expo skills

Expo publishes official skills (`/plugin install expo@claude-plugins-official`). They are **task playbooks for the newest SDK**; this skill is a **version-pinned API reference**. They compose well — neither substitutes for the other.

The IDs below are those of the currently shipping plugin (v1.3.0, **16 skills**). Expo's upstream registry has since renamed several and added more, so if an ID doesn't resolve, try the newer name in parentheses.

| Task | Official skill |
|---|---|
| Screen/UI construction, Apple HIG, safe areas, blur & liquid glass, SF Symbols, native tabs, form sheets, Link previews | `building-native-ui` (→ `expo-native-ui`) |
| SwiftUI / Jetpack Compose component recipes | `expo-ui` |
| Native module view lifecycle, AppDelegate/Activity hooks, autolinking config | `expo-module` |
| Embedding RN in an existing native app | `expo-brownfield` |
| Store submission, TestFlight, ASO, App Store metadata | `expo-deployment` (→ `eas-app-stores`) |
| Writing `.eas/workflows/*.yml` | `expo-cicd-workflows` (→ `eas-workflows`) — it fetches the live schema; never hand-write from a frozen copy |
| React Query / SWR / offline / retry / cancellation | `native-data-fetching` (→ `expo-data-fetching`) |
| Expo Router API routes (`+api.ts`) | `expo-api-routes` |
| SDK upgrade procedure and topic migrations | `upgrading-expo` (→ `expo-upgrade`) — pair it with `references/21` here for the version specifics |
| NativeWind, DOM components (`'use dom'`), App Clips, EAS Observe, dev clients, update health, `expo/examples` | `expo-tailwind-setup`, `use-dom` (→ `expo-dom`), `add-app-clip` (→ `expo-app-clip`), `expo-observe` (→ `eas-observe`), `expo-dev-client`, `eas-update-insights`, `expo-examples` |

*(Upstream-only, not in v1.3.0: `expo-web-to-native`, `eas-simulator`, `eas-hosting`, `expo-router`, `expo-project-structure`.)*

**Use this skill — no official skill covers these:** location, sensors, calendar, contacts, media-library, maps, notifications & push credentials, task-manager/background-task, device/application, secure-store, auth-session, local-authentication, crypto, image-manipulator, font/linking/localization, the `app.json` field schema, config-plugin authoring, Expo CLI & Metro config, inline modules / `expo-type-information`, jest-expo, and Maestro.

**Always take version facts from here, even mid-task in an official skill.** Official skills carry no pins and assume the newest SDK. Confirm the project's SDK, then take dependency versions, API shapes and minimums from the matching section here.

**Where the official skills are stale — trust this skill (verified against source):**
- **Hermes v1 is the default since SDK 56**, not opt-in (`expo-build-properties` resolves `useHermesV1 ?? true`). Opting out also needs `buildReactNativeFromSource: true` and a pinned legacy `hermes-compiler`.
- **`@expo/vector-icons` migrates to `@react-native-vector-icons/*`** via `npx @react-native-vector-icons/codemod`, *not* to `expo-symbols` (which is iOS SF Symbols only).
- **`expo-linear-gradient` is not deprecated.** `experimental_backgroundImage` is an experimental alternative, not a replacement.
- **Don't default to Expo Go.** Expo's own docs: "Expo Go is limited and not useful for building production-grade projects." Push notifications, SQLCipher and Apple/Google Pay do not work there.

If a conflict isn't listed here, check the pinned schema and the package's `CHANGELOG.md` before picking a side — and tell the user which source you used.

## Routing table

Open the file whose domain matches the task, jump to the relevant `## ` section, and add that file's `## SDK 57 delta` section when the project is on 57.

| If the task involves… | Read |
|---|---|
| Versions, tooling minimums, New Architecture, **creating** a project, **upgrading**, the top-level breaking-change & codemod list | `references/01-core-setup-upgrade.md` |
| Routing, navigation, `<Stack>`/`<Tabs>`/`<Link>`, layouts, typed routes, API routes, **React Navigation removal + codemod**, native stack, streaming SSR, data loaders | `references/02-expo-router.md` |
| `@expo/ui` (SwiftUI / Jetpack Compose), universal components, datetimepicker/bottom-sheet drop-in replacements | `references/03-expo-ui.md` |
| Authoring native modules, the Module DSL, inline modules, `expo-type-information`, `create-expo-module`, type-safe config plugins | `references/04-expo-modules-api.md` |
| Expo CLI flags, Metro config, on-demand filesystem, tree-shaking, env vars, TypeScript, `import.meta`, **`prebuild` semantics** | `references/05-cli-metro-bundling.md` |
| `expo-file-system` (File/Directory/Paths, upload/download tasks, async copy/move) and `expo/fetch` networking | `references/06-filesystem-networking.md` |
| **Redesigned** `expo-calendar`, `expo-contacts`, `expo-media-library`; `expo-audio` `useAudioStream`; `expo-haptics`; `expo-asset` | `references/07-media-device-apis.md` |
| `expo-sqlite`, `expo-updates` / EAS Update + Hermes bytecode diffing, **runtime-version policy**, Convex integration | `references/08-sqlite-updates-convex.md` |
| `<StatusBar>`, `<NavigationBar>`, iOS Widgets (`expo-widgets`), `expo-dev-client`, vector-icons migration | `references/09-system-ui-components.md` |
| iOS/Android build performance, EAS Build precompiled artifacts, brownfield/native integration, ProGuard/R8, AGENTS.md scaffolding | `references/10-eas-build-brownfield.md` |
| `expo-camera`, `expo-image`, `expo-image-picker`, `expo-image-manipulator`, `expo-gl`, `expo-video-thumbnails` | `references/11-camera-visual-media.md` |
| `expo-video`, `expo-audio` (playback/recording), **`expo-av` removal/migration**, `expo-screen-capture` | `references/12-video-audio-playback.md` |
| `expo-auth-session`, `expo-secure-store`, `expo-local-authentication`, `expo-crypto`, `expo-apple-authentication`, `expo-web-browser`, **Router auth (`Stack.Protected`)** | `references/13-auth-security.md` |
| `expo-notifications`, push setup, FCM/APNs credentials, `expo-task-manager`, `expo-background-task`, `expo-device`, `expo-application` | `references/14-notifications-background.md` |
| `expo-location`, `expo-sensors`, `expo-screen-orientation`, `expo-brightness`, `expo-battery`, `expo-cellular`, `expo-network` | `references/15-location-sensors.md` |
| `app.json`/`app.config.[js,ts]` schema, config plugins, icons & splash, `expo-build-properties`, `expo-constants`, `expo-system-ui`, `expo-splash-screen`, `expo-font`, `expo-linking`, `expo-localization` | `references/16-app-config-foundational.md` |
| `react-native-reanimated`, `react-native-gesture-handler`, `react-native-screens`, `react-native-safe-area-context`, `react-native-svg` + bundled versions | `references/17-animation-gesture-deps.md` |
| `eas.json`, EAS Build profiles, internal distribution, credentials, EAS Submit, EAS Update channels/branches, EAS Workflows, EAS Metadata, env vars | `references/18-eas-full-workflow.md` |
| Development builds, debugging, unit testing (`jest-expo`), E2E (Maestro), Expo Go install path & the `expo-go` CLI | `references/19-dev-workflow-testing.md` |
| `expo-maps`, web support / output modes, `expo-clipboard`/`sharing`/`print`/`mail-composer`/`sms`/`store-review`/`tracking-transparency`/`blur`/`linear-gradient` | `references/20-maps-web-utilities.md` |
| **Upgrading SDK 56 → 57** — full breaking-change list, ordered migration steps, per-package detail | `references/21-sdk-57-migration.md` |

If a task spans domains (e.g. "record audio and upload it"), read each relevant file.

## Migration: SDK 56 → 57

Full detail and the complete breaking-change list are in `references/21-sdk-57-migration.md`. The order below matters — the environment gates everything else.

1. **Bump & fix**: `npx expo install expo@^57.0.0 --fix`, then `npx expo-doctor@latest`. There is no Node gate to clear — neither `expo@56` nor `expo@57` declares `engines`; the `^22.13` bump landed *after* the 57 release and ships in SDK 58.
2. **`prebuild` now cleans by default.** If you hand-edited `ios/`/`android/`, either pass `--no-clean` or move those edits into config plugins. Prefer moving them — the new default exists because hand-edits rot.
3. **Runtime version — the silent OTA breaker, but it is *not* a 57 change.** `@expo/config-plugins` now honours `ios.version`/`android.version` in `Updates.getAppVersion`, `getNativeVersion`, and the `appVersion` runtime-version policy; previously those overrides were ignored and fell back to `package.json` (or `"1.0.0"`). Projects using only platform-specific versions get a **different runtime version**, so already-published updates stop matching installed builds. This shipped to **both** lines on 2026-07-07 (`56.0.12` and `57.0.3`), so it bites when you cross that patch on SDK 56 too — `npm ls @expo/config-plugins` tells you which side you're on. Verify and re-publish before shipping.
4. **Animation stack**: reanimated 4.5.0, worklets 0.10.0, gesture-handler ~2.32.0. Worklets 0.8→0.10 is the biggest jump; re-test worklet code.
5. **Check the one default that genuinely flipped in 57**: `expo-camera` iOS `pictureSize` `high`→`photo` (larger images, different EXIF; `expo-camera` 57.0.1, absent from the 56 line). Separately, `expo-widgets`' Android config plugin is opt-in behind `enableAndroid: true` — real, but **also on SDK 56** since `expo-widgets` 56.0.17, so it is not a reason to upgrade.
6. **Native module authors**: `expo-modules-jsi` iOS changed `JavaScriptError`/`JavaScriptValue`; check `expo-modules-core` Android deprecations. See `references/04`.
7. **Removed APIs**: `expo-font` web `Server.resetServerContext()` → `Server.withServerContext()`. See `references/21` for the full table.

**Not in SDK 57 — do not migrate for these yet.** They are on `main` for SDK 58 and are easy to mistake for 57 content: the iOS **UIKit scene lifecycle** (`SceneDelegate.swift`, `UIApplicationSceneManifest`), the enforced **Node `^22.13` `engines`** bump, and Android **`proguard-android-optimize.txt`**/R8-by-default. Verified: `git grep -l SceneDelegate origin/sdk-57 -- templates/` returns nothing (the only hits anywhere on that branch are an `expo-updates` e2e fixture and a macOS protocol shim — no shipped scaffolding).

## Migration: SDK 55 → 56

Full detail in `references/01` §7 and each reference's own "Migration" section.

1. **Bump & fix deps**: `npx expo install expo@^56.0.0 --fix`, then `npx expo-doctor@latest`. Confirm Node ≥20.19.4, Xcode ≥26.4, iOS ≥16.4.
2. **Expo Router → no React Navigation**: `npx expo-codemod sdk-56-expo-router-react-navigation-replace src` maps `@react-navigation/*` → `expo-router/*`. See `references/02` §1.
3. **Vector icons**: `npx @react-native-vector-icons/codemod`; the `expo` package no longer pulls in `@expo/vector-icons`. See `references/09` §4.
4. **`expo/fetch` is the default `fetch`**: audit code relying on RN fetch quirks; opt out with `EXPO_PUBLIC_USE_RN_FETCH=1`. See `references/06` §6.
5. **`expo-file-system`**: `copy()`/`move()` are async (`copySync()`/`moveSync()` keep the old behaviour); migrate to `File`/`Directory`/`Paths`. See `references/06` §1 & §7.
6. **`expo-av` removed**: Video → `expo-video`, Audio → `expo-audio`. See `references/12`.
7. **Calendar / Contacts / MediaLibrary**: old APIs throw unless imported from `/legacy`; adopt the redesigned APIs. See `references/07`.
8. **Notifications**: foreground handler uses `shouldShowBanner`/`shouldShowList` (not `shouldShowAlert`); `expo-background-fetch` → `expo-background-task`. See `references/14`.
9. **DOM WebView**: `@expo/dom-webview` is the default (no longer needs `react-native-webview`).
10. **Hermes v1**: default engine; opt out via `useHermesV1: false` only if you hit issues.
11. **Reanimated 4**: requires the separate `react-native-worklets` package. See `references/17`.

## Known discrepancies

Resolved by auditing package sources; kept because they still catch out anyone reading only the rendered docs.

- **`android.usePrecompiledHeaders`**: **real** — defined in `expo-build-properties`' `pluginConfig.ts` and implemented in `android.ts`, just absent from the auto-generated docs page. (`usePrecompiledModules` is the separate **iOS** key.) See `references/16` §5, `references/10` §2.
- **gesture-handler API**: both SDKs pin 2.31.x/2.32.x, whose canonical API is the `Gesture.Pan()` builder. Ignore any `usePanGesture()` hook shown on unversioned docs. See `references/17`.
- **`expo-type-information`**: **real** — CLI binary `expo-type-information` with commands `module-interface` / `inline-modules-interface` / `short-module-interface`. See `references/04`.
- **Verifying an SDK claim: use the release branch, not `main`.** This is the single biggest source of wrong answers about SDK 57, and it has already caught out published write-ups. `git show origin/sdk-57:<path>` is the shipped state; `main` is SDK 58 in progress. Concretely, all three of these are on `main` and **not** in SDK 57: the iOS scene lifecycle (`SceneDelegate.swift`), the `engines.node ^22.13` bump, and `proguard-android-optimize.txt`. A CHANGELOG entry under `## Unpublished` is never in the current SDK.
- **`packages/*/package.json` on `main` is NOT authoritative** — it still reads `56.0.5` while templates pin RN 0.86.
- **The frozen docs schemas are stale.** `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json` lags what actually shipped (it lists `react-native-screens` 4.25.2 and `expo-router` ~56.2.9 versus the real ~4.26.0 and ~56.2.16). Prefer `packages/expo/bundledNativeModules.json` on the release branch — that file ships inside the `expo` package and is what `expo install` resolves against.
- **The v57 versioned docs are a beta cut-off (2026-06-30)** while the v56 docs kept receiving backports through July. In places the v56 page is *newer and more complete* than its v57 counterpart. Do **not** read a v56↔v57 docs difference as evidence of an API change — use CHANGELOGs on the release branch and package source.
- Some changelog-only items (EAS per-step timing) have no dedicated docs page yet. See `references/10`, `references/18`.

## Conventions when generating code

- Use **TypeScript** and file-based routing (`expo-router`) unless the project clearly does otherwise.
- Install with `npx expo install <pkg>` (not bare `npm install`) so versions match the SDK.
- Use permission **hooks** where offered (`useCameraPermissions`, `useAudioPermissions`) rather than imperative calls, and wire the matching permission strings / config-plugin props in `app.json`.
- Prefer the current API surface (`expo-video`'s `useVideoPlayer`, file-system's `File`/`Paths`) over legacy equivalents, and say so when you're avoiding a deprecated path.
- State which SDK your answer targets whenever the two differ.
