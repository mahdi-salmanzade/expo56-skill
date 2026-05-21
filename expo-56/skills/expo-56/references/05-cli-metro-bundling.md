# Expo SDK 56 — CLI, Metro Bundler, Bundling & Dev Server

> Knowledge-base reference covering CLI performance, the Metro bundler, on-demand
> filesystem, watcher changes, TypeScript support, and environment variables for
> **Expo SDK 56**.
>
> Compiled: 2026-05-22

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

> Note on key naming: the changelog text phrases the toggle as `experiment.onDemandFilesystem: false`; the Metro/Expo config docs reference it as `experiments.onDemandFilesystem` (plural `experiments`, matching the existing `expo.experiments` config block). Use the `experiments` (plural) form under `expo` in **app.json**.

### Practical effect

- You generally no longer need to manually configure `resolver`/`watchFolders` in `metro.config.js` for monorepos — the on-demand filesystem handles resolution without an explicit watch list.
- `watchFolders` still exists as a Metro config key but is no longer "load-bearing" (no longer required for correctness).

---

## 3. Native Node.js Watcher (replaces Watchman)

Source: <https://expo.dev/changelog/sdk-56>

SDK 56 uses **"a native Node.js watcher and crawler by default"** instead of Watchman. This removes the Watchman dependency for most workflows and contributes to the faster crawl and reduced memory usage.

### Switching back to Watchman

You can revert to Watchman via the Metro config key `resolver.useWatchman` (no longer recommended):

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

config.resolver.useWatchman = true; // revert to Watchman — no longer recommended

module.exports = config;
```

- Config key: **`resolver.useWatchman`**
- Status: "you can switch back to Watchman with `resolver.useWatchman` in a Metro config, but it's no longer recommended."

---

## 4. TypeScript 6 Support & TypeScript 7 Readiness

Source: <https://expo.dev/changelog/sdk-56>

- **"TypeScript 6 support and TypeScript 7 readiness: we replaced our TypeScript resolution to support TS 6 and prepare for TS 7."**
- The replaced TypeScript resolution also resolves monorepo `tsconfig.json` `paths` configuration bugs.
- TypeScript path mapping continues to be driven by `compilerOptions.paths` and `compilerOptions.baseUrl` in `tsconfig.json`.

---

## 5. `import.meta` Support & Hermes v1 Transforms

Source: <https://expo.dev/changelog/sdk-56>

- **`import.meta` support: "now enabled automatically."** No manual configuration required.
- **Hermes v1 transforms: "fewer bundler transforms are enabled for Hermes, which reduces bundling times overall."** Fewer transforms = faster bundling, contributing to the warm/cold bundling speedups.

---

## 6. `expo/fetch` as Default `globalThis.fetch`

Source: <https://expo.dev/changelog/sdk-56>, <https://docs.expo.dev/guides/environment-variables/>

SDK 56 installs **`expo/fetch` as the default implementation of `globalThis.fetch`**:

- "providing a WinterTC-compliant API and improved performance. Manual imports are no longer required."
- You no longer need to `import { fetch } from 'expo/fetch'` — the global `fetch` already uses it.

### Opting out (env var)

To opt out and restore the React Native default `fetch`:

```sh
EXPO_PUBLIC_USE_RN_FETCH=1
```

- Env var: **`EXPO_PUBLIC_USE_RN_FETCH=1`** — opts out of the new `expo/fetch` default `globalThis.fetch` implementation.

---

## 7. Expo CLI Commands & Flags (bundling / dev server)

Source: <https://docs.expo.dev/more/expo-cli/>

### `npx expo start`

Starts the Metro dev server (default `http://localhost:8081`).

| Flag | Description |
| --- | --- |
| `--port <port>` | Dev server port (default `8081`; `--port 0` for auto-selection) |
| `--localhost` | LAN-only / localhost connection |
| `--tunnel` | Enable ngrok tunneling for remote access |
| `--offline` | Develop without a network connection |
| `--dev-client` | Force launch a development build |
| `--go` | Force launch in Expo Go |

### `npx expo export`

Bundles JS and assets for **production** (strips code guarded by the `__DEV__` boolean).

| Flag | Description |
| --- | --- |
| `--platform <platform>` | Target: `ios`, `android`, `all`, or `web` |
| `--output-dir <dir>` | Export destination (default `dist`) |
| `--dev` | Bundle for development (no minification) |
| `--no-minify` | Skip minification |
| `--no-bytecode` | Skip Hermes bytecode generation |
| `--no-ssg` | Skip static HTML export (web routes only) |
| `--max-workers <number>` | Limit bundler worker tasks |
| `-c, --clear` | Clear the bundler cache |

### `npx expo run:ios` / `npx expo run:android`

Compiles native apps locally (runs prebuild if native dirs are missing).

| Flag | Description |
| --- | --- |
| `--no-build-cache` | Clear native cache before building |
| `--no-bundler` | Skip starting the dev server |
| `--port <port>` | Dev server port (default `8081`) |
| `-d, --device [device]` | Target a specific device |
| `-o, --output <path>` | Copy the built binary to a directory |

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
| `resolver.useWatchman` | Revert to Watchman watcher (SDK 56; not recommended) |
| `transformer.getTransformOptions` | Returns transform flags (see tree shaking) |
| `transformer.getTransformOptions → transform.experimentalImportSupport` | Enables import support / tree shaking |
| `transformer.getTransformOptions → transform.inlineRequires` | Inline requires for smaller bundles |
| `watchFolders` | Additional watched folders (no longer load-bearing in SDK 56) |

App-level (in **app.json**, not `metro.config.js`):

| Key | Purpose |
| --- | --- |
| `expo.experiments.onDemandFilesystem` | Toggle on-demand filesystem (default `true` in SDK 56) |
| `expo.web.bundler: "metro"` | Enable Metro web bundling |

---

## 9. Tree Shaking

Source: <https://docs.expo.dev/guides/tree-shaking/>

- **Enabled by default in SDK 54 and later** (experimental since SDK 52) — so it is on by default in SDK 56.
- Driven by `transformer.getTransformOptions` returning `transform.experimentalImportSupport: true`.
- Optional pairing with `transform.inlineRequires: true` for further size reduction.

For older SDKs only, two env vars (production mode only) gate the behavior:

| Env var | Effect |
| --- | --- |
| `EXPO_UNSTABLE_TREE_SHAKING=1` | Enables tree shaking |
| `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH=1` | Keeps modules until the full graph is created before optimizing |

Test with: `npx expo export` (production bundle).

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

### All env var names (verbatim)

| Env var | Purpose |
| --- | --- |
| `EXPO_PUBLIC_USE_RN_FETCH=1` | Opt out of `expo/fetch` as default `globalThis.fetch` (SDK 56) |
| `EXPO_NO_DOTENV=1` | Prevent loading `.env` files into the process |
| `EXPO_NO_CLIENT_ENV_VARS=1` | Disable inline serialization of env vars in the client bundle |
| `EXPO_UNSTABLE_TREE_SHAKING=1` | Enable tree shaking (older SDKs) |
| `EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH=1` | Defer optimization until full graph built (older SDKs) |
| `EXPO_PUBLIC_API_URL` | Example public var |
| `EXPO_PUBLIC_API_KEY` | Example public var |

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

## Source URLs

- CLI Performance / SDK 56 changes: <https://expo.dev/changelog/sdk-56>
- Expo CLI commands & flags: <https://docs.expo.dev/more/expo-cli/>
- Customizing Metro: <https://docs.expo.dev/guides/customizing-metro/>
- Tree shaking: <https://docs.expo.dev/guides/tree-shaking/>
- Environment variables: <https://docs.expo.dev/guides/environment-variables/>
- `src/` directory: <https://docs.expo.dev/router/reference/src-directory/>
