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

> **Experimental.** Docs: "Inline modules are experimental and available in Expo SDK 56
> and later. The API is subject to breaking changes."
> (`docs/pages/modules/inline-modules-reference.mdx`)

Workflow:

1. After one-time setup, write Kotlin and Swift files as Expo modules — no additional
   configuration needed.
2. During **prebuild**, the iOS Xcode project is updated and Android project options
   are adjusted so the modules are **automatically autolinked**.
3. Inline modules integrate into the project structure, so you can develop them in
   Android Studio, Xcode, or any IDE.

**The feature is inert without app-config keys.** All three live under
`expo.experiments.inlineModules`:

| Key | Purpose |
|---|---|
| `expo.experiments.inlineModules` | Presence (even `{}`) enables inline modules in Expo CLI and Expo Modules Autolinking. |
| `…inlineModules.watchedDirectories` | Directories scanned for native files, e.g. `["app", "src"]`. Nested directories are included. |
| `…inlineModules.xcodeProjectTargets` *(iOS)* | Xcode target names the files are added to. Defaults to the app's main target. Use the target name only — never the enclosing `abstract_target` name. |

```json app.json
{
  "expo": {
    "experiments": {
      "inlineModules": { "watchedDirectories": ["app", "src"] }
    }
  }
}
```

Constraints on each `watchedDirectories` entry:

- Must sit inside a JS/TS project — some ancestor directory must have a `package.json`.
- Cannot be the project root (`"./"`) or an ancestor of it (`"../"`).
- Cannot be a subdirectory of another entry — use `["app"]`, not `["app", "app/nested"]`.
- Cannot contain `" "`, `"("`, `")"`, `"$"`. So `"app/(tabs)"` is **invalid**, but listing
  `"app"` still picks up native files inside `app/(tabs)`.

Run `npx expo prebuild` after changing these keys for them to take effect.

Naming convention: the file name must match the native module class name (unique across
the whole app). Because `Name(...)` also has to match the file name, you can simply omit it.

Type generation pairs with inline modules: create a Swift module, run a CLI **watcher**,
and a TypeScript interface is generated automatically beside the Swift file. The
generated interface is split into a **generated** part and a **stable** part — you
control the stable section.

SDK 56 patch additions (`expo-modules-autolinking` 56.0.20, 2026-07-15): compile-only
inline module files (compiled into the target but not registered as Expo modules);
iOS target-name matching when deciding which targets get inline modules; and an Android
fix where a Kotlin file with a long comment (e.g. a license header) before its `package`
declaration was silently skipped.

### 1.2 Type generation tools — `expo-type-information`

The `expo-type-information` package (SDK 56 ships **0.0.12**) exports functions that
parse Swift Expo module type information and generate TypeScript interfaces.

> **Prerequisite: macOS + SourceKitten.** The library works only on macOS, and the CLI
> short-circuits with `Sourcekitten not found! Install it like so: brew install sourcekitten`
> before parsing any command if it is missing (`packages/expo-type-information/src/cli.ts`).
> Install with `brew install sourcekitten`.

Common CLI options (every command **except** `inline-modules-interface`):

| Flag | Purpose | Default |
|---|---|---|
| `-i, --input-paths <filePaths...>` | Swift files for a module; globs allowed. | |
| `-m, --module-path <modulePath>` | Expo module root directory. | |
| `-o, --output-path <filePath>` | Where to write; prints to console if omitted. | |
| `-t, --type-inference <level>` | `NO_INFERENCE` / `SIMPLE_INFERENCE` / `PREPROCESS_AND_INFERENCE`. Fall back to a lower level if `PREPROCESS_AND_INFERENCE` errors. | `PREPROCESS_AND_INFERENCE` |
| `-s, --skip-unicode-character-mapping` | Skip mapping non-ASCII characters to ASCII (done by default because SourceKitten miscalculates offsets otherwise). | |
| `-w, --watcher` | Watch input paths and regenerate. | |

Commands:

- **`module-interface`** — accepts Swift module file paths or module root paths,
  generating TypeScript files following a standard scheme:
  - `<ModuleName>.types.ts` — type declarations (note the **dot**, not `Types.ts`)
  - `<ModuleName>Module.ts` — module class
  - one `<ViewName>View.tsx` per view declared in the module — named from the Swift
    **view** class, not the module
  - `index.ts` — re-exports
- **`inline-modules-interface`** — generates TypeScript file pairs for each Swift inline
  module: `<Module>.generated.ts` (regenerated on every run) and `<Module>.tsx` (never
  regenerated once it exists). **Does not take the common options above.** It has a
  *required* `-a, --app-json <appJsonPath>` pointing at the app config that declares the
  watched directories, plus `-w, --watcher` and `-t, --type-inference` — whose default
  here is `SIMPLE_INFERENCE`, not `PREPROCESS_AND_INFERENCE`.
- **`short-module-interface`** — targets specific Swift modules instead of all inline
  modules; overwrites `<ModuleName>.generated.ts` and creates `<ModuleName>.ts` if absent.
- **`generate-mocks-for-file`** — generates mocks for a given Expo module.
- **`other …`** — internal/specific commands: `type-information`,
  `generate-module-types`, `generate-view-types`, `generate-jsx-intrinsics`,
  `preprocess-file`.

All commands support **watch mode** (`-w`) for automatic TypeScript interface regeneration.

### 1.3 Revamped `create-expo-module`

`create-expo-module` is **56.0.3** on SDK 56.

- **New `add-platform-support` subcommand** (kebab-case — there is no
  `addPlatformSupport`) — adds platform support to an existing module (e.g. add Android
  to an iOS-only module). It detects current features and scaffolds the native files
  automatically.

  ```sh
  npx create-expo-module add-platform-support [path]   # run from module root, or pass path
  ```

  | Option | Values |
  |---|---|
  | `-p, --platform <platforms...>` | `apple`, `android`, `web` — the Apple value is **`apple`, not `ios`** |
  | `--features <features...>` | Overrides best-effort feature detection: `Constant`, `Function`, `AsyncFunction`, `Event`, `View`, `ViewEvent`, `SharedObject`, `SwiftUIView`, `SwiftUIModifier`, `ComposeView`, `ComposeModifier` |
  | `-s, --source <source_dir>` | Local template path; defaults to downloading `expo-module-template` from npm |

- **Modular templates** — choose which features and platforms get scaffolded. The same
  `-p/--platform` and `--features` values apply to the main `create-expo-module` command,
  plus `--full-example` (equivalently `--features all`). That set is **not** every feature:
  `FULL_EXAMPLE_FEATURES` is `Constant`, `Function`, `AsyncFunction`, `Event`, `View`,
  `ViewEvent`, `SharedObject` only — the four SwiftUI/Compose features are deliberately
  excluded ("they pull in @expo/ui … so users opt in explicitly",
  `packages/create-expo-module/src/features.ts`) and must be requested by name.
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

Mechanism (verifiable in-repo, in case you need to reason about it): `expo-module-gradle-plugin`
applies the third-party `io.github.lukmccall.pika` Kotlin compiler plugin
(`pika-gradle:0.3.2`) and registers two *introspectable annotations* with it —
`expo.modules.kotlin.types.OptimizedRecord` and
`expo.modules.kotlin.views.OptimizedComposeProps`
(`packages/expo-modules-core/expo-module-gradle-plugin/src/main/kotlin/expo/modules/plugin/ProjectConfiguration.kt`).
Annotate your Record / Compose props classes with those to opt into the compile-time path;
classes that are not introspectable fall back to reflection with a logged warning
(`PropsParsingStrategy.kt`). Byte-identical on SDK 56 and SDK 57.

> Benchmark numbers below are the changelog blog post's claim, not a monorepo-verified
> measurement: "roughly 40% faster cold starts and 33% faster first render, with no
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
                            # (SDK 57: prebuild cleans by default; --clean is redundant, --no-clean opts out)
npx pod-install             # reinstall pods after adding native files
npx expo start              # development server
```

Generated structure (`modules/my-module/`): `android/`, `ios/`, `src/`,
`expo-module.config.json`, plus native sources `MyModule.kt` (Android) and
`MyModule.swift` (iOS).

### 3.1 `expo-module.config.json`

This is the file that actually registers your module classes and AppDelegate
subscribers with autolinking. Source: <https://docs.expo.dev/modules/module-config/>

| Key | Meaning |
|---|---|
| `platforms` | Supported platforms: `android`, `apple` (or the granular `ios` / `macos` / `tvos`), `web`, `devtools`. |
| `apple.modules` | Names of Swift native module classes to put in the generated modules provider. |
| `apple.appDelegateSubscribers` | Names of Swift classes hooking into `ExpoAppDelegate` to receive AppDelegate lifecycle events — this is how you register the subscribers recommended over `withAppDelegate` in Section 8. |
| `android.modules` | **Fully qualified** names (package + class) of Kotlin native module classes for the generated package provider. |

```json expo-module.config.json
{
  "platforms": ["apple", "android"],
  "apple": {
    "modules": ["MyModule"],
    "appDelegateSubscribers": ["MyAppDelegateSubscriber"]
  },
  "android": { "modules": ["expo.modules.mymodule.MyModule"] }
}
```

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

#### Swift macro alternative (iOS, experimental, undocumented on docs.expo.dev)

SDK 56 also ships an attached-macro path that synthesizes the definition instead of you
writing the builder by hand. These are declared in
`packages/expo-modules-core/ios/Core/ExpoModulesMacros.swift` and implemented by the
`@expo/expo-modules-macros-plugin` dependency (`0.2.2` on SDK 56). **Nothing under
`docs/pages/` mentions them** — treat as experimental surface, not a public contract.

| Macro | Effect |
|---|---|
| `@ExpoModule` / `@ExpoModule("Name", classes: [Foo.self])` | On a `Module` subclass: scans for `@JS` members and synthesizes `_synthesizedDefinition()`, which core merges into the definition automatically. Also synthesizes `_decorateModule` binding `@JS` functions straight onto the module's JS object (added 56.0.16). `classes:` wires in `@SharedObject` types. |
| `@JS` / `@JS("jsName")` | Marks a function, an `async throws` function, or a computed `var` for JS exposure. Emits a compile-time assertion that every type crossing the boundary is JS-convertible. |
| `@SharedObject` | On a `SharedObject` subclass: synthesizes `_synthesizedClassDefinition()` from `@JS` members, including a single `@JS init(...)` as the JS constructor. |
| `@OptimizedFunction` | On a Swift func (the doc-comment example uses `private`, but there is no access-level constraint — the declaration is `@attached(peer, names: arbitrary) public macro OptimizedFunction()`): generates an `OptimizedFunctionDescriptor` peer for the `Function("name", descriptor)` overload, using the optimized JSI path. |

```swift
@ExpoModule
public final class MyModule: Module {
  public func definition() -> ModuleDefinition {
    Name("MyModule")   // still required on SDK 56 — see the SDK 57 delta
  }

  @JS
  func greet(name: String) -> String { "Hi, \(name)" }
}
```

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
| `Name("MyViewName")` | Sets the view's JS name. Inferrable from the view class name, but the docs recommend setting it explicitly. Distinct from the module-level `Name`. |
| `Prop(name, defaultValue?, setter)` | Defines a view property setter. |
| `PropGroup` *(Android)* | Batch-registers props with a shared handler (pair-based or string-based positional indexing). |
| `AsyncFunction("name") { (view, …) in }` | View-scoped async function attached to the view ref, for direct native-view mutation. Always dispatched on the main queue; receives the view instance as the first argument. Distinct from the module-level `AsyncFunction`. |
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

> Docs caveat on `PropGroup`: "`PropGroup` is used internally by the CSS prop decorators.
> Most modules should use individual `Prop` definitions unless they have many props with
> a shared setter pattern."

### 4.5 Argument types

- **Primitives** — Swift: `Bool`, `Int` variants, `Float32`, `Double`, `String`;
  Kotlin: `Boolean`, `Int`, `Long`, `Float`, `Double`, `String`, `Pair`.
- **Convertibles** — native types convertible from JS values. iOS uses the
  `Convertible` protocol; Android uses the `ModuleConverters` builder with `.from<T> { }`
  chains.
- **Records** — structured types with typed fields and defaults. Swift: conform to
  `Record`, use `@Field`. Kotlin: extend `Record`, annotate with `@Field`.
  - *iOS macro alternative (`expo-modules-core` **>= 56.0.16**, not in 56.0.5):* apply
    `@Record` to a struct/class and every non-`static`, non-`private`/`fileprivate`,
    non-`lazy`, non-computed stored property becomes a record field — no `@Field` wrappers.
    Requiredness is inferred: a default value makes it optional, an optional type makes it
    nullable + optional, a non-optional without a default is required (the factories throw
    `RecordPropertyRequiredException` when the source omits it). It synthesizes the
    memberwise `init`, `from(object:appContext:)`, `from(dictionary:appContext:)`,
    `toDictionary(appContext:)`, `toObject(appContext:)` and auto-conforms to `Record`,
    so it stays usable anywhere a `Record` argument is expected. Undocumented on
    docs.expo.dev; see `packages/expo-modules-core/ios/Core/ExpoModulesMacros.swift`.

    ```swift
    @Record
    struct Options {
      var name: String     // required
      var count: Int = 0   // optional (has default)
      var note: String?    // nullable + optional
    }
    ```
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

### 4.7 `Class` — shared objects & shared refs

Source: <https://docs.expo.dev/modules/shared-objects/> (this surface is **not** on the
module-api page, so it is easy to miss).

A shared object is a long-lived native instance exposed to JS. It is deallocated
automatically once neither JS nor native holds a reference — JS does not own its lifecycle.
Use it for heavy state (a decoded bitmap, a DB handle, an in-flight request) instead of
re-creating a native instance on every call. `expo-image`, `expo-image-manipulator`,
`expo-sqlite` and `expo/fetch` are all built on it.

Native base classes:

- **Swift** — `SharedObject` from `ExpoModulesCore`; `final class Foo: SharedObject`.
- **Kotlin** — `expo.modules.kotlin.sharedobjects.SharedObject`. The base declaration is
  `open class SharedObject(runtime: Runtime? = null)` (`expo.modules.kotlin.runtime.Runtime`),
  so the argument is **optional**: `class Foo(runtime: Runtime) : SharedObject(runtime)`, or
  just `class Foo : SharedObject()`. A secondary `constructor(appContext: AppContext)` exists.
  The docs page still writes `RuntimeContext`; on SDK 56 that is only a
  `@Deprecated("Use expo.modules.kotlin.runtime.Runtime instead")` typealias and using it
  produces a deprecation warning.
- **`SharedRef<T>`** — specialised subclass for wrapping a single native value:
  `SharedRef<UIImage>` (iOS) / `SharedRef<Bitmap>` (Android). Image-aware modules such as
  `expo-image` already accept these directly. On the JS side:
  `import type { SharedRef } from 'expo'` (e.g. `SharedRef<'image'>`).

Expose it with `Class(...)` in the module definition — `Class(Foo.self) { }` (Swift) /
`Class(Foo::class) { }` (Kotlin), or `Class("JsName", Foo.self) { }` to set the JS name.

| Component (inside `Class`) | Purpose |
|---|---|
| `Constructor { (args) in }` | Enables `new ClassName(args)` from JS. **Without it, instances can only be produced by native functions that return the shared object.** |
| `Function` / `AsyncFunction` | Instance methods; receive the instance as the first argument. |
| `StaticFunction("name") { }` | `ClassName.name()` on the class itself — does **not** receive the instance. |
| `StaticAsyncFunction("name") { }` | `await ClassName.name()`; Kotlin can use the `Coroutine` modifier. |
| `Property("name") { (inst) in }` | Instance property; like `Function`, receives the instance as its first parameter. Chain `.get { }` / `.set { }` for read-write. |

```swift
Class(VideoPlayer.self) {
  Constructor { (source: String) in VideoPlayer(source) }

  Property("isPlaying") { (player: VideoPlayer) -> Bool in player.isPlaying }

  Property("volume")
    .get { (player: VideoPlayer) -> Float in player.volume }
    .set { (player: VideoPlayer, volume: Float) in player.volume = volume }

  AsyncFunction("renderAsync") { (player: VideoPlayer) -> ImageRef in player.render() }
}
```

Note `SharedObject.emit` gained single-payload overloads in `expo-modules-core` 56.0.9,
and the older `emit(event:arguments:)` (iOS) / `vararg emit` (Android) were deprecated in
56.0.6 — existing single-argument call sites keep working.

### 4.8 SDK 56 patch drift — worklets integration

Two `expo-modules-core` worklets fixes are often mistaken for SDK 57 features. They shipped
on **both** release lines; you get them with a patch bump on 56, no SDK upgrade needed.

| Change | Minimum SDK 56 patch |
|---|---|
| The worklet UI runtime is resolved from the `react-native-worklets` holder via `getUIRuntimeHolder()` instead of reanimated's `_WORKLET_RUNTIME` global; `installOnUIRuntime` now takes the holder (#46922, #46935). | `expo-modules-core` **56.0.18** |
| Actionable error when a worklet is used but `react-native-worklets`'s native adapter isn't linked, instead of the misleading "not an instance of Worklet" failure (#46571). | `expo-modules-core` **56.0.15** |

Verified present in both `origin/sdk-56:packages/expo-modules-core/CHANGELOG.md` and
`origin/sdk-57:` the same file.

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
npx expo prebuild --clean   # SDK 57: cleaning is the default; --clean still parses but is
                            # a no-op, and --no-clean is the opt-out
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

Depend on `expo` twice, with two different ranges — a **caret** devDependency and a
**`>=`** optional peerDependency:

```json package.json
{
  "devDependencies": { "expo": "^56.0.0" },
  "peerDependencies": { "expo": ">=56.0.0" },
  "peerDependenciesMeta": { "expo": { "optional": true } }
}
```

(The docs render these from an SDK placeholder — use `^57.0.0` / `>=57.0.0` for SDK 57.)
A loose range like this is fine for plugins that only touch stable surfaces such as
**AndroidManifest.xml** or **Info.plist**. `expo-module-scripts` as a devDependency is
optional.

Import from `expo/config-plugins` and `expo/config` — **never** from the standalone
`@expo/config-plugins` / `@expo/config` packages — to ensure version compatibility.

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
expo prebuild --clean                   # remove generated native dirs first (SDK 56 phrasing;
                                        # on SDK 57 cleaning is the default — --clean is still
                                        # accepted but redundant, opt out with --no-clean)
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
  regex modifications (strongly discouraged). Register them via
  `apple.appDelegateSubscribers` in `expo-module.config.json` (Section 3.1).

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
- Shared objects: <https://docs.expo.dev/modules/shared-objects/>
- `expo-module.config.json`: <https://docs.expo.dev/modules/module-config/>
- Inline modules reference: <https://docs.expo.dev/modules/inline-modules-reference/>
- Type generation reference: <https://docs.expo.dev/modules/type-generation-reference/>

**Not covered here** — consult these directly if the task touches them:
`modules/native-view-tutorial`, `modules/autolinking`, `modules/mocking`,
`modules/appdelegate-subscribers`, `modules/android-lifecycle-listeners`,
`modules/third-party-library`, `modules/existing-library`,
`modules/additional-platform-support`, `modules/design`,
`modules/inline-modules-tutorial`, `modules/type-generation-tutorial`,
`modules/use-standalone-expo-module-in-your-project`, `config-plugins/mods`,
`config-plugins/plugins`, `config-plugins/dangerous-mods`, `config-plugins/patch-project`,
`config-plugins/development-for-libraries`.

---

## SDK 57 delta

Almost nothing in this domain changed. Diffing `docs/pages/modules` and
`docs/pages/config-plugins` between `origin/sdk-56` and `origin/sdk-57` returns **empty**,
so the upstream docs prose is unchanged and the conceptual content of Sections 2-8 carries
over to SDK 57. The SDK-56-specific *annotations* in Sections 3, 4.1, 6 and 8 do not — see
the deltas below. (Both doc trees are unversioned — there is no `modules/` or
`config-plugins/` directory under `docs/pages/versions/v56.0.0/` or `v57.0.0/`.) The real 57
deltas are one iOS breaking change in `expo-modules-jsi`, a handful of iOS-only additions in
`expo-modules-core`, and the `expo prebuild` default flip.

### Breaking

- **iOS, `expo-modules-jsi` 57.0.0 (#47154)** — `JavaScriptError` is now a **copyable
  class conforming to `Error`** (was a non-copyable struct), and **`JavaScriptValue` no
  longer conforms to `Error`**. Swift native code that caught a `JavaScriptValue` as an
  `Error`, or that moved a `JavaScriptError` as a non-copyable value, must be updated.
  Same PR: awaiting a `JavaScriptPromise` rejected *after* the await begins now **throws**
  instead of resuming as if fulfilled. Source: `packages/expo-modules-jsi/CHANGELOG.md`
  → `## 57.0.0 — 2026-06-25` → 🛠 Breaking changes.
- **`npx expo prebuild` now cleans by default.** `clean: !args['--no-clean']`
  (`packages/@expo/cli/src/prebuild/index.ts` on `origin/sdk-57`) versus
  `clean: args['--clean']` on `origin/sdk-56`. `--clean` is still accepted but redundant;
  the opt-out is `--no-clean` ("Apply changes to the existing native folders instead of
  recreating them"). Every `prebuild --clean` in Sections 3, 6 and 8 is SDK-56 phrasing.
- Config-plugin authoring (Section 8 setup): use `"expo": "^57.0.0"` in `devDependencies`
  and `"expo": ">=57.0.0"` in `peerDependencies`.

### New in 57

All iOS-only, all in `expo-modules-core` 57.0.0 unless noted. Source:
`git show origin/sdk-57:packages/expo-modules-core/CHANGELOG.md`, plus
`packages/expo-modules-core/ios/Core/ExpoModulesMacros.swift`.

- **`@Event` macro** (#46938) — turns a function-typed `var` on a module or shared object
  into a typed JS event; call the property to emit. JS name defaults to the property name
  with a leading `on` stripped (`onProgress` emits `progress`); override with
  `@Event("customName")`. Signature: `macro Event(_ name: String? = nil, sync: Bool = false)`.
  A `() -> Void` property is a no-payload event.

  ```swift
  @Event
  var onProgress: (ProgressEvent) -> Void
  // emit:
  onProgress(ProgressEvent(percent: 50))
  ```

- **`@ExpoModule` no longer needs `Name(…)`** (#46938) — it synthesizes the JS name into a
  `_jsName` static from the class name, or from `@ExpoModule("CustomName")`. On SDK 56 the
  macro's own doc comment still showed a `definition() { Name("MyModule") }` block; on 57
  that block can be dropped entirely.
- **`SharedObject.native(from:)`** (#47054) — recovers the native shared object paired with
  a JS object, with a generic overload returning the concrete subclass directly.
- **`Module.emit`** (#46555) — sends an event straight to the module's own JS object,
  mirroring `SharedObject.emit`; both now share an `EventEmitter` protocol.
- **`@SharedObject` prototype binding — requires 57.0.4+, not 57.0.0** (#47107,
  57.0.4 / 2026-07-15): `@JS` methods, properties and the `@JS init` are bound directly
  onto the class prototype via synthesized `_decorateSharedObject(prototype:in:)` and
  `_constructSharedObject(this:arguments:in:)`, so they no longer need `Function(…)` /
  `Property(…)` / `Constructor { … }` entries — the synthesized `Class(…)` block carries
  no DSL entries for `@JS` members.

Supporting APIs for module authors (57.0.0 💡 Others), extending Section 4.6:

- `AppContext.from(runtime:)` is now public (#47111).
- `Exceptions.ArgumentsRangeMismatch` is now public — thrown by synthesized `@JS` bindings
  when JS passes an argument count outside the accepted range (#46901).
- `JavaScriptDecodable` / `JavaScriptEncodable` (composed as `JavaScriptCodable`) — a
  statically dispatched, non-erasing JS↔native conversion path, with conformances for
  primitives, containers, records, enumerables and `Data` (#46893). It maps
  `Int64`/`UInt64` to JS `BigInt`, and `Int`/`UInt` **throw** when encoding outside JS's
  safe-integer range (#46939). `Date` conformance added in `expo-modules-jsi` 57.0.2.

Worklets peer range (a real 57-only change): `expo-modules-core`'s
`react-native-worklets` peer went from `^0.7.4 || ^0.8.0` (56) to
`^0.7.4 || ^0.8.0 || ^0.9.0 || ^0.10.0` (57) — #46950 added the `^0.9.0` entry.
SDK 57 pins worklets **0.10.0** (SDK 56 pins 0.8.3), so the widening is load-bearing.
Source: `packages/expo-modules-core/package.json` on each release branch.

The *behavioural* worklets changes are **not** a reason to upgrade — see §4.8.

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 |
|---|---|---|
| `expo-modules-core` | `~56.0.22` | `~57.0.7` |
| `expo-modules-jsi` | `56.0.12` | `57.0.4` |
| `expo-modules-autolinking` | `56.0.21` | `57.0.9` |
| `create-expo-module` | `56.0.3` | `57.0.0` |
| `expo-type-information` | `0.0.12` | `0.1.5` |
| `@expo/expo-modules-macros-plugin` | `0.2.2` | `0.6.1` |

`expo-modules-core` is the only row in
`git show origin/sdk-5{6,7}:packages/expo/bundledNativeModules.json` — that file ships inside
the `expo` package and is what `expo install` resolves against, so it is the pin that matters.
The rest come from `git show origin/sdk-5{6,7}:packages/<pkg>/package.json`
(`@expo/expo-modules-macros-plugin` from `expo-modules-core`'s own dependency entry). Both
lines are still being patched, so treat these as floors, not ceilings.

> Do **not** take these from `docs/public/static/schemas/v{56,57}.0.0/native-modules.json` —
> those schemas are frozen at SDK-release time and are already stale.

`create-expo-module` and `expo-type-information` are **functionally identical** across the
two — `git diff --stat origin/sdk-56 origin/sdk-57 -- packages/create-expo-module` shows
only the version bump, and `expo-type-information`'s `src/cli.ts` is byte-identical. The
commands, options and generated file names in Sections 1.2 and 1.3 apply unchanged to 57.

### Deliberately NOT in 57

These land on the SDK 58 / UNVERSIONED track (`## Unpublished` in `expo-modules-core`'s
CHANGELOG on `main`) — do not apply them to a 57 project:
`AppContext.hostingRuntimeContext` / `AppContext.errorManager` removal (#46964); the
Android `ArrayBuffer` interface→class replacement, `NativeArrayBuffer` deprecation and
`ArrayBuffer.withJSBytes` (#47106, #47261); `ConverterContext` (#47850); and the iOS
`didCreate` / `willDestroy` / `didStartListening` / `didStopListening` hooks that replace
`OnCreate` / `OnDestroy` / `OnStartObserving` / `OnStopObserving` (#47542). Section 4.3 is
still correct for 57. `origin/sdk-57`'s `## 57.0.0` section has **no** 🛠 Breaking changes
entries for `expo-modules-core` at all. (Each PR number above returns zero hits in
`git show origin/sdk-57:packages/expo-modules-core/CHANGELOG.md`.)

Also **not** in 57, despite circulating as "SDK 57" advice — relevant because it would change
Sections 3.1 and 8:

- **iOS UIKit scene lifecycle.** There is no `ExpoAppSceneDelegate`, no
  `UIApplicationSceneManifest` in the templates, and `AppDelegate` keeps its
  `RCTLinkingManager` overrides. `git grep -l "ExpoAppSceneDelegate\|UIApplicationSceneManifest"
  origin/sdk-57 -- packages/ templates/` returns nothing (the one `SceneDelegate.swift` on the
  branch is an `expo-updates` e2e fixture). **`apple.appDelegateSubscribers` remains the
  supported hook on 57** — do not migrate a config plugin to scene delegates.
- **A Node `engines` bump.** `git show origin/sdk-57:packages/expo/package.json` has no
  `engines` field; Node >= 20.19.4 still works for plugin/module tooling.
