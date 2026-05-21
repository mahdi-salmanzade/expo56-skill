# Expo SDK 56 — System UI Components

Domain: StatusBar, NavigationBar, iOS Widgets, expo-dev-launcher, and the vector-icons migration.

> Compiled for an SDK 56 knowledge base. Code, prop names, and package names captured verbatim from the official Expo documentation and the SDK 56 changelog.

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
  "plugins": [
    ["expo-status-bar", { "style": "light", "hidden": false }],
    ["expo-navigation-bar", { "style": "light", "hidden": false }]
  ]
}
```

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

```jsx
import { NavigationBar } from 'expo-navigation-bar';

export default function App() {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Notice that the navigation bar has light buttons!</Text>
      <NavigationBar style="light" />
    </View>
  );
}
```

#### Component props (`NavigationBarProps`)

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `hidden` | `boolean` | — | Controls navigation bar visibility. |
| `style` | `NavigationBarStyle` | `'auto'` | Determines button color: `'auto'`, `'light'`, `'dark'`, or `'inverted'`. |

> Multiple `NavigationBar` components merge props in mount order.

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

- **Package:** `expo-widgets`
- **Platform:** iOS only
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
              "supportedFamilies": ["systemSmall", "systemMedium"]
            }
          ]
        }
      ]
    ]
  }
}
```

#### Config plugin properties

| Property | Type | Default |
|----------|------|---------|
| `bundleIdentifier` | string | `"<app-id>.ExpoWidgetsTarget"` |
| `groupIdentifier` | string | `"group.<app-id>"` |
| `enablePushNotifications` | boolean | `false` |
| `widgets` | array | Required |

Per-widget properties: `name`, `displayName`, `description`, `contentMarginsDisabled`, `supportedFamilies`.

### Widget families (sizes)

`'systemSmall'` (2×2) | `'systemMedium'` (4×2) | `'systemLarge'` (4×4) | `'systemExtraLarge'` (iPad, 6×4) | `'accessoryCircular'` (Lock Screen) | `'accessoryRectangular'` (Lock Screen) | `'accessoryInline'` (Lock Screen inline text).

### Core API — Widgets

```typescript
import { createWidget } from 'expo-widgets';

// Create widget — receives props and the live environment (no pre-rendering required)
createWidget(name: string, widget: (props: T, environment: WidgetEnvironment) => Element)

// Widget instance methods:
Widget.updateSnapshot(props: T): void
Widget.updateTimeline(entries: WidgetTimelineEntry[]): void
Widget.getTimeline(): Promise<WidgetTimelineEntry[]>
Widget.reload(): void
```

`WidgetTimelineEntry`: `{ date: Date, props: T }`

### Core API — Live Activities

```typescript
import { createLiveActivity } from 'expo-widgets';

createLiveActivity(name: string, liveActivity: LiveActivityComponent): LiveActivityFactory

// Factory methods:
LiveActivityFactory.start(props: T, url?: string): LiveActivity<T>
LiveActivityFactory.getInstances(): LiveActivity[]

// Instance methods:
LiveActivity.update(props: T): Promise<void>
LiveActivity.end(dismissalPolicy?: LiveActivityDismissalPolicy, props?: T, contentDate?: Date): Promise<void>
LiveActivity.getPushToken(): Promise<string>
LiveActivity.addPushTokenListener(listener: (event: PushTokenEvent) => void): EventSubscription
```

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

- `'default'` — System default behavior
- `'immediate'` — Remove immediately
- `after(date)` — Remove at specified time (4-hour window)

### Event listeners

```typescript
addPushToStartTokenListener(listener: (event: PushToStartTokenEvent) => void): EventSubscription
addUserInteractionListener(listener: (event: UserInteractionEvent) => void): EventSubscription
```

- `PushTokenEvent`: `{ activityId: string, pushToken: string }`
- `PushToStartTokenEvent`: `{ activityPushToStartToken: string }`
- `UserInteractionEvent`: `{ type: 'ExpoWidgetsUserInteraction', source: string, target: string, timestamp: number }`

### Environment objects (full environment access — new in stable release)

#### `WidgetEnvironment`
- `widgetFamily: WidgetFamily`
- `date: Date`
- `colorScheme?: 'light' | 'dark'`
- `isLuminanceReduced?: boolean`
- `showsWidgetLabel?: boolean`
- `widgetRenderingMode?: 'fullColor' | 'accented' | 'vibrant'`
- `widgetContentMargins?: { bottom, leading, top, trailing }`
- `levelOfDetail?: 'simplified' | 'default'`

#### `LiveActivityEnvironment`
- `colorScheme: 'light' | 'dark'`
- `isActivityFullscreen?: boolean`
- `isLuminanceReduced?: boolean`
- `activityFamily?: ActivityFamily`
- `isActivityUpdateReduced?: boolean`
- `levelOfDetail?: 'simplified' | 'default'`

---

## 3. expo-dev-launcher / expo-dev-client Updates

**Sources:**
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/sdk/dev-client/
- https://docs.expo.dev/develop/development-builds/development-workflows/

### What's new in SDK 56 (from the changelog)

Notable enhancements to `expo-dev-launcher` / `expo-dev-client`:

- **Error-screen "Copy" button** — copy error details directly from the dev client error screen.
- **NDS service discovery on Android** — Network Discovery Service discovery for finding dev servers on Android.
- **Android edge-to-edge** support in the dev launcher UI.
- New plugin options: `defaultLaunchURL`, `skipOnboarding`, and `showMenuAtLaunch`.

### Package information

- **Package:** `expo-dev-client`
- **Platforms:** Android, iOS, tvOS
- **Installation:** `npx expo install expo-dev-client`

### Config plugin options (from SDK reference)

| Option | Default | Purpose |
|--------|---------|---------|
| `launchMode` | `"most-recent"` | Determines whether to launch the most recently opened project or navigate to the launcher screen. |
| `addGeneratedScheme` | `true` | Registers a custom URL scheme for opening projects. |
| `defaultLaunchURL` | — | Launch directly into this URL instead of navigating to launcher screen. |
| `android.launchMode` | `"most-recent"` | Android-specific launch behavior override. |
| `ios.launchMode` | `"most-recent"` | iOS-specific launch behavior override. |
| `android.defaultLaunchURL` | — | Android-specific URL for direct launch. |
| `ios.defaultLaunchURL` | — | iOS-specific URL for direct launch. |

> Per the changelog, SDK 56 also adds the `skipOnboarding` and `showMenuAtLaunch` plugin options.

### API methods

- `DevClient.closeMenu()` — Closes the development menu.
- `DevClient.hideMenu()` — Hides the development menu.
- `DevClient.openMenu()` — Opens the development menu.
- `DevClient.registerDevMenuItems(items)` — Specify custom entries in the development client menu.

#### `ExpoDevMenuItem` type

- `name` (string): Label for menu entry.
- `callback` (function): Action when selected.
- `shouldCollapse` (optional boolean): Close menu after interaction (default `false`).

### Related development workflows

- **PR Previews:** Publish an EAS Update whenever a pull request is updated, surfacing QR codes for testing in a development build via GitHub Actions.
- **Tunnel URLs:** `npx expo start --tunnel` exposes the dev server on a publicly available URL for restrictive networks.
- **Manual URL entry:** Launch a specific branch using `https://u.expo.dev/[your-project-id]?channel-name=[channel-name]`.
- **EAS Update integration:** `expo-updates` enables viewing/loading published updates via the Extensions panel.

---

## 4. DEPRECATION: `@expo/vector-icons` → `@react-native-vector-icons/*`

**Sources:**
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/guides/icons/

### What changed in SDK 56

- `@expo/vector-icons` is **deprecated** and will be replaced by the scoped `@react-native-vector-icons/*` packages, which target individual icon sets (for example `@react-native-vector-icons/material-design-icons`).
- The **`expo` package no longer depends on `@expo/vector-icons`.** If you continue using it, you must add it as an explicit dependency.

### Migration

Run the official codemod to migrate imports to the new scoped packages:

```sh
npx @react-native-vector-icons/codemod
```

### New import path pattern

Old (deprecated):
```ts
import { MaterialCommunityIcons } from '@expo/vector-icons';
```

New (scoped per icon set):
```ts
// e.g. Material Design Icons set
import MaterialDesignIcons from '@react-native-vector-icons/material-design-icons';
```

Each icon family is now its own scoped package under `@react-native-vector-icons/*`. Install only the icon sets you use.

> If you want to keep using the legacy aggregate package, add it explicitly: `npx expo install @expo/vector-icons`. Note custom icon set helpers (`createIconSet`, `createIconSetFromIcoMoon`, `createIconSetFromFontello`) and the `.Button` component pattern continue to exist in the underlying icon library.

---

## Source URLs (summary)

- SDK 56 changelog: https://expo.dev/changelog/sdk-56
- StatusBar: https://docs.expo.dev/versions/v56.0.0/sdk/status-bar/
- NavigationBar: https://docs.expo.dev/versions/v56.0.0/sdk/navigation-bar/
- Widgets (iOS): https://docs.expo.dev/versions/v56.0.0/sdk/widgets/
- DevClient: https://docs.expo.dev/versions/v56.0.0/sdk/dev-client/
- Development workflows: https://docs.expo.dev/develop/development-builds/development-workflows/
- Icons guide: https://docs.expo.dev/guides/icons/
</content>
</invoke>
