# Expo Router (SDK 56) — Knowledge Base Reference

> Domain: **Expo Router** for Expo SDK 56. SDK 56 is a **MAJOR** release for Router: it no longer depends on React Navigation, adds the alpha `ExperimentalStack`, experimental Android toolbar support, streaming SSR for web, data-loader helpers, and customizable Suspense fallbacks.
>
> **Version pins (SDK 56):** `expo-router ~56.2.16` · `expo-server ~56.0.5` · `react-native-screens ~4.26.0` · `react-native-safe-area-context ~5.7.0` · `react-native-web ~0.21.0`. Source: `packages/expo/bundledNativeModules.json` on the `sdk-56` release branch — this is what `npx expo install` actually resolves against. (Do **not** use `docs/public/static/schemas/v56.0.0/native-modules.json`: it is frozen at the GA cut and still says `expo-router ~56.2.9` / `react-native-screens 4.25.2`.) The SDK 56 line is actively patched and `56.2.16` carries a large number of backports — read the delta section at the end of this file before assuming a feature requires SDK 57.

---

## Import cheat sheet (get these wrong and nothing compiles)

Everything below is verified against `packages/expo-router/src/exports.ts`, `src/index.tsx` and the package's root subpath shim files at the SDK 56 cut. Note: `expo-router` ships **no** `package.json#exports` map in either 56.x or 57.x — every subpath resolves through a root shim file (`head.js`, `drawer.js`, `server.js`, `stack.js`, `tabs.js`, `js-stack.js`, `js-tabs.js`, `js-top-tabs.js`, `react-navigation.js`, `html.js`, `unstable-native-tabs.js`, `testing-library.js`, `ui.js`, `unstable-split-view.js`) or a directory (`rsc/`). Verified by unpacking `expo-router@56.2.9`, `56.2.16`, `57.0.0` and `57.0.8`.

| Symbol | Correct import | Common wrong guess |
|---|---|---|
| `Stack`, `Link`, `Slot`, `Navigator`, `router`, `useRouter`, hooks | `from 'expo-router'` | — |
| `Tabs` (JS tabs) | `from 'expo-router/js-tabs'` | ⚠️ `from 'expo-router'` — still exported, but marked `@deprecated Use 'import { Tabs } from 'expo-router/js-tabs'' instead.` (`build/exports.d.ts:41`) |
| `ExperimentalStack` | `from 'expo-router'` | `'expo-router/experimental-stack'` |
| `NativeTabs` (+ `NativeTabTrigger`) | `from 'expo-router/unstable-native-tabs'` | ❌ `from 'expo-router'` — **not** a root export |
| `Drawer` | `from 'expo-router/drawer'` | ❌ `from 'expo-router'` |
| `Stack as JsStack` (JS stack) | `from 'expo-router/js-stack'` | `'@react-navigation/stack'` |
| `createStaticLoader`, `createServerLoader`, `LoaderFunction`, `GenerateMetadataFunction`, `ImmutableRequest`, `Metadata` | `from 'expo-router/server'` | ❌ `from 'expo-router'` |
| `useLoaderData`, `SuspenseFallbackProps` | `from 'expo-router'` | `'expo-router/server'` |
| `StatusError`, `origin`, `environment`, `setResponseHeaders` | `from 'expo-server'` | `'expo-router/server'` |
| `Head` | `default` export of `'expo-router/head'` | named import |
| `ScrollViewStyleReset`, `useServerDocumentContext` | `from 'expo-router/html'` | `'expo-router'` |
| `ThemeProvider`, `DarkTheme`, `DefaultTheme`, `useTheme`, `useRoute`, `useNavigation` | `from 'expo-router'` | ❌ `'@react-navigation/native'` — hard bundler error in SDK 56. ⚠️ `'expo-router/react-navigation'` works but every one of these six is `@deprecated Import X from 'expo-router' instead. Will be removed in a future SDK.` (`build/react-navigation/{native,core}/index.d.ts`) |

Inside a `NativeTabs` layout, use the **compound** form — `NativeTabs.Trigger`, `NativeTabs.Trigger.Label`, `.Icon`, `.Badge`, `.VectorIcon`, `NativeTabs.BottomAccessory`. `expo-router/unstable-native-tabs` exports only `NativeTabs` and `NativeTabTrigger` as values (`packages/expo-router/src/native-tabs/index.ts`), so the standalone `Icon` / `Label` / `Badge` imports the SDK 54 docs show are **no longer re-exported from that subpath**. They are not gone: they moved to the **`expo-router` root** (`export { Badge, Icon, Label, VectorIcon } from './primitives'`, `build/exports.d.ts:33`), and `NativeTabs.Trigger.Label` / `.Icon` / `.Badge` / `.VectorIcon` are aliases of exactly those components. `Stack.Toolbar` composition uses the same primitives.

```tsx
import { Icon, Label, Badge, VectorIcon } from 'expo-router';   // ✅ root
import { Icon } from 'expo-router/unstable-native-tabs';        // ❌ not exported
```

---

## 1. SDK 56 Headline Changes

Source: https://expo.dev/changelog/sdk-56 · https://docs.expo.dev/router/migrate/sdk-55-to-56

### 1.1 Expo Router no longer depends on React Navigation (BREAKING)

- Expo Router has **forked the components it needs** and now operates independently of React Navigation.
- React Navigation can still be installed/used in an Expo project, but **"most code imported directly from `@react-navigation/*` packages will no longer work out of the box alongside `expo-router`."**
- In SDK 56, *"Expo Router no longer supports importing from external `@react-navigation/*` packages in application code."* You must update imports to the matching `expo-router` entry points. The **runtime API stays the same** — only the import paths change.

#### Migration codemod

```sh
npx expo-codemod sdk-56-expo-router-react-navigation-replace [your-source-directory]
```

Replace the bracketed argument with your real source folder, e.g.:

```sh
npx expo-codemod sdk-56-expo-router-react-navigation-replace src
```

The codemod handles most of the import rewriting automatically; the full migration guide covers remaining manual steps.

#### Import mapping (`@react-navigation/*` → `expo-router/*`)

| SDK 55 import | SDK 56 import |
|---|---|
| `@react-navigation/native` | `expo-router/react-navigation` |
| `@react-navigation/core` | `expo-router/react-navigation` |
| `@react-navigation/elements` | `expo-router/react-navigation` |
| `@react-navigation/routers` | `expo-router/react-navigation` |
| `@react-navigation/stack` | `expo-router/js-stack` |
| `@react-navigation/bottom-tabs` | `expo-router/js-tabs` |
| `@react-navigation/material-top-tabs` | `expo-router/js-top-tabs` |
| `@react-navigation/native-stack` | **No direct equivalent** — use the `Stack` layout |
| `@react-navigation/drawer` | **No direct equivalent** — use the `Drawer` layout (`expo-router/drawer`) |

Source: `docs/pages/router/migrate/sdk-55-to-56.mdx` (migration table).

> The codemod rewrites `@react-navigation/native|core` to `expo-router/react-navigation`, which compiles — but the theme and navigation symbols re-exported there (`ThemeProvider`, `DarkTheme`, `DefaultTheme`, `useTheme`, `useRoute`, `useNavigation`, `Link`, `useLinkTo`) each carry `@deprecated … Will be removed in a future SDK.` Prefer importing those from the `expo-router` root once the codemod has run.

**Before (SDK 55):**
```tsx
import { ThemeProvider, DarkTheme } from '@react-navigation/native';
import { createMaterialTopTabNavigator } from '@react-navigation/material-top-tabs';
```

**After (SDK 56):**
```tsx
import { ThemeProvider, DarkTheme } from 'expo-router/react-navigation';
import { createMaterialTopTabNavigator } from 'expo-router/js-top-tabs';
```

#### Library compatibility shim

Expo CLI automatically rewrites `@react-navigation/core` imports coming from `node_modules` as a temporary shim so third-party libraries keep working. To disable this behavior:

```sh
EXPO_ROUTER_DISABLE_RN_NAVIGATION_CHECK=1
```

### 1.2 `ExperimentalStack` — the new native stack (alpha, SDK 56+)

The API name is **`ExperimentalStack`**, exported from the `expo-router` root. There is no `NativeStack v5` symbol. Details and limitations in §7.

```tsx
import { ExperimentalStack as Stack } from 'expo-router';
```

### 1.3 Android toolbar (experimental)

- Experimental toolbar support that mirrors the existing iOS functionality, exposed via `Stack.Toolbar`. See §7.

### 1.4 Streaming SSR for web

- The `unstable_useServerRendering` flag (set in the `expo-router` config plugin) enables **streaming server-side rendering** for web.
- A new `generateMetadata` function fetches and configures metadata on initial page loads. It complements the existing `<Head>` component, which handles metadata updates **after hydration**. See §12.

### 1.5 Data-loader helpers

Two helpers narrow the loader callback signature (both from `expo-router/server`, which re-exports `expo-server` — see §12):

- `createStaticLoader` — receives **only route parameters**.
- `createServerLoader` — **always passes a request** and **throws** if misused during static generation.

> Data loaders themselves are **alpha and available in SDK 55 and later** (`docs/pages/router/web/data-loaders.mdx` frontmatter `isAlpha: true`, line 13); SDK 56 adds these two helpers, not the feature.

### 1.6 Customizable Suspense fallbacks

Export a `SuspenseFallback` function from a `_layout` route to customize the loading UI across the app. The component receives `{ route, params }`:

```tsx
import type { SuspenseFallbackProps } from 'expo-router';

export function SuspenseFallback({ route, params }: SuspenseFallbackProps) {
  // route: the module's contextKey, e.g. './profile/[id].tsx'
  // params: Record<string, string | string[]>
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <ActivityIndicator size="large" />
    </View>
  );
}
```

> Custom `SuspenseFallback` exports **do not work with async routes** (`docs/pages/router/web/async-routes.mdx`). Type source: `packages/expo-router/src/views/SuspenseFallback.tsx`.

---

## 2. Introduction

Source: https://docs.expo.dev/router/introduction/

Expo Router is *"an open-source routing library for Universal React Native applications built with Expo."* It is a **file-based router** for Android, iOS, and web, built on top of React Native Screens.

Key features:
- **Native & optimized** — built on React Native Screens with lazy-evaluation in production.
- **Deep linking** — *"Every screen in your app is automatically deep linkable."*
- **Offline-first** — apps cache locally and handle native URLs without network connectivity.
- **Universal** — unified navigation structure across all platforms, with platform-specific API access when needed.
- **Discoverable** — supports static web rendering and search-engine indexing.
- **Fast development** — Universal Fast Refresh and artifact memoization across platforms.

**Project-creation note:** `create-expo-app@latest` *without* `--template` still creates **SDK 54** projects (the docs word this as "the SDK 56 transition period" — `docs/pages/router/introduction.mdx:39`, identical text on the `sdk-56` and `sdk-57` branches). Pin the template explicitly:
```sh
npx create-expo-app@latest --template default@sdk-56   # SDK 56
npx create-expo-app@latest --template default@sdk-57   # SDK 57
```
SDK 54 is the last release with an Expo Go store build; **SDK 56 and SDK 57 require development builds** for device testing.

---

## 3. Installation

Source: https://docs.expo.dev/router/installation/

### Required dependencies
```sh
npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar
```

### Entry point
Set in `package.json`:
```json
{ "main": "expo-router/entry" }
```
Root layout file lives at `src/app/_layout.tsx` (or `app/_layout.tsx` without a `src` directory).

### Optional custom entry point
Create a custom entry file (e.g. `index.js`) to initialize services before the app loads:
- Import side effects / services first.
- End the file with `import 'expo-router/entry';`.
- Point `package.json`'s `main` at this file.

### App config (`app.json` / `app.config.js`)
```json
{
  "expo": {
    "scheme": "your-app-scheme",
    "experiments": { "typedRoutes": true }
  }
}
```

### Web (optional)
```sh
npx expo install react-native-web react-dom
```
```json
{ "expo": { "web": { "bundler": "metro" } } }
```

### Babel config
Use `babel-preset-expo` in `babel.config.js`, or delete the file if no custom config is needed.

### Path aliases (with `src` directory) — `tsconfig.json`
```json
{ "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
```

### Final steps
```sh
npx expo start --clear
```
Remove outdated Yarn resolutions / npm overrides for `metro`, `metro-resolver`, and `react-refresh`.

---

## 4. Core Concepts (File-Based Routing)

Source: https://docs.expo.dev/router/basics/core-concepts/

The seven foundational rules:

1. **Routes live in `src/app`** — *"All navigation routes in your app are defined by the files and sub-directories inside the src/app directory."* Each file exports a page, except special `_layout` files. Directories group related screens.
2. **Universal deep linking** — every page gets a URL matching its file path; navigable via browser URL bar (web) or deep links (mobile).
3. **Index files define the initial route** — the first `index.tsx` matching `/` becomes the start screen (default: `src/app/index.tsx`). Route groups (parentheses dirs) **do not affect the URL**.
4. **Root `_layout` replaces `App.jsx`** — *"Every project should have a _layout.tsx file directly inside the src/app directory."* It renders before all routes and holds init code (fonts, theme, splash screen, providers).
5. **Platform-specific implementation** — uses native tabs on Android/iOS and custom tabs on web via **platform-specific file extensions** (e.g. `app-tabs.native.tsx` vs `app-tabs.tsx`).
6. **Non-route code stays outside `src/app`** — put components/hooks/utils in `src/components`, `src/hooks`, `src/constants`; anything in `src/app` is treated as a route.
7. **Navigator customization** — Stack and Tab navigators support headers, animations, gestures, etc.

File-structure example:
```
src/app/index.tsx                  → initial route (/)
src/app/home.tsx                   → /home
src/app/_layout.tsx                → root layout
src/app/profile/friends.tsx        → /profile/friends
src/components/app-tabs.native.tsx → platform-specific, not a route
```

Conventions:
- `_layout.tsx` — layout for the directory (special, not a route).
- `index.tsx` — the directory's `/` route.
- `[param].tsx` — dynamic route segment.
- `[...rest].tsx` — catch-all / rest segment.
- `(group)/` — route group (parentheses dir; does not appear in the URL).
- `name+api.ts` — API route (see §9).
- `+html.tsx` — root HTML wrapper for static rendering (see §10).
- `+not-found.tsx` — fallback for unmatched routes.

---

## 5. Layouts (`_layout` files)

Source: https://docs.expo.dev/router/basics/navigation-layouts

- Each directory in `src/app` can define a `_layout.tsx` that *"defines how all the pages within that directory are arranged."* It exports a default component rendered before navigating to any page in that directory.
- **Root layout** `src/app/_layout.tsx` is the navigation entry point — *"this file is where you would put initialization code that may have previously gone inside an App.jsx file, such as loading fonts, interacting with the splash screen, or adding context providers."*
- **`Stack`** implements the native stack; files in the directory automatically become routes. Optional `Stack.Screen` children configure options, matched by the `name` prop.
- **Tabs** — two implementations:
  - **JavaScript tabs**: the `Tabs` component with `Tabs.Screen` children (cross-platform). `import { Tabs } from 'expo-router/js-tabs';` — the root `expo-router` export still works but is `@deprecated` in SDK 56.
  - **Native tabs**: the `NativeTabs` component (Android/iOS) with `NativeTabs.Trigger` children for platform-native behavior. `import { NativeTabs } from 'expo-router/unstable-native-tabs';` — see §8.
  - Platform extensions (`.native.tsx` vs `.tsx`) select per-platform implementations.
- **`Drawer`** — `import { Drawer } from 'expo-router/drawer';` (not a root export).
- **`Slot`** — *"a placeholder for the current child route"* — used for layouts without a navigator (e.g. wrapping routes with a header/footer while replacing rather than stacking pages).
- **`Stack.Protected`** — route guard: `<Stack.Protected guard={isSignedIn}>…</Stack.Protected>` hides the wrapped screens from the navigator and the linking config while `guard` is false (`packages/expo-router/src/views/Protected.tsx`).

Guidance: avoid unnecessary nested navigators; reuse the same parent navigator unless nesting is genuinely required.

Minimal root layout:
```tsx
import { Stack } from 'expo-router';

export default function Layout() {
  return <Stack />;
}
```

### 5.1 Custom navigators (alpha — SDK 56, `expo-router@56.2.10`+)

Source: `docs/pages/router/advanced/custom-navigators.mdx` line 11 — *"in alpha and available in SDK 56 and later."*

Root exports from `'expo-router'`:
- `unstable_createStandardRouterNavigator(NavigatorContent, router)` — app developers building a navigator for one app.
- `unstable_integrateWithRouter` — library authors shipping a reusable navigator for both Expo Router and React Navigation.
- Routers `StackRouter` / `TabRouter`, and types `IntegrateWithRouterOptions`, `NavigatorContentProps`, `StandardNavigatorEventMapBase`, `StandardUseNavigationBuilderOptions`, `StackNavigationState`, `StackRouterOptions`, `TabNavigationState`, `TabRouterOptions`.

**Version caveat:** these exports are **absent** from `expo-router@56.2.8` and `56.2.9` (the version the *frozen docs schema* pins) and first appear in **`56.2.10`** (published 2026-06-10), alongside the new `standard-navigation ^0.0.5` dependency. The `sdk-56` release branch now pins `~56.2.16`, so a normal `npx expo install` on SDK 56 gets them. Verified by grepping `build/exports.js` in the published tarballs for 56.2.8 / 56.2.9 (0 hits) vs 56.2.10 / 56.2.12 / 56.2.14 / 56.2.16 (present). If a lockfile pins exactly `56.2.9`, bump it — this is an SDK 56 feature, **not** a reason to upgrade to SDK 57.

---

## 6. Navigation (`<Link>`, `useRouter`, params)

Source: https://docs.expo.dev/router/basics/navigation/

### `<Link>` props
- `href` — route path (string, or object `{ pathname, params }`).
- `asChild` — wrap a `Pressable`/custom component while preserving layout control.
- `prefetch` — preload the target screen when the component renders.
- `withAnchor` — force loading the initial route when navigating within stacks.
- `replace` — replace the current route (referenced; behaves like `router.replace`).

```tsx
<Link href="/about">About</Link>

<Link href={{ pathname: '/user/[id]', params: { id: 'bacon' } }}>
  View user
</Link>

<Link href="/about" asChild>
  <Pressable><Text>Go</Text></Pressable>
</Link>
```

### `useRouter` hook
```tsx
const router = useRouter();
router.navigate('/about'); // push a new page OR unwind to an existing route
router.push('/about');     // always push a new page
router.replace('/about');  // replace current page
router.back();             // go to previous page
router.setParams(params);  // update query params without navigating
```
Stack dismissal methods (see §7): `router.dismiss(count)`, `router.dismissTo(href)`, `router.dismissAll()`, `router.canDismiss()`.

### URL parameter hooks
```tsx
import { useLocalSearchParams } from 'expo-router';
const { id, limit } = useLocalSearchParams(); // all params, incl. those passed as params
```
- `useGlobalSearchParams()` — global URL params (referenced).
- Relative routes use `./` and `../` prefixes (not supported when typed routes are enabled — see §11).
- `<Redirect>` immediately navigates without rendering the current page.
- `initialRouteName` (in layout settings) sets the initial route.

---

## 7. Stack Navigator

Source: https://docs.expo.dev/router/advanced/stack/

`<Stack>` from `expo-router` is *"the foundational way of navigating between routes in an app."*

```tsx
import { Stack } from 'expo-router';

export default function Layout() {
  return <Stack />;
}
```

### Configuring screens
```tsx
<Stack.Screen name="home" options={{}} />
```
Screens can also set options from within their own component via `<Stack.Screen options={{ ... }} />`.

### Key `screenOptions` / `options`
| Option | Purpose |
|---|---|
| `title` | Fallback for `headerTitle`. |
| `headerShown` | Boolean — header visibility. |
| `presentation` | Presentation mode (`card`, `modal`, `formSheet`, etc.). |
| `animation` | Transition animation type. |
| `headerStyle` | Header style object (supports `backgroundColor`). |
| `headerTintColor` | Tint for back button and title. |
| `headerTitle` | Custom title element or string. |
| `headerLeft` / `headerRight` | Custom header button components. |
| `headerLargeTitle` | Collapsible large title (iOS). |
| `headerSearchBarOptions` | Native iOS search bar. |

### Custom push behavior
```tsx
<Stack.Screen name="[profile]" getId={({ params }) => String(Date.now())} />
```

### Dismissal API
- `router.dismiss(count)` — remove screens from the closest stack.
- `router.dismissTo(href)` — pop until reaching the given route.
- `router.dismissAll()` — return to the first screen.
- `router.canDismiss()` — whether dismissal is possible.

### Route guards
```tsx
<Stack.Protected guard={isSignedIn}>
  <Stack.Screen name="(app)" />
</Stack.Protected>
```
`ProtectedProps = { guard: boolean; children?: ReactNode }` (`packages/expo-router/src/views/Protected.tsx`). While `guard` is `false` the wrapped screens are removed from the navigator **and** from the linking config, so they are not deep-linkable.

### Stack Toolbar (SDK 56)
`Stack.Toolbar` provides an iOS header toolbar with **Liquid Glass** support; SDK 56 adds **experimental Android toolbar** support mirroring iOS. See `/router/advanced/stack-toolbar`.

**SDK 56 GA Android limits** (`docs/pages/router/advanced/stack-toolbar.mdx` @ SDK 56 cut): `Stack.Toolbar.Button` renders **only its icon** on Android — both `Stack.Toolbar.Badge` and `Stack.Toolbar.Label` children are dropped. Badges work only in **iOS** header placements (`left`/`right`), never in the bottom toolbar. *(The Android badge restriction was lifted in a patch, not in SDK 57 itself: `ToolbarItemBadge.android.js` first appears in `expo-router@56.2.12` and, on the 57 line, only in `57.0.2` — it is absent from 57.0.0 and 57.0.1. See the delta section.)*

### iOS 26+ Liquid Glass opt-out
- Set `UIDesignRequiresCompatibility: true` in the app config, **or**
- Use the JS stack: `import { Stack as JsStack } from 'expo-router/js-stack';` (this entry point is the SDK 56 replacement for `@react-navigation/stack`).

### `ExperimentalStack` (SDK 56+, alpha)
Source: `docs/pages/versions/v56.0.0/sdk/router/experimental-stack.mdx` (`isAlpha: true`); export at `packages/expo-router/src/exports.ts`.

```tsx
import { ExperimentalStack as Stack } from 'expo-router';

export default function Layout() {
  return (
    <Stack screenOptions={{ headerShown: true }}>
      <Stack.Screen name="index" options={{ title: 'Home' }} />
    </Stack>
  );
}
```
A sibling to `Stack` powered by `react-native-screens/experimental`. Opt-in **per navigator**. Also exposes `ExperimentalStack.Screen` / `ExperimentalStack.Protected`, and types `ExperimentalStackNavigationOptions | NavigationEventMap | NavigationProp | ScreenProps`.

Hard constraints — a model must not assume parity with `Stack`:
- **Only four screen options are honored**: `title`, `headerShown`, `headerTransparent`, `headerBackVisible`. Anything else (`headerLeft`, `headerRight`, `headerTitle`, `headerStyle`, `headerTintColor`, `animation`, status bar options) logs a dev warning and is **ignored**.
- No `presentation: 'modal'` / `'transparentModal'`; no `formSheet` or detents; no custom header components.
- **Android**: cannot coexist with the standard `Stack` in the same app — pick one. Predictive back also requires `android.predictiveBackGestureEnabled: true` in the app config.
- **Web**: falls back to the standard `Stack`, so the same layout is cross-platform.

---

## 8. Tabs

Source: https://docs.expo.dev/router/advanced/tabs/

`<Tabs>` creates a tab navigator with a bottom tab bar by default; `<Tabs.Screen>` defines individual tabs (takes `name` + `options`).

```tsx
import { Tabs } from 'expo-router/js-tabs'; // root `expo-router` export is deprecated in SDK 56

export default function Layout() {
  return <Tabs screenOptions={{ tabBarActiveTintColor: 'blue' }} />;
}
```

### Key tab options
| Prop | Purpose |
|---|---|
| `tabBarIcon` | Function `({ color, focused, size }) => ReactElement`. |
| `title` | Tab display text. |
| `tabBarActiveTintColor` | Active icon/label color. |
| `tabBarInactiveTintColor` | Inactive icon/label color. |
| `href` | Route destination; set to `null` to **hide** the tab from the bar (route stays accessible). |

### Hide a tab
```tsx
<Tabs.Screen name="index" options={{ href: null }} />
```

### Dynamic route as a tab
```tsx
<Tabs.Screen name="[user]" options={{ href: '/evanbacon' }} />
```

### Native tabs (`NativeTabs`) — Android/iOS

The default tab experience in the SDK 56/57 templates. **Not** a root export:

```tsx
import { NativeTabs } from 'expo-router/unstable-native-tabs';

export default function TabLayout() {
  return (
    <NativeTabs>
      <NativeTabs.Trigger name="index">
        <NativeTabs.Trigger.Label>Home</NativeTabs.Trigger.Label>
        <NativeTabs.Trigger.Icon sf="house.fill" md="home" />
      </NativeTabs.Trigger>
      <NativeTabs.Trigger name="settings">
        <NativeTabs.Trigger.Icon sf="gear" md="settings" />
        <NativeTabs.Trigger.Label>Settings</NativeTabs.Trigger.Label>
      </NativeTabs.Trigger>
    </NativeTabs>
  );
}
```

Compound API (SDK 55+ form): `NativeTabs.Trigger`, `NativeTabs.Trigger.Label`, `.Icon`, `.Badge`, `.VectorIcon`, `NativeTabs.BottomAccessory`. The standalone `Icon` / `Label` / `Badge` / `VectorIcon` values the SDK 54 docs import from `expo-router/unstable-native-tabs` are no longer exported from that subpath — they live at the **`expo-router` root**, and the compound members are aliases of them (see the import cheat sheet).

Added across the SDK 56 patch line (`packages/expo-router/CHANGELOG.md`):

| Version | Addition |
|---|---|
| 56.0.3 | `unstable_nativeProps` on the `NativeTabs` host (escape hatch to `react-native-screens` props). |
| 56.2.0 | `disabled` on `NativeTabs.Trigger` (**Android + iOS** — both declaration sites in `src/native-tabs/types.ts` carry `@platform android` *and* `@platform ios`; it only suppresses the native tap, so `router.push()` / `<Link />` still navigate); `tabBarRespectsIMEInsets` on `NativeTabs` (Android only). |
| 56.2.1 | Android `selectedIcon`: `drawable` / `md` accept `{ default, selected }`, matching iOS `sf` / `xcasset`. |
| 56.2.3 | `EXPO_ROUTER_DISABLE_NATIVE_TABS_MD=1` env var disables Material Symbols (`md`) icons on Android. |
| 56.2.6 | Per-tab Android props on `NativeTabs.Trigger`: `rippleColor`, `indicatorColor`, `disableIndicator`, `labelVisibilityMode`. |

Icon prop selection: `sf` (SF Symbol, iOS), `xcasset` (iOS asset), `md` (Material Symbol, Android), `drawable` (Android drawable), `src` (image source).

---

## 9. API Routes

Source: https://docs.expo.dev/router/web/api-routes

- File convention: `name+api.ts` in the `app` directory (e.g. `hello+api.ts` → `/hello`).
- Export HTTP method functions: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. *"Making requests with an undefined method will automatically return `405: Method not allowed`. If an error is thrown during the request, it will automatically return `500: Internal server error`."*
- **API route filenames cannot use platform extensions** — `hello+api.web.ts` will not work.

```ts
export function GET(request: Request) {
  return Response.json({ hello: 'world' });
}
```

### Request handling (standard `Request`)
```ts
export async function POST(request: Request) {
  const body = await request.json();
  return Response.json({ /* ... */ });
}
```
Query params:
```ts
const url = new URL(request.url);
const post = url.searchParams.get('post');
```

### Dynamic API routes
Use `[param]+api.ts`; params arrive as the **second handler argument**:
```ts
export async function GET(request: Request, { post }: Record<string, string>) { /* ... */ }
```

### Error handling — `StatusError` from `expo-server`
```ts
import { StatusError } from 'expo-server';

export async function GET(request: Request, { post }: Record<string, string>) {
  if (!post) {
    throw new StatusError(404, 'No post found');
  }
}
```
`StatusError` always produces a JSON body with an `error` key. To bypass that wrapper (e.g. redirects), **throw a `Response` directly** — it replaces the resolved response as-is:
```ts
throw Response.redirect('https://expo.dev', 302);
```

### `expo-server` runtime helpers (SDK 54+)
`npx expo install expo-server`. These are request-scoped — callable only in server code during an ongoing request, and usable outside API routes too (e.g. in `+middleware.ts`, loaders):
- `origin(): string | null` — the **current request's URL** (on some platforms, the origin of the request). Per its TSDoc it deliberately **does not** use the `Origin` header in development, "as it may contain an untrusted value" (`packages/expo-server/src/runtime/api.ts`).
- `environment(): string | null` — the request's environment name, or `null` for production. On EAS Hosting this is the alias or deployment identifier; other providers may report something else.
- `requestHeaders()` — an `ImmutableHeaders` copy of the incoming request headers.
- `setResponseHeaders({ ... })` or `setResponseHeaders(headers => { ... })` — mutate the outgoing response headers.

### Server middleware (`+middleware.ts`) — alpha, SDK 54+
Runs for **every** server request, before routes. Enable with the `unstable_useServerMiddleware` config-plugin flag; requires `web.output: 'server'`. Client-side navigation (native, or `<Link />` on web) does **not** pass through middleware. Type: `MiddlewareFunction` from `expo-router/server`. See `docs/pages/router/web/middleware.mdx`.

### Deployment
```sh
npx expo export --platform web
```
Deploy to EAS Hosting, Express, Bun, Netlify, Vercel, or other WinterCG-compliant environments.

---

## 10. Static Rendering

Source: https://docs.expo.dev/router/web/static-rendering/

### Enable static output
```json
{ "expo": { "web": { "output": "static" } } }
```
For request-time dynamic rendering, use **server rendering** with `web.output: 'server'` instead (see §12).

### `generateStaticParams`
A **server-only function evaluated at build-time** in Node.js by Expo CLI. Has env vars + filesystem access, but no browser APIs or native Expo modules. *"`generateStaticParams` cascades from nested parents down to children. The cascading parameters are passed to every dynamic child route **that exports `generateStaticParams`**."*

### `+html.tsx`
Root HTML wrapper for all routes. *"This file exports a React component that only ever runs in Node.js, which means global CSS cannot be imported inside of it."* It is never rehydrated on the client and must render its `children` prop.

Under **server rendering** a hand-written `+html.tsx` must consume `useServerDocumentContext()` and use **all four** returned values, or metadata, fonts and CSS silently drop out of the SSR HTML:

```tsx
import { ScrollViewStyleReset, useServerDocumentContext } from 'expo-router/html';

export default function Root({ children }: { children: React.ReactNode }) {
  const { htmlAttributes, bodyAttributes, headNodes, bodyNodes } = useServerDocumentContext();
  return (
    <html lang="en" {...htmlAttributes}>
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no" />
        <ScrollViewStyleReset />
        {headNodes}
      </head>
      <body {...bodyAttributes}>
        {children}
        {bodyNodes}
      </body>
    </html>
  );
}
```
- `htmlAttributes` / `bodyAttributes` — attributes to spread onto `<html>` / `<body>`.
- `headNodes` — metadata, CSS and other head assets (this is where `generateMetadata` output lands).
- `bodyNodes` — fonts and other deferred assets.

Source: `docs/pages/router/web/server-rendering.mdx` ("Root HTML").

### `<Head>` component
```tsx
import Head from 'expo-router/head';
```
Useful for SEO — static head elements are rendered ahead of time and can also be updated dynamically (post-hydration; see §12 comparison with `generateMetadata`).

---

## 11. Typed Routes

Source: https://docs.expo.dev/router/reference/typed-routes/

### Enable
```json
{ "expo": { "experiments": { "typedRoutes": true } } }
```
Then run `npx expo customize tsconfig.json` and start with `npx expo start`.

Types are generated automatically when the dev server starts, stored in the git-ignored `expo-env.d.ts`. *"This enables `<Link>`, and the hooks API to be statically typed."*

### `Href<T>` usage
```tsx
<Link href="/about" />
<Link href="/user/1" />
<Link href={`/user/${id}`} />
<Link href={("/user" + id) as Href} />
<Link href={{ pathname: "/user/[id]", params: { id: 1 } }} /> // dynamic
```

### Key limitation
*"Statically typed routes do not support relative paths. You'll need to use absolute paths for all routes."* Use `useSegments()` to build dynamic absolute paths.

### Typed params
```tsx
const { profile, search } = useLocalSearchParams<'/(search)/[profile]/[...search]'>();
```
Query params can be typed manually or as a second generic argument.

---

## 12. Data Loaders & Server Rendering (SDK 55+, alpha)

Sources: https://docs.expo.dev/router/web/data-loaders · https://docs.expo.dev/router/web/server-rendering

> **Status:** data loaders are an **alpha** API available in **SDK 55 and later** — not new in SDK 56 (`docs/pages/router/web/data-loaders.mdx` frontmatter `isAlpha: true`, line 13). SDK 56 adds the `createStaticLoader` / `createServerLoader` helpers. The `expo-server` runtime API and `+middleware.ts` are SDK 54+, also alpha. Loaders require `web.output: 'static'` **or** `'server'`.

All loader/metadata symbols come from `expo-router/server`. That subpath is the package-root shim `packages/expo-router/server.js` (`export { createStaticLoader, createServerLoader } from 'expo-server';`) plus `server.d.ts`, which re-exports the `expo-server` types and adds a local `RequestHandler`. There is **no** `src/server/index.ts` — the only file under `src/server/` is `ServerDocument.tsx`.
```ts
import {
  createStaticLoader,
  createServerLoader,
  type LoaderFunction,
  type GenerateMetadataFunction,
  type ImmutableRequest,
  type Metadata,
  type MiddlewareFunction,
} from 'expo-router/server';
```
`useLoaderData` is the exception — it comes from `expo-router`.

### `loader` + `useLoaderData`
Export a `loader` function from a route file and read it with `useLoaderData`:
```tsx
import { useLoaderData } from 'expo-router';

export async function loader() {
  const response = await fetch('https://api.example.com/data');
  return response.json();
}

export default function Home() {
  const data = useLoaderData<typeof loader>();
  return <View><Text>Data: {JSON.stringify(data)}</Text></View>;
}
```

### `createStaticLoader` — params only
```tsx
import { createStaticLoader } from 'expo-router/server';

export const loader = createStaticLoader(async params => {
  const response = await fetch(`https://api.example.com/posts/${params.postId}`);
  return response.json();
});
```

### `createServerLoader` — request + params (throws if misused during static generation)
```tsx
import { createServerLoader } from 'expo-router/server';

export const loader = createServerLoader(async (request, params) => {
  const authToken = request.headers.get('Authorization');
  // ... handle authentication
});
```

### Loader signature & rendering modes
- Signature: `loader(request, params)` — params are the second argument.
- **Static rendering**: loaders run at build time; `request` is `undefined`.
- **Server rendering**: *"Loaders execute on every request. The `request` parameter contains an immutable version of the incoming HTTP request."*

### Constraints
- *"Loaders must return JSON-serializable data."*
- Loader code is **dropped from the client bundle**.
- Env vars in loaders never expose secrets to clients.

### Config — both flags are required
```json
{
  "expo": {
    "web": { "output": "server" },
    "plugins": [
      ["expo-router", {
        "unstable_useServerDataLoaders": true,
        "unstable_useServerRendering": true
      }]
    ]
  }
}
```
- `unstable_useServerDataLoaders` — **required for loaders to run at all**. Works with `web.output: 'static'` or `'server'`. Omitting it is the most common reason a `loader` export is silently ignored.
- `unstable_useServerRendering` — activates streaming server rendering; HTML is generated per request rather than pre-generated. Only needed for `web.output: 'server'`.

Source: `docs/pages/router/web/data-loaders.mdx` Steps 1–2; `packages/expo-router/plugin/src/withRouter.ts`.

### `generateMetadata` (server-side, before render)
```tsx
import type { GenerateMetadataFunction } from 'expo-router/server';

export const generateMetadata: GenerateMetadataFunction = async (request, params) => {
  const response = await fetch(`https://api.example.com/posts/${params.id}`);
  const post = await response.json();

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: post.coverImage,
    },
  };
};
```
Receives the incoming request and route params. *"generateMetadata is the recommended approach for server rendering because it resolves metadata before the HTML stream begins,"* whereas the `<Head>` component from `expo-router/head` updates metadata **after client hydration**.

### `SuspenseFallback` (SDK 56)
Export from a `_layout` route to customize the app-wide Suspense loading UI — signature, props and the async-routes caveat are in §1.6.

---

## Not covered in depth here

Exists in SDK 56, deliberately out of scope for this file — do not assume it is missing from the framework:

| Feature | Import / file | Doc |
|---|---|---|
| `Link.Preview` / `Link.Trigger` / `Link.Menu`, `useIsPreview` | `expo-router` | `/router/reference/link-preview` |
| Apple zoom transitions (`usePreventZoomTransitionDismissal`, `LinkAppleZoomProps`) | `expo-router` | `/router/advanced/stack` |
| `unstable_navigationEvents` (`pagePreloaded` / `pageFocused` / `pageBlurred` / `pageRemoved`, each with `segments` since 56.2.3) | `expo-router` | — |
| `useCurrentRouteInfo` (56.1.2), `useRoute` (56.1.1) | `expo-router` | — |
| Config-plugin `redirects` / `rewrites` / `headers` / `platformRoutes` / `asyncRoutes` | `app.json` plugins | `/router/reference/redirects` |
| Testing library (`renderRouter`, `screen`) | `expo-router/testing-library` | `/router/reference/testing` |
| `SplitView` | `expo-router/unstable-split-view` (there is **no** `expo-router/split-view` subpath — the package only ships `unstable-split-view.{js,d.ts}`) | `/versions/latest/sdk/router/split-view` |
| RSC (experimental) | `expo-router/rsc` | — |

---

## Source URLs
- Changelog (Router section): https://expo.dev/changelog/sdk-56
- Migration SDK 55→56: https://docs.expo.dev/router/migrate/sdk-55-to-56
- Introduction: https://docs.expo.dev/router/introduction/
- Installation: https://docs.expo.dev/router/installation/
- Core concepts: https://docs.expo.dev/router/basics/core-concepts/
- Navigation layouts: https://docs.expo.dev/router/basics/navigation-layouts
- Navigation: https://docs.expo.dev/router/basics/navigation/
- Stack: https://docs.expo.dev/router/advanced/stack/
- Tabs: https://docs.expo.dev/router/advanced/tabs/
- API routes: https://docs.expo.dev/router/web/api-routes
- Static rendering: https://docs.expo.dev/router/web/static-rendering/
- Server rendering: https://docs.expo.dev/router/web/server-rendering
- Data loaders: https://docs.expo.dev/router/web/data-loaders
- Typed routes: https://docs.expo.dev/router/reference/typed-routes/

---

## SDK 57 delta (and 56.2.x backports)

Expo Router changed **substantially** after the SDK 56 GA cut: a new custom-navigator API, native tabs rewritten on a new navigation core, an `unstable_nativeProps` escape hatch on the native stack, Android toolbar badges, and one Android behavior change. Everything in the sections above still applies unless contradicted here.

**Headline finding: for `expo-router`, the 56 → 57 delta is essentially just the version number.** Every JS/TS API change below is present in `expo-router@56.2.16` as well as in `57.0.x`. Nothing in this section is a reason to upgrade an SDK 56 app to SDK 57. Upgrade for the platform-level 57 changes documented elsewhere (React Native 0.86, Reanimated 4.5, etc.), not for Router.

> **Read this before treating anything below as "57-only".** Almost every item was **backported into the `expo-router` 56.2.x patch line** and is present in `expo-router@56.2.16` — the current `sdk-56` dist-tag, and exactly what `origin/sdk-56:packages/expo/bundledNativeModules.json` pins. Each bullet is annotated with the first 56.2.x patch that carries it. Verified by unpacking the published tarballs (`npm pack expo-router@<version>`), not from the monorepo. **The practical advice for nearly everything below is "bump your SDK 56 patch", not "upgrade to SDK 57".**

Evidence base: `git show origin/sdk-56:packages/expo-router/CHANGELOG.md` vs `origin/sdk-57:` (`## 57.0.0 — 2026-06-25`), cross-checked against published npm tarballs for 56.2.8–56.2.16 and 57.0.0–57.0.8.

> **Do not** treat monorepo commit `15e7ddcc2d3` ("the SDK 57 cut") as what shipped as `expo-router@57.0.0`. It is ahead of the release branch, and several things present there were never in any published 57.0.x — most notably the `package.json#exports` map (added on main by #46801, 2026-06-17) and the `StandardUseNavigationBuilderOptions` form of `NativeTabsProps.screenListeners`. Also **do not** use `docs/pages/versions/v57.0.0/sdk/router/*` — it differs from v56 only by `sourceCodeUrl`, and `docs/public/static/data/v57.0.0/expo-router.json` is staler than the v56 one.

### Breaking (relative to the SDK 56 GA cut — all three also shipped in 56.2.x)

- **Native tabs rewritten on `standard-navigation`** (#46456, "Add `standard-navigation` integration"; new dep `standard-navigation ^0.0.5`). **Also in 56.2.10+** — the dep and the rewrite are in `56.2.10` through `56.2.16`, so this is not a 56→57 delta for anyone on a current SDK 56 patch.
  - `NativeTabTriggerProps.listeners` is now `ScreenProps<any, TabNavigationState<ParamListBase>, NativeTabNavigationEventMap>['listeners']` (was `ScreenListeners<NavigationState, EventMapBase> | ((prop: { route }) => …)`). Present in both `56.2.10+` and all `57.0.x`. Runtime behavior is compatible; **hand-annotated listener types will fail to compile**. Prefer inferring the parameter types.
  - `NativeTabsProps.screenListeners` did **not** change. In `56.2.9`, `56.2.16`, `57.0.0` and `57.0.8` it is still `ScreenListeners<TabNavigationState<ParamListBase>, NativeTabNavigationEventMap> | ((prop: { route: RouteProp<ParamListBase, string> }) => ScreenListeners<…>)` (`build/native-tabs/types.d.ts`). The `StandardUseNavigationBuilderOptions` form exists only on the monorepo main branch and has not shipped.
- **`tabPress` payload changed** (#46445) — **also in 56.2.10+**. `NativeTabNavigationEventMap.tabPress.data` is now `{ __internalTabsType: 'native'; isPrevented: boolean }`, and `OnTabChangeEventPayload` gained `isPrevented?: boolean` (default `false`). A `disabled` tab now **emits `tabPress` with `isPrevented: true` and performs no navigation** — at the SDK 56 GA cut it emitted nothing. Listeners that assumed "`tabPress` fired ⇒ navigation happened" must now check `isPrevented`.
- **[android] Navigation-state restoration across activity recreation was removed** (#47422) — **also in 56.2.14+** (the `storeRef.current.state` restore branch is gone from `build/global-state/useStore.js` in 56.2.14 onward). The restore path in `src/global-state/useStore.ts` / `store.ts` / `fork/useLinking.native.ts` is gone. It is safe only because the companion change (#47423) adds `assetsPaths` to the `MainActivity` `android:configChanges`, so the activity is no longer recreated on dynamic-color changes. **That template change is on the `sdk-56` release branch too** — `templates/expo-template-bare-minimum/android/app/src/main/AndroidManifest.xml` has the identical `…|smallestScreenSize|assetsPaths` string on `origin/sdk-56` and `origin/sdk-57`, so it is *not* an SDK 57 delta. **Migration:** bare / manually-managed Android projects that do not regenerate `AndroidManifest.xml` will silently **lose navigation state** on config changes — this bites SDK 56 projects on 56.2.14+ exactly as it does SDK 57 ones. Re-run prebuild, or hand-edit `android:configChanges` to end with `|assetsPaths`.

### Changes since the SDK 56 GA cut (57 + backported 56.2.x)

Nothing in this list is 57-exclusive: every item below is already in `expo-router@56.2.16`. The first-carrying 56.2.x patch is noted on each bullet.

- **`unstable_nativeProps` on native `Stack` screen options** (#47482) — **also in 56.2.14+** — escape hatch to `react-native-screens` props Expo Router does not expose. `packages/expo-router/src/react-navigation/native-stack/types.tsx`:
  ```ts
  type NativeStackScreenNativeProps = Partial<Omit<ScreenProps, 'children' | 'screenId' | 'activityState'>>;
  type NativeStackHeaderNativeProps = Partial<ScreenStackHeaderConfigProps>;
  type NativeStackNativeProps = NativeStackScreenNativeProps & { headerConfig?: NativeStackHeaderNativeProps };
  // NativeStackNavigationOptions.unstable_nativeProps?: NativeStackNativeProps   // android + ios
  ```
  It **overrides** anything Expo Router sets and may change in minor versions. (`NativeTabs` already had an equivalent host-level `unstable_nativeProps` in 56.0.3.)
- **[android] `Stack.Toolbar.Badge` now works** (#46537, #47276) — **in 56.2.12+ and 57.0.2+; NOT in 57.0.0 or 57.0.1** (`build/layouts/stack-utils/toolbar/ToolbarItemBadge.android.js` is absent there). This reverses the SDK 56 GA limitation in §7. Badges work in **header** placements (`left`/`right`) on both platforms, including on `Stack.Toolbar.Menu` icons. Still unsupported: `Stack.Toolbar.Label` children on Android (the button renders icon + badge only), and badges in the **bottom** toolbar on either platform. The docs also moved to `<Stack.Toolbar.Button icon={process.env.EXPO_OS === 'ios' ? 'bell' : bellIcon}>` instead of a nested `<Stack.Toolbar.Icon sf="bell" />`.
- **`NativeTabs.Trigger` testing/a11y props** (#47472) — **also in 56.2.15+**: `testID` and `accessibilityLabel` on the trigger, plus `tabBarItemTestID` / `tabBarItemAccessibilityLabel` on `NativeTabOptions`. Platform nuance from the TSDoc — on iOS `testID` maps to the accessibility identifier (XCUITest and Maestro see it); on **Android** it maps to the view tag, which Detox reads but **Maestro and Appium do not** — match on `accessibilityLabel` there, which maps to `contentDescription` and needs API 26+.
- **`expo-router/drawer` re-exports the drawer building blocks** (#46635) — **also in 56.2.10+** — so a custom `drawerContent` no longer needs `@react-navigation/drawer`. Values: `DrawerContent`, `DrawerContentScrollView`, `DrawerItem`, `DrawerItemList`, `DrawerToggleButton`, `DrawerView`, `getDrawerStatusFromState`, `useDrawerStatus`, `useDrawerProgress`. Types: `DrawerContentComponentProps`, `DrawerHeaderProps`, `DrawerNavigationEventMap`, `DrawerNavigationOptions`, `DrawerNavigationProp`, `DrawerNavigatorProps`, `DrawerOptionsArgs`, `DrawerScreenProps`. `createDrawerNavigator` is **intentionally omitted** (use the `Drawer` layout), as are `DrawerStatusContext` / `DrawerProgressContext` (use the hooks).
- **`Theme` type** exported from the `expo-router` root (#47476) — **also in 56.2.15+** (`export type { Theme }` in `build/exports.d.ts`).
- **Config-plugin `Props` type resynced with the runtime schema** (#46677) — **also in 56.2.10+** — TS-only fix for typed `app.config.ts`. Before the fix the type was missing `platformRoutes`, `redirects`, `rewrites`, `disableSynchronousScreensUpdates`, still declared the removed `partialTypedGroups`, typed `origin` as `string` (now `string | boolean`) and `asyncRoutes` as a loose `string` (now `'development' | 'production' | boolean`, per-platform object allowed). `partialTypedGroups` → **`partialRouteTypes`**.
- **Custom navigators** — see §5.1; landed in `expo-router@56.2.10`, so it is an SDK 56 feature, not a 57 one.
- Web/SSR and platform fixes worth knowing — **all present on both lines**: async routes rehydrate synchronously by carrying through preloaded modules, removing FOUC in production web output (#46539, `56.2.10+`); [ios] white flash behind screens during the interactive swipe-back gesture fixed (#47121, `56.2.13+`); `extractExactPathFromURL` guards deep-link decoding against malformed percent-encoding (#47526, `56.2.15+`); [android] bottom-toolbar press detection fixed (#47389 — shipped in `56.2.14` and `57.0.4`, so it is **absent from 57.0.0–57.0.3**). These are patch-level fixes; check the specific patch you are on rather than assuming "SDK 57 has it".
- Testing library: `renderRouter` no longer ignores `overrides` / lists duplicate routes when an override key matches a file in `appDir` (#47287 — in `56.2.15` and `57.0.5`, so **not** in 57.0.0–57.0.4); a system time mocked with `jest.setSystemTime` is now preserved across `renderRouter` (#46978, `56.2.12+`).

> **No `package.json#exports` regression exists.** Earlier drafts of this file warned about `expo-router/head`, `expo-router/stack` and `expo-router/server` mis-resolving in 57.0.0 because of a new `exports` map. That is false for every published release: `'exports' in package.json` is `false` for `expo-router@56.2.9`, `56.2.16`, `57.0.0` and `57.0.8`, and `head.js`, `stack.js` and `server.js` are all present at the package root in each. The `exports` map exists only on the monorepo main branch (#46801).

### Unchanged in 57

`Link` / `useRouter` / `router` / params hooks, typed routes, API routes (`docs/pages/router/web/api-routes.mdx` diffs cosmetically only across the 57 window), data loaders and `generateMetadata`, `+html.tsx` / `useServerDocumentContext`, `ExperimentalStack`'s option surface, `Stack.Protected`, the `@react-navigation/*` → `expo-router/*` import mapping (still enforced).

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 |
|---|---|---|
| `expo-router` | `~56.2.16` | `~57.0.8` |
| `expo-server` | `~56.0.5` | `~57.0.1` |
| `expo-linking` | `~56.0.16` | `~57.0.4` |
| `expo-constants` | `~56.0.22` | `~57.0.7` |
| `expo-status-bar` | `~56.0.4` | `~57.0.1` |
| `react-native-screens` | `~4.26.0` | `~4.26.0` (unchanged) |
| `react-native-safe-area-context` | `~5.7.0` | `~5.7.0` (unchanged) |
| `react-native-web` | `~0.21.0` | `~0.21.0` (unchanged) |

Source: `git show origin/sdk-56:packages/expo/bundledNativeModules.json` vs `origin/sdk-57:` — the file that ships inside the `expo` package and that `npx expo install` resolves against.

Two traps:
- **The first-party `expo-*` pins are not flat `~57.0.0`.** SDK 57 ships `expo-router ~57.0.8`, `expo-server ~57.0.1`, `expo-linking ~57.0.4`, `expo-constants ~57.0.7`. Writing `~57.0.0` into `package.json` by hand will under-pin; use `npx expo install`.
- **`react-native-screens` is `~4.26.0` on *both* lines, not `4.25.2`.** The `4.25.2` value only appears in the frozen `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json`, which is stale for both SDKs (it also still says `expo-router ~56.2.9`). Do not source pins from those schemas, and do not assert a screens bump for SDK 57 — the bump (#47770) landed on the `sdk-56` line as well.
