# Expo SDK 56 — Core Platform, Setup, Upgrade & New Architecture

> Knowledge-base reference compiled from official Expo documentation and the `expo/expo` monorepo.
> Domain: Core platform, setup, upgrade & New Architecture.
> Verified against the release branches `origin/sdk-56` (expo **56.0.17**, `@expo/cli` 56.1.21,
> `expo-template-default` 56.0.31) and `origin/sdk-57` (expo **57.0.8**, `@expo/cli` 57.0.10,
> `expo-template-default` 57.0.10), plus the published npm tarballs. SDK 56.0.0 shipped 2026-06-01;
> SDK 57.0.0 shipped 2026-07-08.
>
> Source rules used throughout, in priority order:
> 1. Published npm tarballs (`npm view` / `npm pack`) — ground truth for what a user actually installs.
> 2. `origin/sdk-56` / `origin/sdk-57` — the shipped source for each SDK line.
> 3. `packages/expo/bundledNativeModules.json` on the release branch — the **only** good pin source;
>    it ships inside the `expo` package and is what `expo install` / `--fix` resolves against.
>
> Not authoritative, and deliberately not used here: `main` (that is **SDK 58 in progress**),
> anything under a `## Unpublished` changelog heading, and
> `docs/public/static/schemas/vNN.0.0/native-modules.json` (stale — it pins react-native-screens
> 4.25.2 and expo-router ~56.2.9, neither of which is what ships; there is no `v57.0.0` directory on
> `origin/sdk-57` at all).
>
> One caveat that matters below: **`docs/` is not maintained on the release branches.** The docs site
> deploys from `main`, so doc-derived facts (the SDK compatibility table) can differ between the
> branch snapshot and the live site. Where they do, both values are printed.
> See `## SDK 57 delta` at the end of this file.

---

## 0. What models get wrong from memory

Six facts in this domain that are commonly mis-stated. Details in the sections below.

| Claim | Truth |
|-------|-------|
| React Native version | SDK 56 → **0.85.3**; SDK 57 → **0.86.0** (§1, `## SDK 57 delta`) |
| `npx expo prebuild` is additive | Only in SDK 56. **SDK 57 cleans by default**; pass `--no-clean` to keep the old behaviour (§5) |
| New Architecture can be disabled | No. SDK 55+ is New-Arch-only; `newArchEnabled` is inert (§3) |
| `useHermesV1: false` opts out of Hermes v1 | Alone it **throws at config-validation time**. It also requires `buildReactNativeFromSource: true` resolvable for that platform (§1) |
| There is a 56 → 57 codemod | There is not. The only codemod shipped is `sdk-56-expo-router-react-navigation-replace` (§7) |
| SDK 57 raises the Node floor | It does not. `react-native@0.85.3` and `@0.86.0` ship the **identical** `engines.node` (`^20.19.4 \|\| ^22.13.0 \|\| ^24.3.0 \|\| >= 25.0.0`), and no `expo` package has an `engines` field on either line (§2) |

---

## 1. Core Platform Versions (SDK 56)

Sources: `origin/sdk-56:packages/expo/bundledNativeModules.json` (the pin source `expo install` uses),
`origin/sdk-56:templates/expo-template-default/package.json`, and the `56.0.0` entry of
`docs/ui/components/SDKTables/sdk-versions.json`. https://expo.dev/changelog/sdk-56

| Component | Version (SDK 56) | Exact pin in `expo-template-default` |
|-----------|------------------|--------------------------------------|
| React Native | **0.85** | `0.85.3` |
| React / React DOM | **19.2.3** | `19.2.3` |
| React Native Web | 0.21.0 | `~0.21.0` |
| React Native TV | 0.85-stable | — |
| Hermes | **v1 — now the default engine** | — |
| TypeScript | **6.0.3** | `~6.0.3` (devDependency) |

### Hermes v1
- Hermes **v1** is the default JavaScript engine in SDK 56. The default of `useHermesV1` is `true`
  (`resolveConfigValue(config, platform, 'useHermesV1') ?? true`) — but the published v56 config-data
  JSON (`docs/public/static/data/v56.0.0/expo-build-properties.json`, identical on both release
  branches) still carries `"@default": "false"` and calls the engine "experimental". That is stale;
  trust the code. The corrected `@default: true` text lives in the v57 config-data JSON, which exists
  only on `main` — neither release branch has a `v57.0.0` data directory.
- Opting out is **not** a single flag. `validateConfig` throws unless two conditions hold:
  1. `buildReactNativeFromSource: true` resolvable for that platform. Otherwise it
     throws: "`useHermesV1`: false requires `buildReactNativeFromSource` to be `true` for iOS."
     (and the identical message for Android).
  2. If a `hermes-compiler` version is resolvable in the project, it must be exactly **`0.15.0`**.
     Otherwise it throws: "`useHermesV1`: false, requires setting the hermes-compiler version to
     0.15.0 through resolutions. Found version "&lt;x&gt;" instead."
- Working opt-out:
  ```json
  {
    "plugins": [
      ["expo-build-properties", {
        "useHermesV1": false,
        "buildReactNativeFromSource": true
      }]
    ]
  }
  ```
  plus a package-manager pin (`resolutions` / `overrides`) of `hermes-compiler` to `0.15.0`.
- Both keys resolve per platform via
  `resolveConfigValue = (config, platform, key) => config[platform]?.[key] ?? config[key]`. A
  platform block **overrides** the top-level value, it does not shadow it away — so a platform-scoped
  `useHermesV1: false` only needs `buildReactNativeFromSource: true` *resolvable for that platform*,
  either in the same `ios` / `android` block or at the top level. This passes validation:
  `{ "ios": { "useHermesV1": false }, "buildReactNativeFromSource": true }`.
  (Android additionally accepts the deprecated `android.buildFromSource` as a fallback.)

Source: `packages/expo-build-properties/src/pluginConfig.ts` (`LEGACY_HERMES_COMPILER_VERSION = '0.15.0'`,
`validateConfig`) on `origin/sdk-56` and `origin/sdk-57` — identical on both.

---

## 2. Tooling & Environment Requirements (SDK 56)

Source (enforceable floors): the `engines` field of the published tarballs.
Source (recommendation table): `docs/ui/components/SDKTables/sdk-versions.json`.
Source (general env guide): https://docs.expo.dev/get-started/set-up-your-environment/

### Hard minimums introduced/raised in SDK 56
- **Node.js**: the only enforced floor is `react-native`'s `engines.node`, which is
  **`^20.19.4 || ^22.13.0 || ^24.3.0 || >= 25.0.0`** — byte-identical on `react-native@0.85.3` (SDK 56)
  and `@0.86.0` (SDK 57), and mirrored by `expo-doctor` on both lines. **Node 20.19.4 is enough for
  both SDKs.** Neither `expo` (no `engines` field at all) nor `create-expo` (`>=18.13.0`) gates on it.
- **iOS / tvOS**: minimum **16.4** (bumped from 15.1).
- **macOS**: minimum **13.4** (`origin/sdk-56:CHANGELOG.md`, 56.0.0 §Breaking, #43296 — applied across
  ~30 `expo-*` packages).
- **Xcode**: **26.2+** per the table snapshot on both release branches; the live docs site says
  **26.4+**. See the caveat below — take 26.4+ as the safe number.

> **The docs compatibility table is edited only on `main`, so it drifts from the release branches.**
> Two edits landed on `main` and were never merged to `origin/sdk-56` or `origin/sdk-57`:
> #46308 (2026-05-27) raised the SDK 56 Xcode cell 26.2+ → 26.4+, and #47757 (2026-07-16, i.e. *after*
> SDK 57 shipped) lowered the SDK 56 Node cell 22.13.x → 20.19.x while adding an SDK 57 row at 22.13.x.
> Both release branches still show SDK 56 as Xcode 26.2+ / Node 22.13.x and have **no SDK 57 row at
> all**. The Node cell is a documented recommendation, not a gate — the `engines` floor above is the
> number that can actually fail an install.

### General environment (from set-up-your-environment, applies broadly)
- **JDK**: Java **17** required on all platforms.
  - macOS: `brew install --cask zulu@17`
  - Windows: `choco install -y microsoft-openjdk17`
  - Linux: OpenJDK 17
  - `JAVA_HOME` must be configured.
- **Watchman** (macOS): `brew update && brew install watchman`
- **Android Studio** (latest) + Android SDK Platform **36**. SDK 55, 56 and 57 all use
  `compileSdkVersion 36` / `targetSdkVersion 36` / `buildToolsVersion 36.0.0`, runtime **Android 7+**.
  `ANDROID_HOME` env var required; verify ADB with `adb --version`.
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

Other package managers: `yarn create expo-app --template default@sdk-56`,
`pnpm create expo-app --template default@sdk-56`, `bun create expo --template default@sdk-56`.

### What happens without `--template`
Source: `packages/create-expo/src/promptSdkVersion.ts` (identical on `origin/sdk-56` and `origin/sdk-57`).

`create-expo-app@latest` with no explicit SDK tag runs `applySdkVersionToTemplateAsync`, which fetches
`https://api.expo.dev/v2/versions` and **prompts** `Select an Expo SDK version:` with:
- `Latest (SDK N)` — "Recommended for most projects"
- `For learning with Expo Go (SDK M)` — shown only when the store Expo Go SDK differs from latest
- `Other SDK version…` — a second prompt listing the four most recent released SDKs

Non-interactive (`--yes`, `CI=1`, or no TTY) with the default template pins to
`expo-template-default@sdk-<latest>`. Passing an explicit tag (`--template default@sdk-56`) skips the
prompt entirely, as does `EXPO_BETA=1`.

> The doc note in `docs/pages/get-started/create-a-project.mdx` claims that omitting `--template`
> "creates an SDK N project" — a flat statement that the code contradicts, because the code prompts.
> The number in that note is also branch-dependent, so do not quote it: `origin/sdk-56` and
> `origin/sdk-57` both say *"During the SDK 56 transition period … creates an SDK **55** project"*,
> while `main` says *"During the SDK 57 transition period … creates an SDK **54** project"*. The
> number it is reaching for is whichever SDK the store build of Expo Go is on, which the CLI resolves
> at runtime from `https://api.expo.dev/v2/versions`. Trust the prompt, not the note.

### AI agent scaffolding (new in SDK 56)
Source: `packages/create-expo/src/generateAgentFiles.ts`, `packages/create-expo/src/cli.ts`.

New projects get three files, written **only if they do not already exist**:
- **AGENTS.md** — a 3-line stub: `# Expo HAS CHANGED` / `Read the exact versioned docs at
  https://docs.expo.dev/versions/v<major>.0.0/ before writing any code.`
- **CLAUDE.md** — literally `@AGENTS.md`
- **`.claude/settings.json`** — `{"enabledPlugins": {"expo@claude-plugins-official": true}}`

Skip all three with `--no-agents-md`.

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

- Bundled CLI version: **`@expo/cli` 56.1.21** on `origin/sdk-56` (57.0.10 on `origin/sdk-57`).

Common commands:
- `npx expo start` — launch dev server
- `npx expo prebuild` — generate native directories
- `npx expo run:ios` / `npx expo run:android` — compile locally
- `npx expo install [package]` — install version-compatible packages

### `prebuild` — the one version-sensitive default in this domain
Source: `packages/@expo/cli/src/prebuild/index.ts`.

| | SDK 56 (`@expo/cli` 56.1.21) | SDK 57 (`@expo/cli` 57.0.10) |
|---|---|---|
| Default | **Additive** — applies changes on top of existing `ios/` and `android/` | **Clean** — deletes and regenerates both folders |
| Flags | `--clean` deletes the native folders first | `--no-clean` restores the additive behaviour; `--clean` still parses but is now the default |
| Parsed as | `clean: args['--clean']` | `clean: !args['--no-clean']` |

Any workflow with hand-edited files under `ios/` or `android/` must pass `--no-clean` on SDK 57, or
move those edits into config plugins.

Verified in the published tarballs, not just the branches: `@expo/cli@56.1.21`'s
`build/src/prebuild/index.js` parses only `'--clean': Boolean` and sets `clean: args['--clean']`;
`@expo/cli@57.0.10` parses both `--clean` and `--no-clean` and sets `clean: !args['--no-clean']`.

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

   `expo@<version>` combined with `--fix` is a supported special case, but you cannot list other
   packages alongside it — `packages/@expo/cli/src/install/installAsync.ts` throws
   `Cannot install other packages with expo@<version> and --fix or --check`.

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
Expo Skills ships an upgrade skill. Install with `/plugin install expo@claude-plugins-official` in
Claude Code, or `npx skills add expo/skills`.

**The skill was renamed after SDK 57, so the name depends on when your plugin was fetched.** On both
release branches, `docs/ui/components/ExpoSkillsTable/data/expo-skills.json` is a 14-skill roster
(fetched 2026-05-21) in which the skill is **`upgrading-expo`**. On `main` it is a 21-skill roster
(fetched 2026-07-23, i.e. after the 57 release) where it has been renamed **`expo-upgrade`** — that
name is post-57. Note also that the release-branch walkthrough contains no `<RelatedSkills>` component
at all; the `<RelatedSkills names={['expo-upgrade']} />` line is `main`-only. If `upgrading-expo`
resolves in your environment, that is the correct 56/57-era name, not a stale one.

### Support window
SDK **54, 55 and 56** are listed as current on both release branches, with SDK 57 added on the live
site after it shipped; 53 and earlier sit under "Deprecated SDK Version Changelogs". Upgrade one major
at a time.
Source: `origin/sdk-57:docs/pages/workflow/upgrading-expo-sdk-walkthrough.mdx` §SDK Changelogs (lists
56 / 55 / 54) vs the same file on `main` (adds 57) — another instance of `docs/` being frozen on the
release branches.

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
- https://expo.dev/changelog/sdk-56 · https://expo.dev/changelog/sdk-57
- `origin/sdk-NN:packages/expo/bundledNativeModules.json` — the pin source `expo install` resolves against
- `origin/sdk-NN:templates/expo-template-default/package.json` — what a new project actually gets
- Published npm tarballs (`npm view <pkg>@<v> engines/dependencies`, `npm pack`) — final arbiter
- `docs/ui/components/SDKTables/sdk-versions.json` — tooling recommendation table (maintained on `main` only)
- `origin/sdk-56:CHANGELOG.md` · `origin/sdk-57:CHANGELOG.md` — top-level SDK breaking-change lists
- `packages/@expo/cli/src/prebuild/index.ts`, `packages/@expo/cli/src/install/installAsync.ts`
- `packages/expo-build-properties/src/pluginConfig.ts`, `packages/create-expo/src/`
- https://docs.expo.dev/get-started/set-up-your-environment/ · /get-started/create-a-project/
- https://docs.expo.dev/workflow/upgrading-expo-sdk-walkthrough/ · /guides/new-architecture/ · /more/expo-cli/

---

## SDK 57 delta

SDK 57.0.0 (2026-07-08, `expo@57.0.8` at the time of writing) changes **one** thing behaviourally in this
domain — `npx expo prebuild` now cleans by default — plus the React Native 0.85.3 → 0.86.0 bump and the
dependency pins. Everything else in sections 1–8 above holds unchanged for SDK 57.

> **SDK 56 is still actively patched (56.0.17 and climbing), and most of what the SDK 57 changelog
> lists was backported.** Every claim below was checked against *both* `origin/sdk-56` and
> `origin/sdk-57` changelogs. Items present in both are filed under "Already in the SDK 56 line" with
> the minimum patch version — for those, "upgrade to 57 to get X" is wrong advice; a patch bump on the
> line you are already on is enough.

### Breaking

**1. `npx expo prebuild` clears and regenerates the native folders by default.**
Listed under 🛠 Breaking changes in `origin/sdk-57:CHANGELOG.md` (## 57.0.0, #47209).

| | SDK 56 | SDK 57 |
|---|---|---|
| Source | `clean: args['--clean']` | `clean: !args['--no-clean']` |
| Help text | `--clean  Delete the native folders and regenerate them before applying changes` | `--no-clean  Apply changes to the existing native folders instead of recreating them` |

Migration: pass `--no-clean`, or move hand edits under `ios/` / `android/` into config plugins.
Source: `packages/@expo/cli/src/prebuild/index.ts` on both release branches.

**Complete SDK 57 top-level breaking list — exactly four items** (`origin/sdk-57:CHANGELOG.md`);
this file owns the index, details live in the referenced files:
1. `@expo/ui` [universal][android] — Android text input renders Compose `BasicTextField` instead of a
   Filled Material `TextField` (#46442). → ref 09/16
   **Also in the SDK 56 line** — `origin/sdk-56:packages/expo-ui/CHANGELOG.md` lists it under
   🛠 Breaking changes in **`@expo/ui` 56.0.17** (2026-06-10). Staying on 56 does not spare you this;
   pinning `@expo/ui` below 56.0.17 does.
2. `expo-modules-jsi` [iOS] — `JavaScriptError` is now a copyable class conforming to `Error` (was a
   non-copyable struct); `JavaScriptValue` no longer conforms to `Error` (#47154). → ref 16
   57-only (absent from every `origin/sdk-56` changelog).
3. `expo-font` [web] — `Server.resetServerContext()` **removed**; server-side font state is now scoped
   per render via `AsyncLocalStorage` (#46669). → ref 09. 57-only.
4. `@expo/cli` — prebuild clean-by-default (#47209), above. 57-only.

Nothing else is a top-level SDK 57 breaking change. Net: **three of the four breaking changes are
genuinely new in 57**; #46442 is a 56 backport.

### Genuinely new in 57 (not in the SDK 56 line)
- `expo-build-properties`: new `android.cmakeVersion`, and `expo-modules-autolinking` applies it to the
  app and all library subprojects (#47377). Absent from `origin/sdk-56` — `git grep cmakeVersion
  origin/sdk-56 -- packages/expo-build-properties/` returns nothing. The whole 56 → 57 diff of
  `packages/expo-build-properties/src/` is +33 lines across `pluginConfig.ts`, `android.ts` and one
  test file; `cmakeVersion` is the entirety of it.

That is the complete list for this domain. Everything else the SDK 57 changelog advertises as a new
feature here was backported — see below.

### Already in the SDK 56 line (do not upgrade for these)
Each of these appears in **both** `origin/sdk-56` and `origin/sdk-57` changelogs. The version given is
the earliest 56-line release containing it; the `expo` column is the `expo` package that bundles that
`@expo/cli` (from the published `dependencies` of each `expo@56.0.x`).

| Feature | PR | Lands in | Needs |
|---|---|---|---|
| Bundler-managed CocoaPods installs | #43605 | `@expo/cli` 56.1.13, `pod-install` 1.0.19 | `expo@>=56.0.7` |
| `expo customize metro.config.js` no longer installs `@expo/metro-config` → ref 05 | #46600 | `@expo/cli` 56.1.14 | `expo@>=56.0.9` |
| Device Hub as a Simulator replacement for Xcode 27+ | #46757 | `@expo/cli` 56.1.16 | `expo@>=56.0.12` |
| Experimental `tvos` / `macos` autolinking, gated by `expriments.outOfTreePlatforms` (the changelog really does misspell `experiments`) | #46344 | `@expo/cli` 56.1.17, `expo-modules-autolinking` 56.0.17 | `expo@>=56.0.13` |

Also do not treat **"Initial release of `@expo/require-utils`"** (SDK 57 top-level changelog) as new.
The package has been on npm since **55.0.0** and `origin/sdk-56` ships **`@expo/require-utils` 56.1.6**;
the changelog line is a top-level-index artifact, not a new package.

### Unchanged in 57 (checked, do not go hunting)
- **New Architecture** (§3): no change. Still always-on for SDK 55+, still no opt-out, still
  `newArchEnabled` honoured only on SDK 54 and earlier. `docs/pages/guides/new-architecture.mdx` is
  byte-identical on `origin/sdk-56` and `origin/sdk-57` (`git diff` between them is empty; both still
  print `--template default@sdk-56`). The restructured version of that page — SDK-55+ callout, a
  `<Tabs>` split for "SDK 55 and later" / "SDK 53 and SDK 54" / "SDK 52", a four-package-manager
  `<Terminal>`, and the `default@sdk-57` snippet — exists only on `main`, i.e. post-57.
- **Codemods** (§7): there is **no 56 → 57 codemod**. `packages/expo-codemod/src/transforms` on
  `origin/sdk-57` still contains only `index.ts` and `sdk-56-expo-router-react-navigation-replace.ts`;
  `docs/pages/router/migrate/` has no `sdk-56-to-57.mdx`.
- **Project scaffolding** (§4): the SDK prompt, the AGENTS.md / CLAUDE.md / `.claude/settings.json`
  trio and `--no-agents-md` are byte-identical on both branches (`create-expo` 4.0.2 → 5.0.0, but
  `git diff origin/sdk-56 origin/sdk-57 -- packages/create-expo/src/` is empty). SDK 57 command:
  `npx create-expo-app@latest --template default@sdk-57`.
- **Hermes v1** (§1): same default (`true`), same two-condition opt-out, same `0.15.0` compiler pin.
- **Native templates**: `git diff --stat origin/sdk-56 origin/sdk-57 -- templates/` touches only five
  `package.json` files and one `.tgz`. **No native file changed between the two releases.**

Three things that are easy to mis-attribute to SDK 57 because they are prominent on `main`. All three
are **not in 57**; they landed after the 57 cut (SDK 58 work). Do not plan the 56 → 57 upgrade around
any of them:
- **iOS UIKit scene lifecycle.** SDK 57 does not ship it. `origin/sdk-57` has no
  `packages/expo/ios/AppDelegates/ExpoAppSceneDelegate.swift`, no `UIApplicationSceneManifest`, and no
  `SceneDelegate.swift` in any template; the AppDelegate keeps its `RCTLinkingManager` overrides. (The
  one `SceneDelegate.swift` on `origin/sdk-57` is an unrelated `expo-updates` e2e fixture,
  `packages/expo-updates/e2e/fixtures/custom_init/`.) #46733 / #46799 sit under `## Unpublished` in
  `main:packages/expo/CHANGELOG.md`.
- **Android R8 / `proguard-android-optimize.txt`.** Not in 57.
  `origin/sdk-57:templates/expo-template-bare-minimum/android/app/build.gradle:119` still reads
  `getDefaultProguardFile("proguard-android.txt")`.
- **A Node `engines` bump.** Not in 57. `expo@57.0.8` has **no `engines` field at all** (nor does
  `expo@56.0.17`); `create-expo@5.0.0` is still `>=18.13.0`; `expo-doctor` is
  `^20.19.4 || ^22.13.0 || ^24.3.0 || >= 25.0.0` on both lines. Verified against the published tarballs,
  not just the branches.

### Tooling minimums (56 → 57)
**Nothing changes.** The Node row is the one people expect to move; it does not.

| | SDK 56 | SDK 57 |
|---|---|---|
| Node — enforced (`react-native` `engines.node`) | `^20.19.4 \|\| ^22.13.0 \|\| ^24.3.0 \|\| >= 25.0.0` | identical |
| Node — docs recommendation | 22.13.x on the release branches; 20.19.x on the live site | 22.13.x (row exists on `main` only) |
| Xcode | 26.2+ on branch / 26.4+ on the live site | same |
| iOS / tvOS | 16.4+ | 16.4+ |
| compileSdk / targetSdk | 36 / 36 | 36 / 36 |
| buildToolsVersion | 36.0.0 | 36.0.0 |
| Android runtime | 7+ | 7+ |
| JDK | 17 | 17 |

Neither release branch's `docs/ui/components/SDKTables/sdk-versions.json` has an SDK 57 row at all —
its top entry is `56.0.0`. The SDK 57 row (Xcode 26.4+, Node 22.13.x, RN 0.86) exists only on `main`.

### Version pins (56 → 57)
Source: `git show origin/sdk-NN:packages/expo/bundledNativeModules.json` — the file that ships inside
the `expo` package and that `npx expo install` / `--fix` resolves against. Package versions for
`expo`, `@expo/cli` and the templates come from their `package.json` on the same branch.

**Do not use `docs/public/static/schemas/vNN.0.0/native-modules.json` for this.** It is stale (it says
`react-native-screens` `4.25.2` and `expo-router` `~56.2.9`, neither of which ships) and
`origin/sdk-57` has no `v57.0.0` schema directory at all.

**Exactly seven third-party packages moved 56 → 57:**

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `react-native` | 0.85.3 | **0.86.0** |
| `react-native-reanimated` | 4.3.1 | **4.5.0** |
| `react-native-worklets` | 0.8.3 | **0.10.0** |
| `react-native-gesture-handler` | ~2.31.1 | **~2.32.0** |
| `react-native-keyboard-controller` | 1.21.6 | **1.21.9** |
| `react-native-pager-view` | 8.0.1 | **8.0.2** |
| `lottie-react-native` | ~7.3.4 | **~7.3.8** |

**Third-party pins that did NOT move** — if a diff or a doc tells you otherwise, it is reading a stale
schema: `react-native-screens` **~4.26.0**, `react-native-safe-area-context` ~5.7.0,
`react-native-svg` 15.15.4, `react-native-webview` 13.16.1, `@shopify/react-native-skia` 2.6.2,
`@shopify/flash-list` 2.0.2, `react-native-web` ~0.21.0, `react-native-maps` 1.27.2,
`@expo/vector-icons` ^15.0.2, `@react-native-async-storage/async-storage` 2.2.0,
`react` / `react-dom` 19.2.3, `typescript` ~6.0.3 (template devDependency).

**First-party pins.** The `expo-*` packages are **not** flat `~57.0.0` — each has its own patch level,
and several are well past `.0`. Sample the ones most likely to be pinned by hand:

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo` | 56.0.17 | **57.0.8** |
| `@expo/cli` | 56.1.21 | **57.0.10** |
| `expo-template-default` | 56.0.31 | **57.0.10** |
| `expo-router` | ~56.2.16 | **~57.0.8** |
| `expo-updates` | ~56.0.23 | **~57.0.10** |
| `expo-dev-client` | ~56.0.24 | **~57.0.9** |
| `expo-video` | ~56.1.4 | **~57.0.2** |
| `@expo/fingerprint` | ~0.19.9 | **~0.20.6** |
| `expo-build-properties` | ~56.0.24 | **~57.0.7** |
| `@expo/ui` | ~56.0.23 | **~57.0.7** |
| `expo-modules-core` | ~56.0.22 | **~57.0.7** |

Do not guess the rest — read `bundledNativeModules.json` on the release branch, or just let
`npx expo install --fix` do it. `react-native-tvos` is not in `bundledNativeModules.json` at all; the
docs table gives 0.85-stable for SDK 56 (present on both release branches) and 0.86-stable for SDK 57
(that row exists only on `main`, so treat it as weaker evidence than everything else in this table).

### Upgrading 56 → 57
Same four steps as §6, with the version substituted. Source:
`docs/pages/workflow/upgrading-expo-sdk-walkthrough.mdx`.

```sh
npm install expo@^57.0.0     # or: npx expo install expo@^57.0.0 --fix
npx expo install --fix
npx expo-doctor
```
Then handle native projects (CNG: delete `ios/` and `android/`; non-CNG: `npx pod-install` + the Native
project upgrade helper), rebuild any `expo-dev-client` development builds, and read
https://expo.dev/changelog/sdk-57. There is no codemod for this hop.

**What this hop actually buys you, in this domain:** React Native 0.86.0, the seven third-party pin
bumps above, `expo-build-properties`' `android.cmakeVersion`, and two 57-only breaking changes you may
have to absorb (`expo-modules-jsi` `JavaScriptError`, `expo-font` `Server.resetServerContext`) plus
prebuild's new clean-by-default. That is the whole list. No Node upgrade, no Xcode upgrade, no scene
lifecycle migration, no ProGuard/R8 change, no codemod, no scaffolding change — and the CLI features in
the SDK 57 changelog are already on the 56 line at the patch versions tabled above.
