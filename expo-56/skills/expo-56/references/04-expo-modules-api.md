# Expo Modules API & Native Module Authoring (SDK 56)

Knowledge-base reference covering the Expo Modules API, native module authoring, the
Module DSL, type generation tooling, `create-expo-module`, and config plugins as of
Expo SDK 56.

---

## 1. SDK 56 — Expo Modules Enhancements

Source: <https://expo.dev/changelog/sdk-56>

### 1.1 Inline modules

SDK 56 lets you define Expo modules **directly inside a project**, alongside your
JS/TS, without creating a separate package.

Workflow:

1. After one-time setup, write Kotlin and Swift files as Expo modules — no additional
   configuration needed.
2. During **prebuild**, the iOS Xcode project is updated and Android project options
   are adjusted so the modules are **automatically autolinked**.
3. Inline modules integrate into the project structure, so you can develop them in
   Android Studio, Xcode, or any IDE.

Type generation pairs with inline modules: create a Swift module, run a CLI **watcher**,
and a TypeScript interface is generated automatically beside the Swift file. The
generated interface is split into a **generated** part and a **stable** part — you
control the stable section.

### 1.2 Type generation tools — `expo-type-information`

The `expo-type-information` package exports functions that parse Swift Expo module
type information and generate TypeScript interfaces. CLI commands:

- **`module-interface`** — accepts Swift module file paths or module root paths,
  generating TypeScript files following a standard scheme:
  - `[ModuleName]Types.ts` — type declarations
  - `[ModuleName]Module.ts` — module class
  - `[ModuleName]View.tsx` — default view components with typed props
  - `index.ts` — re-exports
- **`inline-modules-interface`** — generates TypeScript file pairs (generated / stable)
  for each Swift inline module in a project.
- **`short-module-interface`** — targets specific Swift modules instead of all inline
  modules.

All commands support **watch mode** for automatic TypeScript interface regeneration.

### 1.3 Revamped `create-expo-module`

- **New `addPlatformSupport` subcommand** — adds platform support to an existing module
  (e.g. add Android to an iOS-only module). It detects current features and scaffolds
  the native files automatically.
- **Modular templates** — choose which features and platforms get scaffolded.
- **Non-interactive mode** — improved field defaults reduce required inputs; the
  defaults used are logged.
- **No barrel file by default** — local modules skip `index.ts`; use `--barrel` to opt in.
- **Windows support** — fully functional on Windows.

### 1.4 Type-safe config plugins

- Every Expo package that exports config plugins now ships **full TypeScript types**.
- Importing plugins from `expo-<name>/plugin` into `app.config.ts` gives autocomplete,
  JSDoc, and deprecation hints without leaving the editor.
- Config plugins now load with the **same module loader** as configs, so you can
  reference local `.ts` files in the `plugins` list and use `.mjs` / `.cjs` extensions.

### 1.5 Performance — Kotlin compiler plugin (Android)

A new **Kotlin compiler plugin** replaces runtime reflection with **build-time code
generation** for Android Expo Modules. It collects module metadata at compile time
rather than runtime, eliminating reflection-based function-type-to-converter mapping.

> Benchmarks show "roughly 40% faster cold starts and 33% faster first render, with no
> app-side changes required."

### 1.6 Performance — Swift / C++ interop (iOS)

A new **JSI layer for iOS native modules** eliminates the Objective-C++ middle layer
using **Swift/C++ interop**. Previously a JS call into a native module crossed three
language boundaries; now direct Swift-to-JSI communication reduces call overhead with
"significant performance improvements across native module calls."

---

## 2. Overview — What the Expo Modules API is

Source: <https://docs.expo.dev/modules/overview/>

- Build native modules in **Swift and Kotlin**, extending React Native apps with native
  capabilities. Prioritizes modern language features, cross-platform consistency,
  minimal boilerplate, and performance comparable to Turbo Modules.
- **Architecture**: works with the New Architecture and is backward compatible with
  legacy React Native apps.
- **App impact**: negligible size increase (a few hundred KB).
- **Performance**: uses **JSI** for direct native communication, eliminating the JSON
  bridge. Both Expo Modules and Turbo Modules can handle "hundreds of thousands of
  native method calls per second."
- **When to use**: choose Expo Modules API when you want better DX and can depend on the
  `expo` package, and you don't need C++ (reserve Turbo Modules for C++-intensive work).
- **Use cases**: integrating third-party SDKs lacking RN libraries; accessing
  specialized system features.
- **Platform support**: Primary — iOS, Android, Web. Experimental — macOS and tvOS.
- Community examples: `burnt`, `react-native-passkeys`, `react-native-mlkit`; plus the
  Expo SDK GitHub repo and Bluesky.

---

## 3. Getting started

Source: <https://docs.expo.dev/modules/get-started/>

Two workflows:

1. **Local module in an existing app** — add a module to a current Expo project for
   testing/development.
2. **Standalone module** — isolated module with example project, for reuse across
   projects or publishing to npm.

CLI commands:

```sh
# Local module inside the current app
npx create-expo-module@latest --local

# Standalone module
npx create-expo-module@latest my-module

# Supporting commands
npx expo prebuild --clean   # generate native android/ ios/ directories
npx pod-install             # reinstall pods after adding native files
npx expo start              # development server
```

Generated structure (`modules/my-module/`): `android/`, `ios/`, `src/`,
`expo-module.config.json`, plus native sources `MyModule.kt` (Android) and
`MyModule.swift` (iOS).

Workflow note: "repeat the build step anytime you make a change to the native code" for
changes to appear in the running app.

Windows note: "If you're using Windows, you can open the example project by opening the
**android** directory in Android Studio, but you cannot open the iOS project files."

> Note: this page's content describes the standalone/local flow; the inline-modules
> workflow and the `addPlatformSupport` subcommand are documented in the SDK 56
> changelog (Section 1).

---

## 4. Module DSL — definition components

Source: <https://docs.expo.dev/modules/module-api/>

### 4.1 Core components

| Component | Purpose / signature |
|---|---|
| `Name("MyModuleName")` | Sets the module's JavaScript identifier. |
| `Constant("PI") { Double.pi }` | Computes a property once on first access, then caches it. |
| `Constants` *(deprecated)* | Defines multiple constants via dictionary or closure returning a dictionary. |
| `Function("name") { ... }` | Synchronous native function callable from JS. Up to **8 arguments**. |
| `AsyncFunction("name") { ... }` | Promise-returning function dispatched to a different thread by default. Up to 8 args (optionally a `Promise`). |
| `Property("foo") { ... }` | Module property with getter/setter. |
| `View(ViewType.self) { ... }` | Exports a native view. |

Examples:

```swift
Function("mySyncFunction") { (message: String) in return message }
```

```swift
Property("foo") { return "bar" }            // read-only
// mutable: chain .get { } and .set { }
```

`AsyncFunction` execution control: use `.runOnQueue(.main)` to specify the execution
queue. In **Kotlin**, async bodies can be suspendable via the `Coroutine` notation.

### 4.2 Event components

| Component | Purpose |
|---|---|
| `Events("onCameraReady", "onPictureSaved")` | Lists event names the module can emit. |
| `OnStartObserving` | Invoked when the first listener for an event is added (takes event name for per-event scoping). |
| `OnStopObserving` | Invoked when all listeners for an event are removed (requires event name). |

### 4.3 Lifecycle components

| Component | When called |
|---|---|
| `OnCreate` | Immediately after module initialization. |
| `OnDestroy` | When the module is about to be deallocated. |
| `OnAppContextDestroys` | When the owning app context deallocates. |
| `OnAppEntersForeground` *(iOS)* | Before app enters foreground. |
| `OnAppEntersBackground` *(iOS)* | When app enters background. |
| `OnAppBecomesActive` *(iOS)* | When app becomes active after `OnAppEntersForeground`. |
| `OnActivityEntersForeground` *(Android)* | After activity resumes. |
| `OnActivityEntersBackground` *(Android)* | After activity pauses. |
| `OnActivityDestroys` *(Android)* | When activity owning the JS context is destroyed. |
| `OnActivityResult` *(Android)* | When an activity started via `startActivityForResult` returns. Payload has `requestCode`, `resultCode`, optional `data`. |
| `OnNewIntent` *(Android)* | When activity receives a new `Intent`. |
| `OnUserLeavesActivity` *(Android)* | When a user action backgrounds the activity (e.g. Home key). |
| `RegisterActivityContracts` *(Android)* | Registers modern activity result contracts via `registerForActivityResult`, usable in async functions. |

### 4.4 View definition components

| Component | Signature / purpose |
|---|---|
| `Prop(name, defaultValue?, setter)` | Defines a view property setter. |
| `PropGroup` *(Android)* | Batch-registers props with a shared handler (pair-based or string-based positional indexing). |
| `OnViewDidUpdateProps` | Called after the view finishes updating all props. |
| `OnViewDestroys` *(Android)* | Called after view is no longer used by RN. |
| `GroupView` *(Android)* | View-group functionality; viewType must inherit `ViewGroup`. |
| `AddChildView` *(Android)* | `(parent, child, index) -> ()` |
| `GetChildCount` *(Android)* | `(parent) -> Int` |
| `GetChildViewAt` *(Android)* | `(parent, index) -> ChildType` |
| `RemoveChildView` *(Android)* | `(parent, child) -> ()` |
| `RemoveChildViewAt` *(Android)* | `(parent, index) -> ()` |

Prop example:

```swift
Prop("background") { (view: UIView, color: UIColor) in /* ... */ }
```

### 4.5 Argument types

- **Primitives** — Swift: `Bool`, `Int` variants, `Float32`, `Double`, `String`;
  Kotlin: `Boolean`, `Int`, `Long`, `Float`, `Double`, `String`, `Pair`.
- **Convertibles** — native types convertible from JS values. iOS uses the
  `Convertible` protocol; Android uses the `ModuleConverters` builder with `.from<T> { }`
  chains.
- **Records** — structured types with typed fields and defaults. Swift: conform to
  `Record`, use `@Field`. Kotlin: extend `Record`, annotate with `@Field`.
- **Formatter** *(experimental)* — customizes Record serialization with `.map()` and
  `.skip()`.
- **Enums** — limit values to discrete options; must conform/extend `Enumerable`.
- **Eithers** — `Either<A, B>`, `EitherOfThree<A, B, C>`, `EitherOfFour<A, B, C, D>`.
- **ValueOrUndefined** *(experimental)* — distinguishes JS `undefined` from `null`;
  properties `isUndefined` (Bool), `optional` (unwrapped value).
- **JavaScript values** — `JavaScriptValue` (any JS value, requires sync functions),
  `JavaScriptObject` (object only), `JavaScriptFunction<ReturnType>` (callback).

### 4.6 Native classes

- **`Module`** — base class. Property `appContext` → `AppContext`; method
  `sendEvent(eventName: String, payload: [String: Any?])`.
- **`AppContext`** — single Expo app interface. Properties: `constants`, `permissions`,
  `activityProvider`, `reactContext`, `hasActiveReactInstance`, `utilities`.
- **`ExpoView`** — base class for exported views (Android must inherit; iOS optional).
  Property `appContext` → `AppContext`.

---

## 5. Native module tutorial (expo-settings)

Source: <https://docs.expo.dev/modules/native-module-tutorial/>

CLI:

```sh
npx create-expo-module expo-settings
```

DSL used: `Name("ExpoSettings")`, synchronous `Function`s (`getTheme()`,
`setTheme(theme)`), and `Events("onChangeTheme")`.

Workflow: clean boilerplate → implement get/set persistence (SharedPreferences on
Android, UserDefaults on iOS) → emit change events → enforce type safety with a `Theme`
enum (`light` / `dark` / `system`).

TypeScript wrapper pattern:

```ts
declare class ExpoSettingsModule extends NativeModule {
  setTheme: (theme: Theme) => void;
  getTheme: () => Theme;
}
```

---

## 6. Config plugin + native module tutorial (expo-native-configuration)

Source: <https://docs.expo.dev/modules/config-plugin-and-native-module-tutorial/>

Builds a native module plus a config plugin that customizes Android/iOS projects during
`npx expo prebuild`.

CLI:

```sh
npx create-expo-module expo-native-configuration
npm run build
npm run build plugin
npx expo prebuild --clean
npx expo run:android
npx expo run:ios
```

Plugin structure:

- `plugin/tsconfig.json` — TypeScript config
- `plugin/src/index.ts` — plugin implementation
- `app.plugin.js` — entry point in project root

Rule: "Plugins must be synchronous, and their return value must be serializable, except
for any `mods` that are added." Naming convention: prefix with `with` (e.g.
`withMyApiKey`).

Type-safe plugin:

```ts
const withMyApiKey: ConfigPlugin<{ apiKey: string }> = (config, { apiKey }) => {
  // implementation
};
```

iOS — `withInfoPlist`:

```ts
config = withInfoPlist(config, config => {
  config.modResults['MY_CUSTOM_API_KEY'] = apiKey;
  return config;
});
```

Android — `withAndroidManifest`:

```ts
config = withAndroidManifest(config, config => {
  const mainApplication = AndroidConfig.Manifest.getMainApplicationOrThrow(
    config.modResults
  );
  AndroidConfig.Manifest.addMetaDataItemToMainApplication(
    mainApplication,
    'MY_CUSTOM_API_KEY',
    apiKey
  );
  return config;
});
```

Reading the value in native code:

```kotlin
val applicationInfo = appContext?.reactContext?.packageManager?.getApplicationInfo(
  appContext?.reactContext?.packageName.toString(),
  PackageManager.GET_META_DATA
)
applicationInfo?.metaData?.getString("MY_CUSTOM_API_KEY")
```

```swift
Bundle.main.object(forInfoDictionaryKey: "MY_CUSTOM_API_KEY") as? String
```

---

## 7. Config plugins — introduction

Source: <https://docs.expo.dev/config-plugins/introduction/>

Config plugins modify native Android/iOS projects during the **Continuous Native
Generation (CNG)** prebuild process without editing native files directly.

Hierarchy:

- **Plugin** — entry point (e.g. `withMyPlugin`).
- **Plugin functions** — platform-specific wrappers within the plugin.
- **Mod plugin functions** — utilities from `expo/config-plugins` that safely modify
  native files.
- **Mods** — underlying platform modifiers (e.g. `mods.android.manifest`).

Plugins are referenced in the `plugins` property of the app config (`app.json` /
`app.config.ts`).

Characteristics: synchronous functions taking an `ExpoConfig` and returning a modified
config; named `with<PluginFunctionality>`; serializable (except `mods`); evaluated
during the app-config phase. "mods are only evaluated during the syncing phase of
`npx expo prebuild`," so direct app-config modifications should occur outside mods to
work in non-prebuild scenarios.

Import from `expo/config-plugins`.

---

## 8. Config plugins — development and debugging

Source: <https://docs.expo.dev/config-plugins/development-and-debugging/>

### Setup

Install `expo` as a **devDependency (>=56.0.0)** with optional peerDependencies. Import
from `expo/config-plugins` and `expo/config` (not separate packages) to ensure version
compatibility.

### Best practices for mods

- "Avoid regex: static modification is key" — use gradle.properties, JSON configs, or
  strings.xml instead of dangerous regex transformations.
- Avoid long-running tasks, network requests, or interactive prompts in mods.
- Generate/delete files only in dangerous mods to keep introspection compatibility.
- Use built-in helpers like `withXcodeProject` to minimize file read/parse cycles.
- Stick with the XML libraries prebuild uses internally.
- Plugin properties must be static values (boolean, number, string, null, arrays,
  objects) since app configs serialize to JSON.

### Tooling & local testing

- Install the **Expo Tools VS Code extension** for automatic validation.
- Local testing: (1) monorepo `packages/` directory for true npm-like imports, or
  (2) `npm pack` the plugin → `npm install path/to/package.tgz` → add to `plugins`.

### Debugging commands

```bash
EXPO_DEBUG=1 expo prebuild              # plugin stack logs and mod execution order
EXPO_CONFIG_PLUGIN_VERBOSE_ERRORS=1     # all resolution errors (authors only)
expo prebuild --clean                   # remove generated native dirs first
expo config --type prebuild             # print unevaluated plugin results
expo config --type introspect           # preview evaluated modifiers without codegen
EXPO_PROFILE=1                          # profile CLI command performance
```

### Modifying native files

- **AndroidManifest.xml** — built-in merging for static features; `withAndroidManifest`
  and helpers like `addMetaDataItemToMainApplication` when more control is needed.
- **Info.plist** — `withInfoPlist` safely merges, preserving existing values.
- **Podfile** — cannot be modified directly; use `withPodfileProperties` to read/write
  the static **Podfile.properties.json**, which the Podfile reads at build time.
- **Gradle** — avoid regex; write camelCase keys prefixed with `expo.` into
  **gradle.properties**, read via `property()` / `findProperty()`.
- **AppDelegate** — use AppDelegate subscribers (safe) rather than `withAppDelegate`
  regex modifications (strongly discouraged).

### Advanced

- **Introspection** — read evaluated modifier results without generating code. Supports
  static modifiers (manifest, gradleProperties, strings, colors, infoPlist,
  entitlements, podfileProperties); results shown under `_internal.modResults`.
- **Custom base modifiers** — `BaseMods.withGeneratedBaseMods()` with custom providers
  (getFilePath, read, write). Base mods must be added **last** in the plugins array.
- **Plugin history** — `createRunOncePlugin()` prevents duplicate execution; tracked via
  `_internal.pluginHistory` (stores plugin name and version).
- **`expo install` integration** — auto-adds plugins to app.json (root config only) on
  package install; works with static configs only — dynamic `app.config.js` needs
  manual addition.

---

## Sources

- SDK 56 changelog: <https://expo.dev/changelog/sdk-56>
- Modules overview: <https://docs.expo.dev/modules/overview/>
- Get started: <https://docs.expo.dev/modules/get-started/>
- Module API reference: <https://docs.expo.dev/modules/module-api/>
- Native module tutorial: <https://docs.expo.dev/modules/native-module-tutorial/>
- Config plugin + native module tutorial: <https://docs.expo.dev/modules/config-plugin-and-native-module-tutorial/>
- Config plugins introduction: <https://docs.expo.dev/config-plugins/introduction/>
- Config plugins development and debugging: <https://docs.expo.dev/config-plugins/development-and-debugging/>
