# Expo SDK 56 — System UI Components

Domain: StatusBar, NavigationBar, iOS Widgets, expo-dev-launcher, and the vector-icons migration.

> Compiled for an SDK 56 knowledge base. Code, prop names, and package names captured verbatim from the official Expo documentation and the SDK 56 changelog. Verified against the `sdk-56` branch of `expo/expo`. An `## SDK 57 delta` section at the end of this file covers what moved in SDK 57.

### Things a model routinely gets wrong here

1. **Widget components need the `'widget'` directive** as their first statement, and they run in an isolated runtime: only `@expo/ui/swift-ui` components, no hooks, no async, no imports, and **no references to anything declared outside the function** — including a `const` at module scope in the same file. See [The `'widget'` directive](#the-widget-directive).
2. **`NavigationBar.setStyle` / `<NavigationBar style>` is a no-op under the default config.** It only takes effect when the device uses button navigation *and* the `expo-navigation-bar` plugin's `enforceContrast` is set to `false`.
3. **`"style": "auto"` is invalid in a config plugin.** At build time both `expo-status-bar` and `expo-navigation-bar` accept only `'light' | 'dark'`. `'auto'`/`'inverted'` exist only at runtime.
4. **`expo` no longer depends on `@expo/vector-icons`.** Verified: `@expo/vector-icons` does not appear in `packages/expo/package.json` on either the `sdk-56` or `sdk-57` branch.
5. **Live Activities must NOT get a `widgets[]` entry** in app config — that generates an invalid widget target and fails the build.

---

## 1. StatusBar (`expo-status-bar`) & NavigationBar (`expo-navigation-bar`) — Consistent React Components

**Sources:**
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/sdk/status-bar/
- https://docs.expo.dev/versions/v56.0.0/sdk/navigation-bar/

### What's new in SDK 56

In SDK 56, both `expo-status-bar` and `expo-navigation-bar` expose **React components with a consistent prop surface**. `NavigationBar` is now offered as a declarative React component (mirroring the long-standing `StatusBar` component). When multiple instances of either component are mounted, **their props merge in mount order**.

Each library now offers a matching declarative component + imperative API + config plugin.

### Merged component usage (declarative + imperative)

```tsx
import { StatusBar } from 'expo-status-bar';
import { NavigationBar } from 'expo-navigation-bar';

const App = () => {
  useEffect(() => {
    // Imperative API
    StatusBar.setStyle('auto');
    StatusBar.setHidden(false);

    NavigationBar.setStyle('auto');
    NavigationBar.setHidden(false);
  }, []);

  return (
    <>
      {/* Declarative API: equivalent to the imperative calls above */}
      <StatusBar style="auto" hidden={false} />
      <NavigationBar style="auto" hidden={false} />
    </>
  );
};
```

### Config plugin alignment (`app.json`)

```json
{
  "expo": {
    "plugins": [
      ["expo-status-bar", { "style": "light", "hidden": false }],
      ["expo-navigation-bar", { "enforceContrast": false, "style": "light", "hidden": false }]
    ]
  }
}
```

> `enforceContrast` is `false` here on purpose: `style` / `setStyle` only take effect when contrast enforcement is off. See [1b](#1b-navigationbar-api-expo-navigation-bar).

> **Build-time vs runtime `style` are different types.** Both config plugins declare `type StatusBarStyle/NavigationBarStyle = 'light' | 'dark'` (`plugin/src/withStatusBar.ts`, `plugin/src/withNavigationBar.ts`). The runtime component prop / `setStyle` accepts `'auto' | 'inverted' | 'light' | 'dark'`. Writing `"style": "auto"` in **app.json** is invalid.

---

### 1a. StatusBar API (`expo-status-bar`)

#### Component props (`StatusBarProps`)

| Prop | Type | Default | Platforms | Description |
|------|------|---------|-----------|-------------|
| `animated` | `boolean` | — | Android, iOS, tvOS, Web | If the transition between status bar property changes should be animated (for `style` and `hidden`). |
| `hidden` | `boolean` | — | Android, iOS, tvOS, Web | If the status bar is hidden. |
| `hideTransitionAnimation` | `StatusBarAnimation` | `'fade'` | iOS | The transition effect when showing and hiding the status bar using the `hidden` prop. |
| `style` | `StatusBarStyle` | `'auto'` | Android, iOS, tvOS, Web | Sets the color of the status bar text. Default value is `"auto"`, which picks the appropriate value according to the active color scheme. |

#### Imperative methods

- `setHidden(hidden, animation)` — Toggle visibility with optional animation (defaults to `'none'`). Returns `void`.
- `setStyle(style, animated)` — Adjust text color with optional animation. Returns `void`.

#### Types

- `StatusBarAnimation`: `'none'` | `'fade'` | `'slide'`
- `StatusBarStyle`: `'auto'` | `'inverted'` | `'light'` | `'dark'`

#### Examples

Declarative:
```jsx
import { StatusBar } from 'expo-status-bar';

<StatusBar style="light" />
```

Imperative:
```ts
StatusBar.setHidden(true, 'slide');
StatusBar.setStyle('dark', true);
```

> Note: Deprecated standalone functions `setStatusBarHidden()` and `setStatusBarStyle()` remain available but are marked for removal.

---

### 1b. NavigationBar API (`expo-navigation-bar`)

Android-only. `NavigationBar` is the new declarative React component for navigation bar configuration.

> **Read this before using `style`.** Per the JSDoc on `NavigationBar.setStyle` and `NavigationBarProps.style` (`packages/expo-navigation-bar/src/NavigationBar.ts`, `NavigationBar.types.ts`), the style only has an effect when **both** conditions hold:
> - the device navigation bar is using **buttons**, not gesture navigation; and
> - the `expo-navigation-bar` config plugin's `enforceContrast` option is set to **`false`** (it defaults to `true`).
>
> The docs also warn that a bug in the **Android 15 emulator** can make it a no-op regardless — test on a physical device or a different API level.

```jsx
import { NavigationBar } from 'expo-navigation-bar';

export default function App() {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Notice that the navigation bar has light buttons!</Text>
      {/* Requires `{ "enforceContrast": false }` on the expo-navigation-bar plugin. */}
      <NavigationBar style="light" />
    </View>
  );
}
```

#### Component props (`NavigationBarProps`)

| Prop | Type | Default | Platforms | Description |
|------|------|---------|-----------|-------------|
| `hidden` | `boolean` | — | Android | Controls navigation bar visibility. |
| `style` | `NavigationBarStyle` | `'auto'` | Android | Determines button color: `'auto'`, `'light'`, `'dark'`, or `'inverted'`. Subject to the `enforceContrast` caveat above. |

> Multiple `NavigationBar` components merge props in mount order.

#### Config plugin properties (`plugin/src/withNavigationBar.ts`)

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `enforceContrast` | `boolean` | `true` | Whether the OS keeps the navigation bar translucent for contrast between buttons and app content. Set to `false` to make `style` / `setStyle` effective. In SDK 56 the v56 docs note it has **no effect on Android 9 and below** (see the SDK 57 delta). |
| `hidden` | `boolean` | `undefined` | Whether the navigation bar starts hidden. |
| `style` | `'light' \| 'dark'` | `undefined` | Which style the navigation bar starts with. **Not** `'auto'`/`'inverted'`. |
| `barStyle` | `'light' \| 'dark' \| null` | — | **Deprecated**, superseded by `style`; emits an Android config warning. |
| `visibility` | `NavigationBarVisibility` | — | **Deprecated**, superseded by `hidden`; emits an Android config warning. |

#### Imperative methods

```ts
NavigationBar.setHidden(true);   // setHidden(hidden: boolean): void
NavigationBar.setStyle('dark');  // setStyle(style: NavigationBarStyle): void
```

#### Deprecated hooks/methods (still present, marked for removal)

- `useVisibility(): NavigationBarVisibility | null` — current visibility during async init.
- `getVisibilityAsync(): Promise<NavigationBarVisibility>`
- `setVisibilityAsync(visibility): Promise<void>`
- `addVisibilityListener(listener): EventSubscription`
- `setStyle` as a **standalone named export** (`export const setStyle = NavigationBar.setStyle`) — use `NavigationBar.setStyle` instead. Mirrors `expo-status-bar`'s deprecated `setStatusBarStyle` / `setStatusBarHidden`.

> **Name collision.** `@expo/ui/jetpack-compose` also exports a component called `NavigationBar` (a Material bottom nav bar). It is unrelated to `expo-navigation-bar`, which controls the Android *system* navigation bar. Both exist in SDK 56 and 57; see reference 03 for the `@expo/ui` one.

---

## 2. iOS Widgets — Promoted to Stable (`expo-widgets`)

**Sources:**
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/sdk/widgets/

### What's new in SDK 56

After introducing an alpha version of Expo Widgets for iOS in SDK 55, the library is now production-ready. Per the changelog, the highlights:

- "Widgets and Live Activities have full access to the environment and no longer need to be pre-rendered."
- Improved **timeline management**, **error handling**, **config plugin**, and **render timeline**.

### Package & status

- **Package:** `expo-widgets` (pinned `~56.0.24` by `expo install` on the SDK 56 line)
- **Platform:** **iOS only** for rendering. The docs page is `platforms: ['ios']`.
  - An Android side *does* exist but is scaffolding: `packages/expo-widgets/android` contains a stub `WidgetsModule` plus `ExpoWidgetsGlanceWidget`, whose `provideGlance` renders a hardcoded `Text(widgetName)` placeholder. The Android config plugin is opt-in behind `enableAndroid` (added in 56.0.17, `#46463`). Do not promise working Android widgets in SDK 56 **or** 57.
- **Availability:** Requires development builds (not available in Expo Go)
- **Status:** Production-ready with config plugin support

### Installation

```sh
npx expo install expo-widgets
```

### Config plugin setup (`app.json`)

```json
{
  "expo": {
    "plugins": [
      [
        "expo-widgets",
        {
          "bundleIdentifier": "com.example.myapp.widgets",
          "groupIdentifier": "group.com.example.myapp",
          "enablePushNotifications": true,
          "widgets": [
            {
              "name": "MyWidget",
              "displayName": "My Widget",
              "description": "Widget description",
              "ios": {
                "supportedFamilies": ["systemSmall", "systemMedium"],
                "contentMarginsDisabled": false
              }
            }
          ]
        }
      ]
    ]
  }
}
```

> **Use the nested `ios` form.** Since `expo-widgets` **56.0.15** (`#46091`) the top-level `supportedFamilies`, `contentMarginsDisabled` and `configuration` keys are **deprecated aliases** for `ios.supportedFamilies`, `ios.contentMarginsDisabled` and `ios.configuration` (see the `// @deprecated` comments in `plugin/src/types/WidgetConfig.type.ts`). They still work, but new code should nest.

#### Config plugin properties (`ExpoWidgetsConfigPluginProps`)

| Property | Type | Default | Since | Notes |
|----------|------|---------|-------|-------|
| `bundleIdentifier` | `string` | `"<app bundle identifier>.ExpoWidgetsTarget"` | 56.0.0 | Bundle identifier of the generated widget-extension target. |
| `groupIdentifier` | `string` | `"group.<app bundle identifier>"` | 56.0.0 | App group used to share data between app and widgets. **If both `ios.bundleIdentifier` and `groupIdentifier` are unset, prebuild fails** — the bundle identifier is required to derive the group. |
| `enablePushNotifications` | `boolean` | `false` | 56.0.0 | Adds the `aps-environment` entitlement and sets **`ExpoWidgets_EnablePushNotifications`** in **Info.plist**. ⚠️ The v56 **and** v57 docs pages call this key `ExpoLiveActivity_EnablePushNotifications` — that string exists nowhere in the source. `plugin/src/ios/withPushNotifications.ts` writes and `ios/WidgetsModule.swift` reads `ExpoWidgets_EnablePushNotifications` on both branches. |
| `frequentUpdates` | `boolean` | `false` | 56.0.0 | Writes `NSSupportsLiveActivitiesFrequentUpdates` into the **main app's** Info.plist. This is the plugin equivalent of the manual Info.plist edit the "Known limitations" docs describe. **Undocumented on docs.expo.dev** — source: `plugin/src/ios/withAppInfoPlist.ts`. |
| `enableAndroid` | `boolean` | `false` | 56.0.17 | Opts into the Android config plugin (scaffolding only). **Undocumented on docs.expo.dev** — source: `plugin/src/withWidgets.ts`. |
| `widgets` | `WidgetConfig[]` | `[]` | 56.0.0 | **Optional**, not required (`props?.widgets ?? []`). Each entry becomes a separate widget kind. |

`Since` = first version in the SDK 56 line where the option is available. `56.0.17` matters in practice because many projects pinned at `56.0.5`.

#### Per-widget properties (`WidgetConfig`)

| Property | Type | Notes |
|----------|------|-------|
| `name` | `string` | Swift struct name; must be a valid Swift identifier and must match the `name` passed to `createWidget`. |
| `displayName` | `string` | User-facing name in the widget gallery. |
| `description` | `string` | Shown in the widget gallery. |
| `ios.supportedFamilies` | `WidgetFamily[]` | Required inside `ios`. |
| `ios.contentMarginsDisabled` | `boolean` (default `false`) | Disables the system's automatic content margins. |
| `ios.configuration` | `{ title, description?, parameters }` | Makes the widget user-configurable; see [Configurable widgets](#configurable-widgets-ios-17). |
| `android` | `{ minWidth?, minHeight?, targetCellWidth?, targetCellHeight?, resizeMode? }` | `resizeMode` is `'none' \| 'horizontal' \| 'vertical' \| 'both'`. Scaffolding only, and only applied when the plugin's `enableAndroid` is `true`. Set `android: null` to skip a widget on Android. |
| `supportedFamilies`, `contentMarginsDisabled`, `configuration` | — | **Deprecated** flat aliases for the `ios.*` keys. |

`configuration.parameters` is a `Record<string, WidgetParameter>` where each parameter is `{ title, type: 'string' | 'number' | 'boolean' | 'enum', default }`, and an `enum` parameter additionally takes `values: { name, value }[]`.

### Widget families (sizes)

`'systemSmall'` (2×2) | `'systemMedium'` (4×2) | `'systemLarge'` (4×4) | `'systemExtraLarge'` (iPad, 6×4) | `'accessoryCircular'` (Lock Screen) | `'accessoryRectangular'` (Lock Screen) | `'accessoryInline'` (Lock Screen inline text).

### The `'widget'` directive

**Every component passed to `createWidget` or `createLiveActivity` must start with the `'widget'` directive as its first statement.** The directive tells the bundler to compile that function into a separate JS bundle that runs in an isolated runtime inside the widget extension — not in your app's React Native runtime.

Consequences of that isolation:

- Only [`@expo/ui/swift-ui`](https://docs.expo.dev/versions/v56.0.0/sdk/ui/swift-ui/) components and modifiers render. `View` / `Text` from `react-native` are **not** available.
- No React hooks (`useState`, `useEffect`, …), no component state, no context. The function must be pure and return its layout synchronously.
- No asynchronous work, no `import`s inside the function, no access to app runtime or in-memory state.
- **It cannot reference anything declared outside the function** — including plain module-scope `const`s in the *same file*. Only the function body is serialized, so a top-level constant throws `Can't find variable: X` at runtime. Declare constants and helpers **inside** the widget function, or pass them in via props.

All data must arrive through props (`updateSnapshot`, `updateTimeline`, or a Live Activity's `start` / `update`) or the `environment` argument. Images must be written to [`widgetsDirectory`](#sharing-images-with-widgetsdirectory) and referenced by path.

```tsx
import { Text } from '@expo/ui/swift-ui';
import { createWidget, type WidgetEnvironment } from 'expo-widgets';

// Declared at module scope — NOT included in the widget bundle.
const CITY_NAMES: Record<string, string> = { sf: 'San Francisco' };

const CityWidget = (props: object, environment: WidgetEnvironment<{ city: string }>) => {
  'widget';
  // Throws at runtime: Can't find variable: CITY_NAMES
  return <Text>{CITY_NAMES[environment.configuration.city]}</Text>;
};

export default createWidget('CityWidget', CityWidget);
```

### Core API — Widgets

```typescript
import { createWidget, type WidgetEnvironment } from 'expo-widgets';

// Create widget — receives props and the live environment (no pre-rendering required)
function createWidget<
  PropsType extends object = object,
  ConfigurationType extends object | undefined = undefined,
>(
  name: string,
  widget: (props: PropsType, environment: WidgetEnvironment<ConfigurationType>) => React.JSX.Element
): Widget<PropsType, ConfigurationType>

// Widget instance methods:
Widget.updateSnapshot(props: PropsType): void
Widget.updateTimeline(entries: WidgetTimelineEntry<PropsType>[]): void
Widget.getTimeline(): Promise<WidgetTimelineEntry<PropsType>[]>
Widget.reload(): void
```

`WidgetTimelineEntry`: `{ date: Date, props: T }`

Semantics (`packages/expo-widgets/src/Widgets.ts`):

- `updateSnapshot(props)` is implemented as a single-entry timeline — `updateTimeline([{ timestamp: Date.now(), props }])` — so the new content displays immediately. It is the normal way to push an update from the app.
- `updateTimeline(entries)` schedules future, system-driven refreshes; the system swaps entries in at their `date`.
- `getTimeline()` returns **past and future** entries as `{ date: Date, props: T }[]`.
- `reload()` forces the system to refresh content and timeline now.
- The `name` argument must match a `widgets[].name` in app config.

#### Interactive widgets (iOS 17+)

A `Button` from `@expo/ui/swift-ui` placed inside a widget takes a `target` identifier and an `onPress` callback. **The value `onPress` returns becomes the widget's new props** — the runtime persists it and reloads the widget on device with no app process running. This is the primary self-update mechanism for a widget.

```tsx
import { Button, Text, VStack } from '@expo/ui/swift-ui';
import { createWidget } from 'expo-widgets';

const CounterWidget = (props: { count: number }) => {
  'widget';
  return (
    <VStack>
      <Text>Count: {props.count}</Text>
      <Button label="Increment" target="increment" onPress={() => ({ count: props.count + 1 })} />
    </VStack>
  );
};

export default createWidget('CounterWidget', CounterWidget);
```

`addUserInteractionListener` is **not** the update mechanism — it only mirrors taps into app state and fires **only while the app process is alive**. It receives `{ type: 'ExpoWidgetsUserInteraction', source: <widget name>, target: <control target>, timestamp }`.

#### Configurable widgets (iOS 17+)

Adding `ios.configuration` to a widget's app-config entry lets users long-press and edit parameters. The chosen values arrive as `environment.configuration`. Type it with the second generic argument:

```tsx
type WeatherProps = { temperature: number };
type WeatherConfiguration = { city: string };

const WeatherWidget = (
  props: WeatherProps,
  environment: WidgetEnvironment<WeatherConfiguration>
) => {
  'widget';
  return <Text>{environment.configuration.city}</Text>;
};

export default createWidget<WeatherProps, WeatherConfiguration>('WeatherWidget', WeatherWidget);
```

Shipped in `expo-widgets` 56.0.8 (`#45726`).

#### Sharing images with `widgetsDirectory`

```ts
import { widgetsDirectory } from 'expo-widgets';
```

A widget cannot read your app's sandbox, so images must live in the shared app-group container. `widgetsDirectory` is a `file://` URL string for a directory both the app and its widgets can read; write images there from the app and reference them by path inside the widget. The `groupIdentifier` plugin option sets the app group up automatically (falling back to `group.<bundleIdentifier>`).

Added in `expo-widgets` **56.0.17** (`#46339`) — i.e. it does not exist in 56.0.5.

> **Type discrepancy — handle defensively.** The docs say `widgetsDirectory` is `null` when no app group is configured, but the declared type is a non-nullable `string` (`src/ExpoWidgets.ios.ts`), and the non-iOS stub returns `''` (`src/ExpoWidgets.ts`). Check for falsiness, not for `null`.

### Core API — Live Activities

> **Do not add a `widgets[]` entry for a Live Activity.** `createLiveActivity` registers the activity entirely at runtime and the library's built-in Live Activity target renders it. An app-config `widgets[]` entry without `supportedFamilies` produces an invalid widget target and **fails the build**. The `name` you pass to `createLiveActivity` only has to match that call, not any app-config entry.

```typescript
import { createLiveActivity } from 'expo-widgets';

createLiveActivity<T extends object = object>(
  name: string,
  liveActivity: LiveActivityComponent<T>
): LiveActivityFactory<T>

// Factory methods:
LiveActivityFactory.start(props: T, url?: string): LiveActivity<T>
LiveActivityFactory.getInstances(): LiveActivity[]

// Instance methods:
LiveActivity.update(props: T): Promise<void>
LiveActivity.end(dismissalPolicy?: LiveActivityDismissalPolicy, props?: T, contentDate?: Date): Promise<void>
LiveActivity.getPushToken(): Promise<string | null>   // null when push is not enabled / token not yet available
LiveActivity.addPushTokenListener(listener: (event: PushTokenEvent) => void): EventSubscription
```

`LiveActivityComponent<T> = (props: T, environment: LiveActivityEnvironment) => LiveActivityLayout` — it returns a **layout object**, not a single element. It also needs the `'widget'` directive.

#### Live Activity layout structure

```typescript
interface LiveActivityLayout {
  banner: ReactNode
  bannerSmall?: ReactNode
  minimal?: ReactNode
  compactLeading?: ReactNode
  compactTrailing?: ReactNode
  expandedLeading?: ReactNode
  expandedCenter?: ReactNode
  expandedTrailing?: ReactNode
  expandedBottom?: ReactNode
}
```

#### Dismissal policies

`LiveActivityDismissalPolicy = 'default' | 'immediate' | ReturnType<typeof after>`

- `'default'` — System default behavior
- `'immediate'` — Remove immediately
- `after(date)` — Remove at specified time (4-hour window). `after` is a **named export** you must import; `after(date: Date): { after: Date }`.

```ts
import { after } from 'expo-widgets';

await activity.end(after(new Date(Date.now() + 60_000)));
```

### Event listeners

```typescript
addPushToStartTokenListener(listener: (event: PushToStartTokenEvent) => void): EventSubscription
addUserInteractionListener(listener: (event: UserInteractionEvent) => void): EventSubscription
```

- `PushTokenEvent`: `{ activityId: string, pushToken: string }`
- `PushToStartTokenEvent`: `{ activityPushToStartToken: string }`
- `UserInteractionEvent`: `{ type: 'ExpoWidgetsUserInteraction', source: string, target: string, timestamp: number }`

### Environment objects (full environment access — new in stable release)

#### `WidgetEnvironment<T extends object | undefined = undefined>`
- `widgetFamily: WidgetFamily`
- `date: Date`
- `colorScheme?: 'light' | 'dark'`
- `isLuminanceReduced?: boolean` (iOS 16+)
- `showsWidgetLabel?: boolean` (iOS 16+)
- `widgetRenderingMode?: 'fullColor' | 'accented' | 'vibrant'` (iOS 16+) — `'fullColor'` home screen, `'vibrant'` Lock Screen, `'accented'` tinted widgets on iOS 18+
- `widgetContentMargins?: { bottom, leading, top, trailing }` (iOS 17+)
- `levelOfDetail?: 'simplified' | 'default'` (iOS 26+)
- `configuration: T` (iOS 17+) — **not optional** on the type; the user-chosen values from `ios.configuration`

#### `LiveActivityEnvironment`
- `colorScheme: 'light' | 'dark'`
- `isActivityFullscreen?: boolean`
- `isLuminanceReduced?: boolean`
- `activityFamily?: ActivityFamily` — `ActivityFamily = 'small' | 'medium'` (iOS 18+)
- `isActivityUpdateReduced?: boolean` (iOS 18+)
- `levelOfDetail?: 'simplified' | 'default'` (iOS 26+)

---

## 3. expo-dev-launcher / expo-dev-client Updates

**Sources:**
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/sdk/dev-client/
- https://docs.expo.dev/develop/development-builds/development-workflows/

### What's new in SDK 56 (from the changelog)

Notable enhancements to `expo-dev-launcher` / `expo-dev-client`:

- **Error-screen "Copy" button** — copy error details directly from the dev client error screen.
- **NSD service discovery on Android** — Android **Network Service Discovery** (platform `NsdManager`) for finding dev servers. Implemented in `packages/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/nsd/` (`NsdDiscovery.kt`, `NsdDiscoveryApi34.kt`, `NsdDiscoveryLegacy.kt`, …). The SDK 56 changelog entry spells it "NDS" — that is a typo in the changelog, the feature is NSD.
- **Android edge-to-edge** support in the dev launcher UI.
- **Embedded bundle loading** (`#44396`) — surfaces a "Load embedded bundle" option, gated by the `embeddedBundle` plugin option.
- New plugin options: `defaultLaunchURL`, `skipOnboarding`, and `showMenuAtLaunch`.

### Package information

- **Package:** `expo-dev-client` (pinned `~56.0.24` by `expo install` on the SDK 56 line)
- **Platforms:** Android, iOS, tvOS
- **Installation:** `npx expo install expo-dev-client`

> **EAS Build requires `developmentClient: true`.** Installing `expo-dev-client` is not enough — you must also set it on the build profile in **eas.json**, or EAS produces a standalone build with no development tools. This is the single most common reason a "development build" comes out without dev tools.
>
> ```json eas.json
> { "build": { "development": { "developmentClient": true, "distribution": "internal" } } }
> ```

### Config plugin options

The dev-launcher plugin type is `PluginConfigType = PluginConfigOptionsByPlatform & PluginConfigOptions` (`packages/expo-dev-launcher/plugin/src/pluginConfig.ts`). **Every option below can also be nested under an `android` or `ios` key**, and the platform-specific value wins over the top-level one (`props.ios?.X ?? props.X` in `withDevLauncher.ts`).

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `launchMode` | `'most-recent' \| 'launcher'` | `'most-recent'` | `'most-recent'` tries to reconnect to the last opened project and falls back to the launcher; `'launcher'` always opens the launcher screen. |
| `defaultLaunchURL` | `string` | — | Launch directly into this URL instead of the launcher screen. With `launchMode: 'most-recent'` it is used as the fallback when reconnecting fails. |
| `launchModeExperimental` | `'most-recent' \| 'launcher'` | — | **Deprecated** — use `launchMode`. |
| `toolsButton` | `boolean` | `true` | Whether to show the floating tools button. |
| `embeddedBundle` | `boolean` | `false` | When enabled and a bundle file is present, adds a "Load embedded bundle" option to the launcher UI. |
| `skipOnboarding` | `boolean` | `false` | Skips the first-launch dev-menu onboarding overlay. Useful for E2E/CI where the overlay blocks automated input. |
| `showMenuAtLaunch` | `boolean` | `true` | Auto-opens the dev menu on app launch. Set `false` for E2E runs. |

`addGeneratedScheme` (`boolean`, default `true`) is **not** a dev-launcher option — it belongs to `expo-dev-client`'s own plugin (`packages/expo-dev-client/plugin/src/withDevClient.ts`) and registers the generated custom URL scheme on Android and iOS.

```json app.json
{
  "expo": {
    "plugins": [
      [
        "expo-dev-client",
        {
          "launchMode": "most-recent",
          "defaultLaunchURL": "http://localhost:8081",
          "android": { "defaultLaunchURL": "http://10.0.0.2:8081" },
          "toolsButton": true,
          "skipOnboarding": false,
          "showMenuAtLaunch": true
        }
      ]
    ]
  }
}
```

### API methods

`packages/expo-dev-client/src/DevClient.ts` is literally `export * from 'expo-dev-menu';`, so `import * as DevClient from 'expo-dev-client'` and `import * as DevMenu from 'expo-dev-menu'` expose the same API.

- `DevClient.closeMenu(): void` — Closes the development menu.
- `DevClient.hideMenu(): void` — Hides the development menu.
- `DevClient.openMenu(): void` — Opens the development menu.
- `DevClient.registerDevMenuItems(items: ExpoDevMenuItem[]): Promise<void>` — **async**, unlike the three above. Specify custom entries in the development client menu. Each call replaces the previously registered set.

#### `ExpoDevMenuItem` type

- `name` (string): Label for menu entry.
- `callback` (function): Action when selected.
- `shouldCollapse` (optional boolean): Close menu after interaction (default `false`).

### Related development workflows

`npx expo start --tunnel` exposes the dev server publicly for restrictive networks; a specific EAS Update branch can be loaded manually from `https://u.expo.dev/[project-id]?channel-name=[channel-name]`. PR previews and the Extensions/Updates panel belong to the deployment and EAS Update references.

---

## 4. DEPRECATION: `@expo/vector-icons` → `@react-native-vector-icons/*`

**Sources:**
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/guides/icons/

### What changed in SDK 56

- The icons guide (`docs/pages/guides/icons.mdx`) now warns: "`@expo/vector-icons` **will be deprecated and is not recommended**". The replacement is the scoped `@react-native-vector-icons/*` packages, one per icon set (for example `@react-native-vector-icons/material-design-icons`).
- The **`expo` package no longer depends on `@expo/vector-icons`.** Verified — `@expo/vector-icons` does not appear in `packages/expo/package.json` on either the `sdk-56` or `sdk-57` branch. If you keep using it, add it as an explicit dependency.
- `@expo/vector-icons` is still resolvable via `npx expo install @expo/vector-icons`; it is pinned **`^15.0.2`** in `packages/expo/bundledNativeModules.json` on both the `sdk-56` and `sdk-57` release branches.

> Do **not** confuse this migration with `expo-symbols`. `expo-symbols` renders Apple **SF Symbols** and is iOS-only; it is not a replacement for a cross-platform icon font library.

### Migration

Run the official codemod to migrate imports to the new scoped packages:

```sh
npx @react-native-vector-icons/codemod
```

The practical trigger is `expo-doctor`. `VectorIconsCheck` (`packages/expo-doctor/src/checks/VectorIconsCheck.ts`, `sdkVersionRange = '>=56.0.0'`, unchanged in SDK 57) fails when a project depends on `@react-native-vector-icons/common` **and** either `@expo/vector-icons` or `react-native-vector-icons`, warning that mixing them "can lead to icon rendering issues due to conflicts between the packages". Its single `advice` entry is the sentence "If you wish to use the scoped icon packages (recommended), migrate your project by running the codemod: `npx @react-native-vector-icons/codemod`".

### New import path pattern

Old (deprecated) — both forms:
```ts
import { MaterialCommunityIcons } from '@expo/vector-icons';
import Ionicons from '@expo/vector-icons/Ionicons'; // per-set subpath, the form the docs use
```

New (scoped per icon set):
```ts
// e.g. Material Design Icons set
import MaterialDesignIcons from '@react-native-vector-icons/material-design-icons';
```

Each icon family is now its own scoped package under `@react-native-vector-icons/*`. Install only the icon sets you use.

**Some consumers need a companion package.** Expo Router native tabs, for instance, install the icon set *plus* `@react-native-vector-icons/get-image`, which provides the native module that `getImageSourceSync` relies on:

```sh
npx expo install @react-native-vector-icons/material-design-icons @react-native-vector-icons/get-image
```

> If you want to keep using the legacy aggregate package, add it explicitly: `npx expo install @expo/vector-icons`. Note custom icon set helpers (`createIconSet`, `createIconSetFromIcoMoon`, `createIconSetFromFontello`) and the `.Button` component pattern continue to exist in the underlying icon library.

---

## SDK 57 delta

Almost nothing changed in this domain. `expo-status-bar`, the `expo-widgets` JS API and `expo-dev-client` itself have **zero** user-facing changes in 57. The deltas are one `expo-navigation-bar` feature plus one `expo-navigation-bar` fix, one `expo-widgets` config-plugin fix, and a batch of `expo-dev-launcher` UI-only changes. Everything else in this file applies unchanged to SDK 57.

Every item below was checked against `origin/sdk-56` as well and is **not** backported — these are genuine 57-only changes, not SDK 56 patch drift.

### Breaking

None. No API, prop, type or config-plugin option in this file was removed or renamed in SDK 57.

### New in 57

**`expo-navigation-bar` 57.0.0 (`#46491`) — `setStyle` / `setHidden` now apply to React Native `<Modal>` windows on Android.** `NavigationBarModule` implements `com.facebook.react.interfaces.ExtraWindowEventListener` and tracks extra windows, so `NavigationBar.setStyle` / `setHidden` and the `<NavigationBar>` props are applied to modal windows as well as the activity window. In SDK 56 a modal reverted to the default system bar appearance. Source: `packages/expo-navigation-bar/android/src/main/java/expo/modules/navigationbar/NavigationBarModule.kt` — the file exists on **both** branches (it is the module that implements `setStyle`/`setHidden` in SDK 56 too); what is absent on `origin/sdk-56` is the `ExtraWindowEventListener` implementation, the `extraWindows` weak set and the `onExtraWindowCreate`/`onExtraWindowDestroy` overrides. Listed under "### 🎉 New features" in `packages/expo-navigation-bar/CHANGELOG.md` "## 57.0.0 — 2026-06-25".

**`expo-navigation-bar` 57.0.1 (`#47382`) — `enforceContrast` is polyfilled on Android 8 and 9.** `android:enforceNavigationBarContrast` only exists on API 29+, so on Android 8/9 the option previously did nothing (the v56 docs say so explicitly). The plugin now writes **two** items into `AppTheme` — `android:enforceNavigationBarContrast` (with `tools:targetApi="29"`) and a new `expoEnforceNavigationBarContrast` attribute — and the native side reads the Expo attribute below API 29, setting a transparent/scrim `navigationBarColor` to emulate the behaviour. Practical effect: on Android 8/9 `enforceContrast: false` now removes the contrast scrim (`navigationBarColor = Color.TRANSPARENT`) instead of being ignored. `setStyle` itself already worked on Android 8/9 in SDK 56 — 56.0.0 shipped `#44477`, which sets an explicit `LightNavigationBarColor`/`DarkNavigationBarColor` below API 29 so the buttons stay legible; it just always applied that scrim regardless of `enforceContrast`. Source: `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-navigation-bar/plugin/src/withNavigationBar.ts` (`applyEnforceNavigationBarContrast`), `android/src/main/res/values/attrs.xml`, `NavigationBarReactActivityLifecycleListener.kt` (initial state at activity create) and `NavigationBarModule.kt` (`Window.setNavigationBarStyle`, at `setStyle` time).

**`expo-widgets` 57.0.1 (`#47061`) — the generated widget-extension target inherits the app's version numbers.** SDK 56 hardcoded `marketingVersion: '1.0'` / `currentProjectVersion: '1'`; SDK 57 uses `config.ios?.version ?? config.version ?? '1.0'` and `config.ios?.buildNumber ?? '1'`. This removes App Store validation failures caused by a `CFBundleVersion` mismatch between the app and its widget extension. Source: `packages/expo-widgets/plugin/src/ios/xcode/withTargetXcodeProject.ts`.

**`expo-dev-launcher` 57.0.1 — launcher UI only, no plugin or JS API change.** A Settings toggle to switch between auto-launching the most recent app and showing the launcher (`#47131`, which also fixes Android auto-launch), iOS build expiration date in Settings (`#47190`), Local Network permission shown as a status row (`#46758`), reworked Updates-tab empty state (`#46759`), dev-server info as a native sheet (`#46760`), logout confirmation (`#46761`), persistent dev-server discovery with pull-to-refresh (`#46811`), and specific load-failure reasons instead of a generic "Failed to connect" (`#46866`). 57.0.5 fixes a tvOS compile error in `DevServersView` (`#47082`); 57.0.9 fixes an Android `onUserLeaveHint` NPE and a lost EAS sign-in redirect (`#47347`). Source: `packages/expo-dev-launcher/CHANGELOG.md` on `origin/sdk-57`.

### Verified unchanged in 57

- **`expo-status-bar`** — both 57.0.0 and 57.0.1 are "_This version does not introduce any user-facing changes._"; `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-status-bar` touches only CHANGELOG, `android/build.gradle` and `package.json`.
- **`expo-widgets` JS API** — 57.0.0 is "_This version does not introduce any user-facing changes._" and `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-widgets/src` is **empty**. `createWidget`, `createLiveActivity`, `WidgetEnvironment`, `widgetsDirectory`, `after`, the config-plugin `WidgetConfig` type and the `ios`/`android` nesting are byte-identical.
- **`expo-dev-client` / `expo-dev-launcher` config plugins** — every 57.0.x entry for `expo-dev-client` is "no user-facing changes", and `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-dev-launcher/plugin/src packages/expo-dev-client/plugin/src` is **empty**. The full option set (`launchMode`, `defaultLaunchURL`, `launchModeExperimental`, `toolsButton`, `embeddedBundle`, `skipOnboarding`, `showMenuAtLaunch`, plus `android`/`ios` nesting, plus `addGeneratedScheme`) is identical.
- **vector-icons** — `@expo/vector-icons` is `^15.0.2` in `bundledNativeModules.json` on both release branches; `expo-doctor`'s `VectorIconsCheck` is identical on both branches. No SDK 57 change.

> **Docs warning.** `docs/pages/versions/v57.0.0/sdk/widgets.mdx` and `.../dev-client.mdx` are a **stale cut** — they were branched before several June 2026 doc expansions, so the v57 pages are a *subset* of the v56 pages (they still show the deprecated flat `widgets[].supportedFamilies` form, and omit the `toolsButton`/`skipOnboarding`/`showMenuAtLaunch` rows and the eas.json `developmentClient` block). The v56 content in this file is the accurate description of SDK 57 behaviour; verify against `origin/sdk-57` package source, not the v57 docs cut. `v57.0.0/sdk/navigation-bar.mdx` and `status-bar.mdx` differ from v56 only in `sourceCodeUrl`, and the nav-bar page still carries the pre-57.0.1 "Has no effect on Android 9 and below" note for `enforceContrast`.

### Version pins (56 → 57)

Source: `packages/expo/bundledNativeModules.json` on `origin/sdk-56` and `origin/sdk-57` — this file ships inside the `expo` package and is what `expo install` actually resolves against. Do **not** use `docs/public/static/schemas/v5*.0.0/native-modules.json`; those cuts are stale.

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-status-bar` | `~56.0.4` | `~57.0.1` |
| `expo-navigation-bar` | `~56.0.3` | `~57.0.2` |
| `expo-widgets` | `~56.0.24` | `~57.0.6` |
| `expo-dev-client` | `~56.0.24` | `~57.0.9` |
| `@expo/vector-icons` | `^15.0.2` | `^15.0.2` (unchanged) |

First-party `expo-*` packages are **not** flat `~57.0.0` — each has its own patch level, as above.

Both lines are still being patched, so these values move. Re-read `bundledNativeModules.json` on the release branch rather than trusting a memorised number.

`expo-dev-launcher` and `expo-dev-menu` are not listed — they are transitive dependencies of `expo-dev-client`, so the `expo-dev-launcher` versions quoted above (57.0.1 / 57.0.5 / 57.0.9) are what you get via `expo-dev-client`, not something you pin directly.
