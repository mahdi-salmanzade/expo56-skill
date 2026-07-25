# Expo SDK 56 — CLI, Metro Bundler, Bundling & Dev Server

> Knowledge-base reference covering CLI performance, the Metro bundler, on-demand
> filesystem, watcher changes, TypeScript support, and environment variables for
> **Expo SDK 56**.
>
> Compiled: 2026-05-22 · Re-verified 2026-07-25 against the **release branches** `origin/sdk-56`
> (`expo` 56.0.17) and `origin/sdk-57` (`expo` 57.0.8), and against **published npm tarballs**
> (`@expo/cli`, `@expo/metro-config`, `babel-preset-expo`). Pins come from
> `packages/expo/bundledNativeModules.json` on the release branch, **not** from
> `docs/public/static/schemas/*/native-modules.json` (stale) and not from `main` (that is SDK 58).

---

## 0. Gotchas (commonly mis-remembered)

| Claim a model often makes | Reality |
| --- | --- |
| "Tree shaking is on by default in SDK 54+" | **No.** Only `experimentalImportSupport` is on by default. Real unused-import/export removal still needs **both** `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH=1` and `EXPO_UNSTABLE_TREE_SHAKING=1`, production only. See Section 9. |
| "`config.resolver.useWatchman = true` re-enables Watchman" | It does **now**, but was a silent no-op on early patches. Needs `@expo/metro-config` **>= 56.0.17** or **>= 57.0.4**. See Section 3. |
| "`npx expo export --dev` disables minification" | No — `--dev` only configures static files for a non-https local server. Minification is controlled solely by `--no-minify`. See Section 7. |
| "`--localhost` is LAN-only" | `--localhost` is an alias for `--host localhost`; `--lan` is the LAN one. See Section 7. |
| "`npx expo prebuild` is additive by default" | True on SDK 56 — **false on SDK 57**, where prebuild cleans by default. See the SDK 57 delta. |
| "`EXPO_PUBLIC_USE_RN_FETCH=1` works in production" | It does on **expo >= 56.0.13**; broken on earlier 56 patches. This is a 56 patch bump, **not** a reason to go to 57. See Section 6. |
| "You need SDK 57 for `--platform tvos`/`macos`, or for the production `EXPO_PUBLIC_USE_RN_FETCH` fix" | **No.** Both were backported into the SDK 56 line. See Section 12 (SDK 56 patch drift). |
| "SDK 57 raises the minimum Node.js version" | **No.** `NODE_MIN` is still `[20, 19, 4]` in published `@expo/cli` 57.0.10, and `packages/expo/package.json` on `origin/sdk-57` has no `engines` field. |

---

## 1. CLI Performance Improvements (SDK 56)

Source: <https://expo.dev/changelog/sdk-56>

SDK 56 ships a substantially faster bundling pipeline. Reported gains (verbatim from the changelog):

| Metric | Improvement |
| --- | --- |
| `expo start` | **5x faster** |
| Metro crawl | **6x faster** |
| Development memory | **−28%** |
| Cold bundling | **20–50%** faster |
| Warm bundling | **3–8x faster** |

### What enables these gains

These improvements come from several distinct changes working together:

1. **On-demand Filesystem** — removes the up-front filesystem crawl. (Section 2)
2. **Native Node.js watcher & crawler** — replaces Watchman as the default. (Section 3)
3. **Hermes v1 transforms** — "fewer bundler transforms are enabled for Hermes, which reduces bundling times overall." (Section 5)
4. **`import.meta` support enabled automatically** — fewer compatibility transforms. (Section 5)
5. **Replaced TypeScript resolution** — supports TS 6 and prepares for TS 7, also fixing monorepo `tsconfig.json` `paths` bugs. (Section 4)

---

## 2. On-Demand Filesystem (eliminates `watchFolders`)

Source: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/guides/customizing-metro/>

The new **On-demand Filesystem** "eliminates `watchFolders` as a load-bearing configuration option." Instead of crawling and watching whole folder trees up front, the filesystem is resolved lazily / on demand, which is the primary driver of the 6x faster Metro crawl and the reduced development memory footprint.

### Configuration

- **Enabled by default** in SDK 56.
- To **disable** it, add the following to **app.json**:

```json
{
  "expo": {
    "experiments": {
      "onDemandFilesystem": false
    }
  }
}
```

Key naming is settled: it is **`expo.experiments.onDemandFilesystem`** (plural `experiments`), a `boolean` defaulting to `true`. The changelog's `experiment.onDemandFilesystem` phrasing is a typo.

- Typed in `packages/@expo/config-types/src/ExpoConfig.ts` (under `experiments`).
- Read in `packages/@expo/cli/src/start/server/metro/instantiateMetro.ts` — `exp.experiments?.onDemandFilesystem ?? true`, then copied to the internal Metro key `resolver.unstable_onDemandFilesystem`.
- Caveat: the key is **not** present in the versioned app-config reference for either v56 or v57 (`docs/pages/versions/v56.0.0/config/app.mdx`, `v57.0.0/config/app.mdx`), so the Metro guide's link to `/versions/v56.0.0/config/app/#ondemandfilesystem` is a dead anchor. The prose lives at `docs/pages/versions/v56.0.0/config/metro.mdx:388-406` instead.
- Undocumented escape hatch: the *internal* `resolver.unstable_onDemandFilesystem` also accepts the string `'UNSTABLE_ALLOW_ALL'`, which lets the fallback filesystem read outside the server root (`packages/@expo/cli/src/start/server/metro/createFileMap-fork.ts:59-67`, identical at both SDK cuts). The app-config flag is boolean-only.

### Practical effect

- You generally no longer need to manually configure `resolver`/`watchFolders` in `metro.config.js` for monorepos — the on-demand filesystem lazily reads files outside `watchFolders` as the resolver requests them. It also allows symlinks to resolve outside the monorepo root (Global Virtual Store support for Bun/pnpm).
- `watchFolders` still exists as a Metro config key but is no longer "load-bearing" (no longer required for correctness). The concrete, checkable meaning: `expo-doctor`'s `MetroConfigCheck` **skips** the `"watchFolders" does not contain all entries from Expo's defaults` assertion on SDK >= 56 unless `experiments.onDemandFilesystem === false` (`packages/expo-doctor/src/checks/MetroConfigCheck.ts:57-69`).
- Trade-off: fewer `watchFolders` means less file-watcher coverage. Lazily-read files outside the watch set are not watched for changes.

---

## 3. Native Node.js Watcher (replaces Watchman)

Source: <https://expo.dev/changelog/sdk-56>

SDK 56 uses **"a native Node.js watcher and crawler by default"** instead of Watchman. This removes the Watchman dependency for most workflows and contributes to the faster crawl and reduced memory usage.

### Switching back to Watchman — works, but only on recent patches

The SDK 56 changelog says "you can switch back to Watchman with `resolver.useWatchman` in a Metro config." On **early** SDK 56 and SDK 57 patches this did not work: setting `config.resolver.useWatchman = true` was a silent no-op — Watchman was never contacted (`watchman watch-list` showed no roots), the Node watcher was used anyway, and nothing was logged.

Root cause (two collisions):

1. `@expo/metro-config`'s `loadUserConfig` rewrote the **merged** config's `useWatchman === true` to `null` — post-merge it could not distinguish Metro's own default `true` from the user's explicit `true`.
2. Expo CLI's file-map fork then coalesced the nullish value: `useWatchman: config.resolver.useWatchman ?? false` (`packages/@expo/cli/src/start/server/metro/createFileMap-fork.ts`).

Net: `true → null → ?? false → false`.

**Fixed** (expo/expo#47662): `getDefaultConfig` now neutralises Metro's default at source (`metroConfig.resolver.useWatchman = null` in `ExpoMetroConfig.ts`) and the post-merge rewrite in `loadUserConfig` is gone, so an explicit `= true` survives to the file map.

| Line | First fixed `@expo/metro-config` | Changelog date |
| --- | --- | --- |
| SDK 56 | **56.0.17** | 2026-07-15 |
| SDK 57 | **57.0.4** | 2026-07-15 |

Verified in the published tarballs: 56.0.14–56.0.16 and 57.0.0–57.0.3 still contain the `loadUserConfig` rewrite; 56.0.17+/57.0.4+ contain `metroConfig.resolver.useWatchman = null` in `getDefaultConfig` instead. Note the CLI side still reads `?? false`, which is correct — `true ?? false` is `true`.

```js
// metro.config.js — works on @expo/metro-config >= 56.0.17 / >= 57.0.4
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);
config.resolver.useWatchman = true;

module.exports = config;
```

Check what you actually resolved with `npm ls @expo/metro-config` — the `expo` package pins it with a tilde range, so a stale lockfile can hold you on a pre-fix build. On a pre-fix build the workaround is a **truthy non-`true`** value (`config.resolver.useWatchman = 1`), which slips past the `=== true` rewrite and survives the `?? false`.

### Why you might still want Watchman

The Node watcher opens one OS handle per directory, which exhausts the handle pool (`EMFILE`) on Windows monorepos, and crashes with `ENOENT` on directories deleted mid-build (e.g. Gradle output). Watchman uses a single recursive watch per root and avoids both. Source: expo/expo#47662.

- Config key: **`resolver.useWatchman`** (`boolean | null`; Expo's default is effectively `false` via the `?? false` coalesce)
- Setting it to `false` explicitly is fine and is what the E2E fixtures do.

---

## 4. TypeScript 6 Support & TypeScript 7 Readiness

Source: <https://expo.dev/changelog/sdk-56>

- **"TypeScript 6 support and TypeScript 7 readiness: we replaced our TypeScript resolution to support TS 6 and prepare for TS 7."**
- The replaced TypeScript resolution also resolves monorepo `tsconfig.json` `paths` configuration bugs.
- TypeScript path mapping continues to be driven by `compilerOptions.paths` and `compilerOptions.baseUrl` in `tsconfig.json`.

### Actionable config

Source: `docs/pages/guides/typescript.mdx:145-215`

- Path aliases are gated by the app-config flag **`expo.experiments.tsconfigPaths`**, **enabled by default**. Disable with:

```json
{ "expo": { "experiments": { "tsconfigPaths": false } } }
```

- `compilerOptions.paths` resolve relative to `compilerOptions.baseUrl` when it is set, otherwise against the project root. `baseUrl` is resolved *before* node modules, so a local `./path.ts` can shadow the `path` node module.
- After editing `tsconfig.json` aliases you must **restart Expo CLI**, but you do **not** need to clear the Metro cache.
- **jsconfig.json** is a valid substitute in non-TypeScript projects.
- Aliases are Metro-only (including Metro web). Bare React Native projects need the extra Metro setup described in `versions/latest/config/metro#existing-react-native-apps`.
- `EXPO_NO_TYPESCRIPT_SETUP=1` stops the CLI from forcing TypeScript setup on `npx expo start` (`docs/pages/more/expo-cli.mdx`, env var table).
- The template pins `typescript` at `~6.0.3` — identical on `origin/sdk-56` and `origin/sdk-57` (`templates/expo-template-default/package.json`). TypeScript is **not** in `packages/expo/bundledNativeModules.json` on either branch, so `expo install typescript` will not manage it: treat the pin as template-observed, not SDK-enforced.

---

## 5. `import.meta` Support & Hermes v1 Transforms

Source: <https://expo.dev/changelog/sdk-56>

- **`import.meta` support: "now enabled automatically."** No manual configuration required.
- **Hermes v1 transforms: "fewer bundler transforms are enabled for Hermes, which reduces bundling times overall."** Fewer transforms = faster bundling, contributing to the warm/cold bundling speedups.

### The mechanism (and how to opt out)

Source: `packages/babel-preset-expo/src/index.ts:71-79`, `src/configs/expo.ts:149-150`, `src/plugins/import-meta-transform-plugin.ts:13-33`

`babel-preset-expo` accepts **`transformImportMeta`** (`boolean`, `@default true`). When on, the `expo-import-meta-transform` plugin rewrites every `import.meta` meta-property to `globalThis.__ExpoImportMetaRegistry`. Turn it off in **babel.config.js**:

```js
module.exports = function (api) {
  api.cache(true);
  return { presets: [['babel-preset-expo', { transformImportMeta: false }]] };
};
```

- The option's own doc comment warns the transform "may interfere with the native implementation" on engines that support `import.meta` natively — that is the only reason to disable it.
- With it disabled, using `import.meta` on a non-web platform throws at build time: `` `import.meta` is not supported in Hermes. Enable the polyfill `transformImportMeta` in babel-preset-expo to use this syntax. `` On web it is left untouched instead of throwing.
- Unchanged in SDK 57.

---

## 6. `expo/fetch` as Default `globalThis.fetch`

Source: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/guides/environment-variables/>

SDK 56 installs **`expo/fetch` as the default implementation of `globalThis.fetch`**:

- Providing a **WinterCG**-compliant API and improved performance. Manual imports are no longer required. (The versioned doc says "WinterCG", not "WinterTC" — `docs/pages/versions/v56.0.0/sdk/expo.mdx:28,49`.)
- You no longer need to `import { fetch } from 'expo/fetch'` — the global `fetch` already uses it. Named imports from `expo/fetch` keep working regardless of the flag below.
- **Native only.** The swap lives in `packages/expo/src/winter/runtime.native.ts:37-51` (Android/iOS). The web runtime (`runtime.ts`) does not touch `fetch`.

### Opting out (env var)

To opt out and restore the React Native default `fetch`:

```sh
EXPO_PUBLIC_USE_RN_FETCH=1
```

- Env var: **`EXPO_PUBLIC_USE_RN_FETCH`** — opts out of the new `expo/fetch` default `globalThis.fetch` implementation.
- Accepted values are exactly `'1'` or `'true'` (`runtime.native.ts:37-38`) — not any truthy string.
- **Patch caveat: on early SDK 56 patches this opt-out was development-only.** `EXPO_PUBLIC_` variables are deliberately not inlined inside `node_modules`, and the check lives in `expo`'s own runtime, so in a production bundle `process.env.EXPO_PUBLIC_USE_RN_FETCH` was never substituted and the opt-out silently failed — production kept `expo/fetch` as `globalThis.fetch`. (expo/expo#46982.)
- **Fixed on the SDK 56 line, not only in 57.** `babel-preset-expo` special-cases this one variable so it is inlined inside `node_modules` too (`configs/expo.ts` — `if (options.isNodeModule && process.env.EXPO_PUBLIC_USE_RN_FETCH != null)`, #46986). Landed in `babel-preset-expo` **56.0.16** (and 57.x). That is bundled by **`expo` >= 56.0.13**. Byte-identical on `origin/sdk-56` and `origin/sdk-57` — do not upgrade to 57 for this. See Section 12.

---

## 7. Expo CLI Commands & Flags (bundling / dev server)

Source: <https://docs.expo.dev/more/expo-cli/>; flag text verified verbatim against `packages/@expo/cli/src/*/index.ts` at the SDK 56 cut. Unchanged in SDK 57 except where noted.

### `npx expo start`

Starts the Metro dev server (default `http://localhost:8081`). Help text: `packages/@expo/cli/src/start/index.ts:47-80`.

| Flag | Description |
| --- | --- |
| `-a, --android` / `-i, --ios` / `-w, --web` | Open on a connected Android device / iOS simulator / web browser |
| `-d, --dev-client` | Launch in a custom native app (development build) |
| `-g, --go` | Launch in Expo Go |
| `-c, --clear` (alias `--reset-cache`) | Clear the bundler cache |
| `--max-workers <number>` | Maximum number of tasks to allow Metro to spawn |
| `--no-dev` | Bundle in production mode |
| `--minify` | Minify JavaScript |
| `-m, --host <string>` | Dev server hosting type: `lan` (default) / `tunnel` / `localhost` |
| `--tunnel` / `--lan` / `--localhost` | Exact aliases for `--host tunnel` / `--host lan` / `--host localhost` |
| `--offline` | Skip network requests and use anonymous manifest signatures |
| `--https` | **Deprecated** in favour of `--tunnel` |
| `--scheme <scheme>` | Custom URI protocol to use when launching an app |
| `-p, --port <number>` | Dev server port, default `8081` (does not apply to web or tunnel). `--port 0` means "pick any available port" and scans from the fallback without prompting — `packages/@expo/cli/src/utils/port.ts` |
| `--private-key-path <path>` | Private key for code signing |

> `--localhost` is **not** "LAN-only" — it is the localhost-only mode. `--lan` is the LAN mode.

### `npx expo export`

Bundles JS and assets for **production** (strips code guarded by the `__DEV__` boolean). Help text: `packages/@expo/cli/src/export/index.ts` (`printHelp`); option derivation: `src/export/resolveOptions.ts` (`resolveOptionsAsync`).

| Flag | Description |
| --- | --- |
| `-p, --platform <platform>` | Options: `android`, `ios`, `web`, `all` (default `all`), plus experimental `tvos` and `macos`. `knownPlatforms` in `src/export/resolveOptions.ts:58` is **identical on `origin/sdk-56` and `origin/sdk-57`** — this is not a 57 feature; it needs `expo` >= 56.0.13 and `experiments.outOfTreePlatforms` |
| `--output-dir <dir>` | Export destination (default `dist`) |
| `--dev` | Configure static files for developing locally using a non-https server. **Does not affect minification** |
| `--no-minify` | Prevent minifying source — the *only* minification control (`minify: !args['--no-minify']`) |
| `--no-bytecode` | Prevent generating Hermes bytecode |
| `--dump-assetmap` | Emit an asset map for further processing |
| `--no-ssg`, `--api-only` | Skip exporting static HTML files and only export API routes for web |
| `--max-workers <number>` | Limit bundler worker tasks |
| `-s, --source-maps [mode]` | Emit JS source maps. Options: `true`, `false`, `inline`, `external` (default `false`). `--dump-sourcemap` is the deprecated alias |
| `-c, --clear` (alias `--reset-cache`) | Clear the bundler cache |

### `npx expo run:ios`

Compiles the native app locally (runs prebuild if native dirs are missing). Help text: `packages/@expo/cli/src/run/ios/index.ts:12-56`.

| Flag | Description |
| --- | --- |
| `--no-build-cache` | Clear the native derived data before building |
| `--no-install` | Skip installing dependencies |
| `--no-bundler` | Skip starting the Metro bundler |
| `--scheme [scheme]` | Scheme to build |
| `--binary <path>` | Path to an existing **.app** or **.ipa** to install |
| `--configuration <configuration>` | Xcode configuration: `Debug` (default) or `Release` |
| `-d, --device [device]` | Device name, UDID, or `generic` for build-only |
| `-o, --output <path>` | Directory to output the built app binary |
| `-p, --port <port>` | Metro port (default `8081`) |

### `npx expo run:android`

Help text: `packages/@expo/cli/src/run/android/index.ts:12-58`. **There is no `--output`/`-o` flag on `run:android`** — it exists only on `run:ios`.

| Flag | Description |
| --- | --- |
| `--no-build-cache` | Clear the native build cache |
| `--no-install` | Skip installing dependencies |
| `--no-bundler` | Skip starting the bundler |
| `--app-id <appId>` | Custom Android application ID to launch |
| `--variant <name>` | Build variant / product flavor + variant (default `debug`) |
| `--binary <path>` | Path to an existing **.apk** or **.aab** to install |
| `-d, --device [device]` | Device name to run the app on |
| `-p, --port <port>` | Dev server port (default `8081`) |
| `--all-arch` | Undocumented; disables the active-archs-only behaviour |

### `npx expo prebuild`

**SDK 56 behaviour:** additive by default; `--clean` deletes and regenerates **ios/** and **android/** first. This inverted in SDK 57 — see the SDK 57 delta before writing any prebuild instructions.

---

## 8. `metro.config.js` Keys

Sources: <https://docs.expo.dev/guides/customizing-metro/>, <https://docs.expo.dev/guides/tree-shaking/>

Base config is always created via `expo/metro-config`:

```js
const { getDefaultConfig } = require('expo/metro-config');
const config = getDefaultConfig(__dirname);
module.exports = config;
```

| Key | Purpose |
| --- | --- |
| `resolver.sourceExts` | Source file extensions (JS, TS, JSON) |
| `resolver.assetExts` | Asset file extensions (images, fonts, etc.) |
| `resolver.resolveRequest` | Custom resolver (module redirects/aliases) |
| `resolver.resolverMainFields` | package.json fields to try. Expo forces `['browser','module','main']` on web; set `EXPO_METRO_NO_MAIN_FIELD_OVERRIDE=1` to use your value on all platforms |
| `resolver.blockList` | Exclude paths. `resolver.blacklistRE` is **deprecated** and flagged by `expo-doctor`; `g`/`y` regex flags in `blockList` are also flagged (`packages/expo-doctor/src/checks/MetroConfigCheck.ts:70-90`) |
| `resolver.useWatchman` | Watchman opt-in. Works on `@expo/metro-config` >= 56.0.17 / >= 57.0.4; silent no-op below that (Section 3) |
| `server.unstable_serverRoot` | Metro server root; auto-detected as the workspace root in monorepos. Disable detection with `EXPO_NO_METRO_WORKSPACE_ROOT=1` |
| `transformer.assetPlugins` | Extra asset transform plugins |
| `transformer.getTransformOptions` | Returns transform flags |
| `transformer.getTransformOptions → transform.experimentalImportSupport` | ESM→CJS import support. Default `true` since SDK 54 (`packages/@expo/metro-config/src/ExpoMetroConfig.ts`, `getDefaultConfig`). A **prerequisite for**, not an enabler of, tree shaking — see Section 9 |
| `transformer.getTransformOptions → transform.inlineRequires` | Inline requires for smaller bundles. Expo's default is **`false`** |
| `watchFolders` | Additional watched folders (no longer load-bearing in SDK 56 — Section 2) |

App-level (in **app.json**, not `metro.config.js`):

| Key | Purpose |
| --- | --- |
| `expo.experiments.onDemandFilesystem` | Toggle on-demand filesystem (default `true` in SDK 56) |
| `expo.experiments.tsconfigPaths` | Toggle `tsconfig.json` path aliases (default `true`) |
| `expo.experiments.autolinkingModuleResolution` | Apply autolinking results to Metro resolution. Defaults to `isWorkspace \|\| targetsOutOfTreePlatform` — i.e. on in a monorepo **or** when targeting `tvos`/`macos` (`instantiateMetro.ts`; identical on both release branches) |
| `expo.experiments.outOfTreePlatforms` | Experimental `tvos` / `macos` support, if the support packages are installed. Present on both lines (`@expo/config-types`, `expo` >= 56.0.13) |
| `expo.web.bundler: "metro"` | Enable Metro web bundling |

> Always `require('expo/metro-config')`, never `@expo/metro-config` directly — `expo/metro-config` re-exports it and is guaranteed present. `npx expo customize metro.config.js` stopped installing `@expo/metro-config` as a dependency in `@expo/cli` **56.1.14** (`expo` >= 56.0.9), so a project importing `@expo/metro-config` directly must add it to `devDependencies` by hand. That is an SDK 56 patch change, not a 57 one — `customize/templates.ts` has `dependencies: []` on both release branches.

---

## 9. Tree Shaking

Source: `docs/pages/guides/tree-shaking.mdx`; gating verified in `packages/@expo/cli/src/start/server/middleware/metroOptions.ts:100-110` and `src/start/server/metro/instantiateMetro.ts` (`getExpoMetroConfig` warnings block). Identical at the SDK 56 and SDK 57 cuts.

> **Unused import/export removal is NOT on by default in SDK 56 (or SDK 57).** It is still marked "Experimentally available in SDK 52 and later" and requires two env vars. What became default in SDK 54 is only step 1 of the enable flow.

### What IS always on in production

These need no configuration and run on every production bundle:

- **Platform shaking** — per-platform bundles; `Platform.OS` / `Platform.select` conditionals are removed for other platforms, but only when `Platform` is imported directly from `react-native` in that file (re-exports defeat it).
- **`__DEV__` and `process.env.NODE_ENV` constant folding** — dev-only branches are dropped.
- **`EXPO_PUBLIC_*` inlining** (Section 10) and `process.env.EXPO_OS`.
- **`react-native-web` barrel-file optimization** (web, production): `import { View } from 'react-native'` is rewritten to `react-native-web/dist/exports/View`. `require()` defeats it.

### What is NOT always on (commonly mis-grouped)

- **Server-code removal** — the `typeof window === 'undefined'` transform is **not** unconditional. `babel-preset-expo` computes `const minifyTypeofWindow = options.minifyTypeofWindow ?? options.isServerEnv` (`src/configs/expo.ts`; verified in the published `babel-preset-expo@57.0.4` build at line 221), so it is enabled automatically **only** when bundling for a server environment (API routes, SSR). For client bundles — native *and* web — it is off by default; opt in with `['babel-preset-expo', { minifyTypeofWindow: true }]`. The transform runs in dev and prod but only strips the dead branch in production. It stays off for web by default because web workers have no `window` global.

### What requires opting in

| Step | Setting | Notes |
| --- | --- | --- |
| 1 | `transformer.getTransformOptions → transform.experimentalImportSupport: true` | **Default `true` since SDK 54**, so nothing to do on SDK 56/57. Prerequisite only |
| 2 | `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH=1` | Eager bundling: keeps modules until the whole graph exists before optimizing. Production only |
| 3 | `EXPO_UNSTABLE_TREE_SHAKING=1` | Actually removes unused imports/exports. Production only |
| 4 | `npx expo export` | Bundle in production mode to observe the effect |

Gating in source:

```ts
// metroOptions.ts
const optimize = props.optimize ?? (environment !== 'node' && mode === 'production' && env.EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH);
// ...
usedExports: optimize && env.EXPO_UNSTABLE_TREE_SHAKING,
```

Consequences:

- Setting `EXPO_UNSTABLE_TREE_SHAKING` alone is a **hard error**: `EXPO_UNSTABLE_TREE_SHAKING requires EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH to be enabled.`
- Both are ignored outside `mode === 'production'` and for the `node` environment.
- The CLI logs `Experimental bundle optimization is enabled.` and `Experimental tree shaking is enabled.` (suppressed when logs are reduced). If you do not see these, tree shaking is not running.

### Limitations

- Only modules using `import`/`export` syntax are shaken; `require`/`module.exports` files are not. Do **not** add `@babel/plugin-transform-modules-commonjs` — it breaks shaking project-wide.
- Modules marked as having side effects are kept.
- `export * from '...'` is expanded and optimized unless the target uses `module.exports`/`exports`.
- All Expo SDK modules ship as ESM and are exhaustively shakeable.
- `transform.inlineRequires: true` is an independent, optional size lever (Expo's default is `false`).

---

## 10. Environment Variables Reference

Source: <https://docs.expo.dev/guides/environment-variables/>, <https://expo.dev/changelog/sdk-56>

### `EXPO_PUBLIC_` prefix

- Variables prefixed `EXPO_PUBLIC_` are loaded from **.env** files and inlined into the client JS bundle.
- Must be referenced statically with dot notation: `process.env.EXPO_PUBLIC_KEY` (bracket notation and destructuring are NOT inlined).
- Editable without restarting the CLI / clearing cache, but a full reload is needed to see updated values.
- **Security:** never store secrets in `EXPO_PUBLIC_` vars — they appear in plain text in the compiled app. `node_modules` code is not inlined.

### .env file resolution

- `.env` (committable defaults)
- `.env.local` (machine-specific; gitignore)
- plus standard `.env.*` variants, loaded by conventional priority order.
- EAS Build and EAS Update use the same Metro inlining mechanism.

### Bundling-relevant CLI env vars

The canonical table is `docs/pages/more/expo-cli.mdx` → "## Environment variables" (~45 entries). The subset that affects bundling / the dev server:

| Env var | Purpose |
| --- | --- |
| `EXPO_PUBLIC_USE_RN_FETCH` | Opt out of `expo/fetch` as `globalThis.fetch`. Values `1` or `true`. Dev-only below `expo` 56.0.13; works in production from 56.0.13 / any 57 (Section 6) |
| `EXPO_NO_DOTENV` | Prevent all `.env` file loading across Expo CLI |
| `EXPO_NO_CLIENT_ENV_VARS` | Prevent inlining `EXPO_PUBLIC_` vars in client bundles |
| `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH` | Eager bundling; required for production tree shaking (SDK 52+, experimental) |
| `EXPO_UNSTABLE_TREE_SHAKING` | Unused import/export removal (SDK 52+, experimental; requires the above) |
| `EXPO_UNSTABLE_LIVE_BINDINGS` | Live bindings in experimental import/export support are **on by default** (SDK 54+) — `boolish('EXPO_UNSTABLE_LIVE_BINDINGS', true)` in `src/utils/env.ts`, same on both release branches. Set it to `0`/`false` to **disable** them (worse circular-dependency handling, marginally faster). Setting it to `1` is a no-op |
| `EXPO_UNSTABLE_LOG_BOX` | Experimental LogBox error overlay for native (SDK 55+; already default on web) |
| `EXPO_METRO_NO_MAIN_FIELD_OVERRIDE` | Use your `resolver.resolverMainFields` on all platforms instead of Expo forcing `['browser','module','main']` on web |
| `EXPO_NO_METRO_WORKSPACE_ROOT` | Disable auto workspace-root detection for the Metro server root (SDK 52+). `EXPO_USE_METRO_WORKSPACE_ROOT` is deprecated |
| `EXPO_NO_METRO_LAZY` | Drop `lazy=true` from Metro URLs; disables `import()` support |
| `EXPO_USE_METRO_REQUIRE` | Expo's custom Metro `require` + string module IDs (SDK 52+). No legacy RAM bundle support |
| `EXPO_NO_BUNDLE_SPLITTING` | Disable chunk splitting on async imports in production (web only, experimental) |
| `EXPO_ATLAS` | Gather Metro bundle info during dev/export (SDK 53+). `EXPO_UNSTABLE_ATLAS` is the deprecated name |
| `EXPO_PUBLIC_FOLDER` | Public directory for Metro web (default `public`) |
| `EXPO_NO_TYPESCRIPT_SETUP` | Stop the CLI forcing TypeScript setup on `npx expo start` |
| `EXPO_NO_DEPENDENCY_VALIDATION` | Disable dependency validation in `npx expo install` / `npx expo start` (SDK 52+) |
| `EXPO_NO_REACT_NATIVE_WEB` | **Deprecated SDK 56+.** Experimental mode where `react-native-web` isn't required on web |
| `EXPO_NO_GIT_STATUS` | Skip the git-status warning before dangerous actions like `npx expo prebuild --clean`. Far more relevant on SDK 57, where every prebuild is the destructive variant |
| `EXPO_OFFLINE` | Skip network requests where possible |
| `EXPO_NO_CACHE` | Disable all global caching (`~/.expo`) |
| `EXPO_DEBUG` / `DEBUG=expo:*` | CLI debug logs |

---

## 11. Filesystem / `src/` Directory (Expo Router)

Source: <https://docs.expo.dev/router/reference/src-directory/>

- **`src/app` takes higher precedence than the root `app` directory.** If both exist, only `src/app` is used. Static rendering also prefers `src/app`.
- Customize the routes root via the `expo-router` config plugin in **app.json**:

```json
{ "plugins": [["expo-router", { "root": "./src/routes" }]] }
```

- Keep TS path aliases in sync: `"paths": { "@/*": ["./src/*"] }` in `tsconfig.json`.
- **Note:** The `src/` directory docs do **not** reference SDK 56 on-demand filesystem or `watchFolders` changes — these are independent features. The on-demand filesystem (Section 2) is what removed the `watchFolders` requirement, not the router's `src/` resolution.

---

## 12. SDK 56 patch drift (things you get without upgrading to 57)

SDK 56 is still actively patched (`expo` 56.0.17 as of 2026-07-25). Several changes that read like "SDK 57 features" were backported into the 56 line and are **byte-identical on `origin/sdk-56` and `origin/sdk-57`**. If you are on SDK 56, bump the patch instead of the major.

| Change | Landed in | Minimum `expo` on the 56 line |
| --- | --- | --- |
| `npx expo customize metro.config.js` no longer installs `@expo/metro-config` (#46600) | `@expo/cli` 56.1.14 | **56.0.9** |
| Experimental `tvos`/`macos` out-of-tree platforms: `knownPlatforms` accepts them, `experiments.outOfTreePlatforms` exists, `autolinkingModuleResolution` defaults on for out-of-tree targets (#46344) | `@expo/cli` 56.1.17 | **56.0.13** |
| Dynamic `import()` with a rejection handler treated as an optional dependency — `import('x').catch(...)` no longer fails the graph when `x` is missing (ported from metro#1697, #47334) | `@expo/metro-config` 56.0.15 | **56.0.13** |
| `EXPO_PUBLIC_USE_RN_FETCH` inlined inside `node_modules`, so the `expo/fetch` opt-out works in production (#46986) | `babel-preset-expo` 56.0.16 | **56.0.13** |
| `resolver.useWatchman: true` actually re-enables Watchman (#47662) | `@expo/metro-config` 56.0.17 | **56.0.16** |
| `composeSourceMaps` no longer crashes on Hermes segments with a negative original position (#47752) | `@expo/metro-config` 56.0.18 | **56.0.17** |

Net: **`expo@56.0.16`** (or later) gets you every bundling/CLI item above except the one 56.0.17 source-map fix. The one thing on this page that a 56 patch cannot give you is the `expo prebuild` default inversion, and that is a *regression* for most workflows, not a feature.

Because `expo` pins these with tilde ranges, a fresh install often resolves higher than the pin implies while a stale lockfile does not. Confirm with `npm ls @expo/cli @expo/metro-config babel-preset-expo` rather than reading `expo`'s version alone.

---

## Source URLs

- CLI Performance / SDK 56 changes: <https://expo.dev/changelog/sdk-56>
- Expo CLI commands & flags: <https://docs.expo.dev/more/expo-cli/>
- Customizing Metro: <https://docs.expo.dev/guides/customizing-metro/>
- Tree shaking: <https://docs.expo.dev/guides/tree-shaking/>
- Environment variables: <https://docs.expo.dev/guides/environment-variables/>
- `src/` directory: <https://docs.expo.dev/router/reference/src-directory/>

---

## SDK 57 delta

For this domain, SDK 57 is a **small** delta. Exactly one breaking change matters (`npx expo prebuild` inverts its default) plus two DevTools-plugin API changes. The Metro *config surface itself* is unchanged, and the `expo start` / `expo export` flag sets are identical. Most of what looks like a 57 feature was backported into the SDK 56 line — see Section 12 before planning an upgrade around it.

Verification method: `git show origin/sdk-56:<path>` vs `git show origin/sdk-57:<path>` on the **release branches**, cross-checked against published npm tarballs (`@expo/cli@57.0.10`, `@expo/metro-config@57.0.7`, `babel-preset-expo@57.0.4` — the versions `expo@57.0.8` resolves). `main` was not used: it is SDK 58 in progress. Anything under a `## Unpublished` changelog heading was ignored entirely.

### Breaking

**`npx expo prebuild` now CLEANS by default.**

`packages/@expo/cli/src/prebuild/index.ts` — `clean: args['--clean']` became `clean: !args['--no-clean']` (#47209, `@expo/cli` 57.0.0; root `CHANGELOG.md`, 57.0.0 → Breaking → `@expo/cli`).

| SDK 56 | SDK 57 |
| --- | --- |
| `npx expo prebuild` — additive, applies changes to existing **ios/** and **android/** | `npx expo prebuild` — **deletes and regenerates** **ios/** and **android/** |
| `--clean` — "Delete the native folders and regenerate them before applying changes" | `--no-clean` — "Apply changes to the existing native folders instead of recreating them" |

`--clean` is still accepted by the arg spec in 57 but its value is never read: it is a **silent no-op**. Any local edits to **ios/** or **android/** are lost on a bare `npx expo prebuild`, which makes `EXPO_NO_GIT_STATUS` (Section 10) and a clean working tree far more important. Confirmed in the published `@expo/cli@57.0.10` build (`clean: !args['--no-clean']`) and confirmed *absent* from `@expo/cli@56.1.21` (`clean: args['--clean']`), so this genuinely does not exist anywhere on the 56 line.

### New in 57 (verified absent from `origin/sdk-56`)

- **DevTools plugin WebSocket handlers receive a fetch-API `Request`** instead of a Node `IncomingMessage` — read `request.url` / `request.headers` (#47410, `@expo/cli` 57.0.5, i.e. `expo` >= 57.0.3).
- **MCP and DevTools plugin server modules are loaded via `loadModule`** (#47139, same `@expo/cli` 57.0.5 release).

That is the complete list for this domain. If you are on SDK 56 and do not author a DevTools plugin, nothing in "New in 57" is a reason to upgrade.

### Claims to NOT make about SDK 57 (verified false)

These were previously attributed to SDK 57 and are wrong. They are either SDK 58 work on `main` or never existed.

- **No Node.js minimum bump.** `NODE_MIN` is still `[20, 19, 4]` in `origin/sdk-57:packages/@expo/cli/bin/cli.ts` and in the published `@expo/cli@57.0.10` build. `origin/sdk-57:packages/expo/package.json` has **no `engines` field**, and neither does `@expo/cli`'s. Node >= 20.19.4 is still fine on SDK 57. (The `^22.13.0 || ^24.3.0 || ...` engines range is post-57 work.)
- **`babel-preset-expo` does not gain `ios`/`android`/`macos`/`tvos` per-platform option keys.** Both release branches and the published `babel-preset-expo@57.0.4` typings expose only `web?` and `native?`.
- **`expo login` does not default to browser-based login.** `src/login/index.ts` is identical on both branches (`browser: !!args['--browser']`), and the published `@expo/cli@57.0.10` build has no `--no-browser` flag.
- **`npx expo customize metro.config.js` dropping `@expo/metro-config`** is not a 57 change — `customize/templates.ts` has `dependencies: []` on **both** branches. It landed in `@expo/cli` 56.1.14. See Section 12.

### Unchanged in 57 (checked, so you don't have to)

- **Tree shaking gating is identical** — still `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH` + `EXPO_UNSTABLE_TREE_SHAKING`, production only, still "Experimentally available in SDK 52 and later". Section 9 applies verbatim to 57.
- **`resolver.useWatchman = true`** behaves the same on both lines: broken below `@expo/metro-config` 56.0.17 / 57.0.4, working at or above. Section 3 applies verbatim.
- **`npx expo start` and `npx expo export` flag sets are unchanged**, including `--platform` accepting `tvos`/`macos` (which 56 already accepts).
- **`experiments.onDemandFilesystem`, `experiments.tsconfigPaths`, `experiments.outOfTreePlatforms`, `experiments.autolinkingModuleResolution`, `transformImportMeta`** all behave identically; the `instantiateMetro.ts` defaults are line-for-line the same.
- `expo/fetch` is still WinterCG-compliant and still the native `globalThis.fetch`.

### Version pins (56 → 57)

Source: `git show origin/sdk-56:packages/expo/bundledNativeModules.json` and the `origin/sdk-57` equivalent — the file that ships inside the `expo` package and that `expo install` resolves against. Do **not** use `docs/public/static/schemas/*/native-modules.json`; it is stale.

| Package | SDK 56 | SDK 57 |
| --- | --- | --- |
| `react-native` | `0.85.3` | `0.86.0` |
| `@expo/metro-runtime` | `~56.0.18` | `~57.0.7` |

`@expo/cli`, `@expo/metro-config`, `babel-preset-expo` and `@expo/config-types` are **not** in `bundledNativeModules.json` — they ship transitively with `expo`. As of 2026-07-25: `expo@56.0.17` → `@expo/cli ^56.1.21`, `@expo/metro-config ~56.0.18`, `babel-preset-expo ~56.0.18`; `expo@57.0.8` → `@expo/cli ^57.0.10`, `@expo/metro-config ~57.0.7`, `babel-preset-expo ~57.0.4`.

> Post-57, in **neither** shipped line as of 2026-07-25 — do not attribute these to SDK 57 *or* to a 56 patch: `woff`/`woff2` in default `assetExts` (#47565), `pageHeaders` in exported route manifests (#47429), and the direct `@react-native/js-polyfills` dependency (#48034). They live under `## Unpublished` on `main`, which is SDK 58.
>
> Conversely, `resolver.useWatchman: true` (#47662), `expo serve` `headers` (#47780) and the `composeSourceMaps` negative-position fix (#47752) **did** ship — on **both** the 56 and 57 lines. Check `origin/sdk-56` and `origin/sdk-57` changelogs, never `main`.
