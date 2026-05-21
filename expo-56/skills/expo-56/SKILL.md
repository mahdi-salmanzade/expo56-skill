---
name: expo-56
description: >-
  Authoritative reference for Expo SDK 56 (React Native 0.85, React 19.2, Hermes
  v1) — the current/latest Expo release, which postdates your training, so defer
  to this skill over memory for anything Expo. Use it whenever the user is
  building, configuring, debugging, or upgrading an Expo app: expo-router file
  routes (e.g. `app/(tabs)/_layout.tsx`, protected/auth routes), @expo/ui, Expo
  Modules (Swift/Kotlin native modules, config plugins), EAS Build/Update/Submit
  (channels, OTA, profiles, credentials), app.json/app.config, or any `expo-*`
  package (camera, notifications, location, audio, video, sqlite, file-system,
  secure-store, av, etc.). Trigger even when the user doesn't say "SDK 56" but
  shows any Expo signal — `npx expo`, `expo install`, eas.json, an `expo-`
  import — or asks about the "latest/newest Expo SDK", upgrading an Expo app a
  full SDK version, or errors after `expo install expo@latest --fix`; the named
  version may be wrong, so still consult this. SDK 56 changed many APIs,
  defaults, and minimums (expo-av removed, file-system rewritten with async
  copy/move, expo/fetch as default fetch, redesigned Calendar/Contacts/MediaLibrary,
  expo-router dropped React Navigation, @expo/vector-icons deprecated). Do NOT
  trigger for bare React Native projects without Expo, or for Flutter, native
  iOS/Android, or web-only work.
---

# Expo SDK 56

> **Verified against the live Expo docs and the SDK 56 changelog as of 2026-05-22.** Bundled native-dep versions below (reanimated `4.3.1`, screens `4.25.1`, React `19.2.3`, etc.) are SDK 56.0 pins — re-verify against `npx expo install --fix` output after any SDK patch/minor bump, since these references are a point-in-time snapshot.
>
> **Verification status (spot-check 2026-05-22):** Independently re-confirmed against live docs — core versions (RN 0.85.2, React 19.2.3, Hermes v1, Node ≥20.19.4), `expo-contacts` (`Contact.getAllDetails` / `ContactField` / `ContactsSortOrder`), `expo-file-system` (async `copy`/`move` + `*Sync` + `File`/`Directory`/`Paths`), and `expo-widgets` (`createWidget` / `updateTimeline`). **Not yet confirmed:** the `expo-type-information` package name and its `module-interface` / `inline-modules-interface` / `short-module-interface` command names — the *capability* (CLI TypeScript generation for inline modules with watch mode) is documented, but these exact names were not found on npm or the docs index. Treat them as unverified; check `npx <pkg> --help` / the live inline-modules reference before relying on them. SDK 56 was in **beta** at this date, so some package versions/APIs may still shift before stable.

This skill covers **both** building new SDK 56 apps with correct, current APIs **and** migrating existing apps from SDK 55. SDK 56 shipped many breaking changes and new defaults that postdate model training — treat the bundled `references/` files as ground truth and consult them before writing non-trivial code. Don't rely on memory for Expo APIs; verify against the relevant reference.

## Why this matters

Several SDK 56 APIs are **new or breaking** versus what most React Native knowledge assumes:
- `expo/fetch` is now the default `globalThis.fetch`.
- `expo-file-system` `copy()`/`move()` are now **async**; the whole module moved to an object-oriented `File`/`Directory`/`Paths` API.
- `expo-router` **no longer depends on React Navigation**.
- `expo-av` is **removed** — use `expo-video` + `expo-audio`.
- Calendar / Contacts / MediaLibrary were **redesigned** (old APIs deprecated, importable only from `/legacy`).
- `@expo/vector-icons` is deprecated in favor of scoped `@react-native-vector-icons/*`.

If you write code from memory for any of these, it will likely be wrong. Check the reference.

## Quick facts (SDK 56)

| Thing | Value |
|-------|-------|
| React Native | 0.85 |
| React | 19.2 (index pins 19.2.3) |
| JS engine | **Hermes v1** (default; opt out `useHermesV1: false` in expo-build-properties) |
| New Architecture | **Mandatory** — no opt-out |
| Node.js | ≥ 20.19.4 |
| Xcode | ≥ 26.4 |
| iOS / tvOS min | 16.4 (macOS 13.4) |
| TypeScript | 6.0.3 |
| New project | `npx create-expo-app@latest --template default@sdk-56` |
| Upgrade | `npx expo install expo@^56.0.0 --fix` then `npx expo-doctor@latest` |
| Expo Go | **Not on App Store / Play Store for SDK 56** — install via CLI / TestFlight beta / `eas go` (see `references/19`) |

Key bundled native dep versions: reanimated `4.3.1` (needs `react-native-worklets`), gesture-handler `~2.31.1` (use the `Gesture.Pan()` builder API), screens `4.25.1`, safe-area-context `~5.7.0`, svg `15.15.4`. Details in `references/17`.

## How to use this skill

1. Identify the topic of the user's task.
2. Open the matching reference file(s) from the routing table below — read the relevant section, not the whole file. Most files use numbered H2 sections (`## 1.`, `## 2.`, …); a few (`07`, `12`, `17`, `20`) use descriptive un-numbered H2 headings instead. Either way, grep the file's `^## ` headings first and jump to the one you need rather than reading top-to-bottom — several references run 700–1050 lines.
3. For migrations, also read the **Migration checklist** section below and the per-topic "what changed in SDK 56" / "Migration" sections inside each reference.
4. Write code using the verbatim signatures and import paths from the reference. When a reference flags a discrepancy (see "Known discrepancies" below), prefer the reference's note and verify against the live docs if it's load-bearing.

## Routing table

Read the file whose domain matches the task, then jump to the relevant H2 section (see step 2 above on how sections are organized) rather than reading the whole file.

| If the task involves… | Read |
|---|---|
| Versions, environment/tooling requirements, New Architecture, **creating** a project, **upgrading** to 56, the top-level breaking-change & codemod list | `references/01-core-setup-upgrade.md` |
| Routing, navigation, `<Stack>`/`<Tabs>`/`<Link>`, layouts, typed routes, API routes, **React Navigation removal + codemod**, native stack v5, streaming SSR, data loaders, `SuspenseFallback` | `references/02-expo-router.md` |
| `@expo/ui` (SwiftUI / Jetpack Compose), universal components, native components, datetimepicker/bottom-sheet drop-in replacements | `references/03-expo-ui.md` |
| Authoring native modules, the Module DSL, inline modules, `expo-type-information`, `create-expo-module`, type-safe config plugins | `references/04-expo-modules-api.md` |
| Expo CLI flags, Metro config, on-demand filesystem, Node watcher vs Watchman, tree-shaking, env vars, TypeScript 6/7, `import.meta` | `references/05-cli-metro-bundling.md` |
| `expo-file-system` (File/Directory/Paths, upload/download tasks, async copy/move) and `expo/fetch` networking | `references/06-filesystem-networking.md` |
| **Redesigned** `expo-calendar`, `expo-contacts`, `expo-media-library`; `expo-audio` `useAudioStream`; `expo-haptics`; `expo-asset` GLB | `references/07-media-device-apis.md` |
| `expo-sqlite` (blobs, changesets), `expo-updates` / EAS Update + Hermes bytecode diffing, Convex integration | `references/08-sqlite-updates-convex.md` |
| `<StatusBar>`, `<NavigationBar>`, iOS Widgets (`expo-widgets`), `expo-dev-client` updates, vector-icons migration | `references/09-system-ui-components.md` |
| iOS/Android build performance, EAS Build precompiled artifacts, brownfield/native integration (`multipleFrameworks`), AGENTS.md/CLAUDE.md scaffolding | `references/10-eas-build-brownfield.md` |
| `expo-camera`, `expo-image`, `expo-image-picker`, `expo-image-manipulator`, `expo-gl`, `expo-video-thumbnails` | `references/11-camera-visual-media.md` |
| `expo-video`, `expo-audio` (playback/recording), **`expo-av` removal/migration**, `expo-screen-capture` | `references/12-video-audio-playback.md` |
| `expo-auth-session`, `expo-secure-store`, `expo-local-authentication`, `expo-crypto`, `expo-apple-authentication`, `expo-web-browser`, **Router auth pattern (`Stack.Protected`)** | `references/13-auth-security.md` |
| `expo-notifications`, push setup, FCM/APNs credentials, `expo-task-manager`, `expo-background-task`, `expo-device`, `expo-application` | `references/14-notifications-background.md` |
| `expo-location`, `expo-sensors`, `expo-screen-orientation`, `expo-brightness`, `expo-battery`, `expo-cellular`, `expo-network` | `references/15-location-sensors.md` |
| `app.json`/`app.config.[js,ts]` schema, config plugins, app icons & splash, `expo-build-properties`, `expo-constants`, `expo-system-ui`, `expo-splash-screen`, `expo-font`, `expo-linking`, `expo-localization` | `references/16-app-config-foundational.md` |
| `react-native-reanimated`, `react-native-gesture-handler`, `react-native-screens`, `react-native-safe-area-context`, `react-native-svg` + bundled versions | `references/17-animation-gesture-deps.md` |
| `eas.json`, EAS Build profiles, internal distribution, credentials, EAS Submit, EAS Update channels/branches, EAS Workflows, EAS Metadata, EAS env vars | `references/18-eas-full-workflow.md` |
| Development builds, debugging tools, unit testing (`jest-expo`), E2E (Maestro), Expo Go SDK 56 install path | `references/19-dev-workflow-testing.md` |
| `expo-maps`, web support / output modes (`single`/`static`/`server`), `expo-clipboard`/`sharing`/`print`/`mail-composer`/`sms`/`store-review`/`tracking-transparency`/`blur`/`linear-gradient` | `references/20-maps-web-utilities.md` |

If a task spans multiple domains (e.g. "build a screen that records audio and uploads it"), read each relevant file.

## Migration checklist (SDK 55 → 56)

When upgrading an existing app, work through this. Full detail in `references/01` §7 and the per-topic "Migration" sections.

1. **Bump & fix deps**: `npx expo install expo@^56.0.0 --fix`, then `npx expo-doctor@latest`. Confirm tooling minimums (Node ≥20.19.4, Xcode ≥26.4, iOS ≥16.4).
2. **Expo Router → no React Navigation**: run `npx expo-codemod sdk-56-expo-router-react-navigation-replace src`. Maps `@react-navigation/*` imports to `expo-router/*`. See `references/02` §1.
3. **Vector icons**: run `npx @react-native-vector-icons/codemod`; migrate `@expo/vector-icons` → scoped `@react-native-vector-icons/*`. The `expo` package no longer pulls in `@expo/vector-icons`. See `references/09` §4.
4. **`expo/fetch` is now default `fetch`**: audit code relying on RN fetch quirks; opt out with `EXPO_PUBLIC_USE_RN_FETCH=1` if needed. See `references/06` §6.
5. **`expo-file-system`**: `copy()`/`move()` are now async (use `copySync()`/`moveSync()` for old behavior); migrate to the new `File`/`Directory`/`Paths` API. See `references/06` §1 & §7.
6. **`expo-av` removed**: migrate Video → `expo-video`, Audio → `expo-audio`. See `references/12`.
7. **Calendar / Contacts / MediaLibrary**: old APIs deprecated and only importable from `/legacy` (throw at runtime otherwise); adopt the redesigned object-oriented APIs. See `references/07`.
8. **Notifications**: foreground handler now uses `shouldShowBanner`/`shouldShowList` (not `shouldShowAlert`); `expo-background-fetch` → `expo-background-task`. See `references/14`.
9. **DOM WebView**: `@expo/dom-webview` is the default (no longer needs `react-native-webview`).
10. **Hermes v1**: default engine; opt out via `useHermesV1: false` in `expo-build-properties` only if you hit issues.
11. **Reanimated 4**: now requires the separate `react-native-worklets` package. See `references/17`.

## Known discrepancies (read before trusting a single source)

These were flagged while compiling the references because the live docs lagged the SDK 56 changelog. When one is load-bearing, verify against the current docs page:
- **`android.usePrecompiledHeaders`**: named in the changelog but not surfaced on the live `expo-build-properties` page; closest documented keys are `usePrecompiledModules` / `buildReactNativeFromSource`. (`references/16` §5, `references/10` §2)
- **gesture-handler API**: the live docs render a newer 3.x `usePanGesture()` hook, but SDK 56 pins 2.31.x whose canonical API is the `Gesture.Pan()` builder. Use the builder API. (`references/17`)
- **`expo-type-information` / typegen CLI**: `references/04` documents this as the package + `module-interface` / `inline-modules-interface` / `short-module-interface` commands for auto-generating TypeScript from Swift/Kotlin native modules. The capability is real and documented (CLI typegen for inline modules, watch mode), but as of the 2026-05-22 spot-check the exact package name and command names could not be confirmed on npm or the docs index. Before instructing the user, verify the installed package and run `npx <pkg> --help`; do not present the names as certain.
- Some changelog-only items (EAS per-step timing, "EAS Observe coming soon") have no dedicated docs page yet. (`references/10`, `references/18`)

## Conventions when generating code

- Use **TypeScript** and file-based routing (`expo-router`) unless the user's project clearly does otherwise.
- Install packages with `npx expo install <pkg>` (not bare `npm install`) so versions match the SDK.
- Use permission **hooks** where a package offers them (e.g. `useCameraPermissions`, `useAudioPermissions`) rather than imperative permission calls, and wire required permission strings/config-plugin props in `app.json`.
- Prefer the SDK 56 API surface (e.g. `expo-video`'s `useVideoPlayer`, file-system's `File`/`Paths`) over deprecated/legacy equivalents, and tell the user when you're avoiding a deprecated path.
