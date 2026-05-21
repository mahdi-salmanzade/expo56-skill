# Expo Router (SDK 56) — Knowledge Base Reference

> Domain: **Expo Router** for Expo SDK 56. SDK 56 is a **MAJOR** release for Router: it no longer depends on React Navigation, adds an experimental Native Stack v5, experimental Android toolbar support, streaming SSR for web, data-loader helpers, and customizable Suspense fallbacks.

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
| `@react-navigation/stack` | `expo-router/js-stack` |
| `@react-navigation/bottom-tabs` | `expo-router/js-tabs` |
| `@react-navigation/material-top-tabs` | `expo-router/js-top-tabs` |
| Native Stack & Drawer | **No direct equivalent** — use the `Stack` and `Drawer` layouts instead |

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

### 1.2 Native Stack v5 (experimental)

- Built in partnership with `react-native-screens`.
- An updated experimental native stack version that introduces **Material-style headers** and **predictive back gesture** support.

### 1.3 Android toolbar (experimental)

- Experimental toolbar support that mirrors the existing iOS functionality, exposed via `Stack.Toolbar`.

### 1.4 Streaming SSR for web

- The `unstable_useServerRendering` flag (set in the `expo-router` config plugin) enables **streaming server-side rendering** for web.
- A new `generateMetadata` function fetches and configures metadata on initial page loads. It complements the existing `<Head>` component, which handles metadata updates **after hydration**.

### 1.5 Data-loader helpers

Two helpers narrow the loader callback signature:

- `createStaticLoader` — receives **only route parameters**.
- `createServerLoader` — **always passes a request** and **throws** if misused during static generation.

### 1.6 Customizable Suspense fallbacks

Export a `SuspenseFallback` function from a `_layout` route to customize the loading UI across the app:

```tsx
export function SuspenseFallback() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <ActivityIndicator size="large" />
    </View>
  );
}
```

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

**SDK 56 project-creation note:** During the SDK 56 transition, `create-expo-app@latest` *without* `--template` creates **SDK 54** projects. For SDK 56 use:
```sh
npx create-expo-app@latest --template default@sdk-56
```
SDK 54 supports Expo Go on physical devices; **SDK 56 requires development builds** for device testing.

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
  - **JavaScript tabs**: the `Tabs` component with `Tabs.Screen` children (cross-platform).
  - **Native tabs**: the `NativeTabs` component (Android/iOS) with `NativeTabs.Trigger` children for platform-native behavior.
  - Platform extensions (`.native.tsx` vs `.tsx`) select per-platform implementations.
- **`Slot`** — *"a placeholder for the current child route"* — used for layouts without a navigator (e.g. wrapping routes with a header/footer while replacing rather than stacking pages).

Guidance: avoid unnecessary nested navigators; reuse the same parent navigator unless nesting is genuinely required.

Minimal root layout:
```tsx
import { Stack } from 'expo-router';

export default function Layout() {
  return <Stack />;
}
```

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

### Stack Toolbar (SDK 56)
`Stack.Toolbar` provides an iOS header toolbar with **Liquid Glass** support; SDK 56 adds **experimental Android toolbar** support mirroring iOS. See `/router/advanced/stack-toolbar`.

### iOS 26+ Liquid Glass opt-out
- Set `UIDesignRequiresCompatibility: true` in the app config, **or**
- Import `Stack as JsStack from 'expo-router/js-stack'`.

### Native Stack v5 (SDK 56, experimental)
Built with `react-native-screens`; adds Material-style headers and predictive back gesture support.

---

## 8. Tabs

Source: https://docs.expo.dev/router/advanced/tabs/

`<Tabs>` creates a tab navigator with a bottom tab bar by default; `<Tabs.Screen>` defines individual tabs (takes `name` + `options`).

```tsx
import { Tabs } from 'expo-router';

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

> Note: SDK 56 also offers **native tabs** via `NativeTabs` / `NativeTabs.Trigger` (Android/iOS) — see `/versions/latest/sdk/router/native-tabs`.

---

## 9. API Routes

Source: https://docs.expo.dev/router/web/api-routes

- File convention: `name+api.ts` in the `app` directory (e.g. `hello+api.ts` → `/hello`).
- Export HTTP method functions: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. *"Unsupported methods will automatically return `405: Method not allowed`."*

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
Use `[param]+api.ts`; params are available in the handler.

### Error handling — `StatusError` from `expo-server`
```ts
import { StatusError } from 'expo-server';

export async function GET(request: Request) {
  if (!post) {
    throw new StatusError(404, 'No post found');
  }
}
```

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
A **server-only function evaluated at build-time** in Node.js by Expo CLI. Has env vars + filesystem access, but no browser APIs or native Expo modules. *"Cascades from nested parents down to children. The cascading parameters are passed to every dynamic child route."*

### `+html.tsx`
Root HTML wrapper for all routes. *"This file exports a React component that only ever runs in Node.js, which means global CSS cannot be imported inside of it."*

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

## 12. Data Loaders & Server Rendering (SDK 56)

Sources: https://docs.expo.dev/router/web/data-loaders · https://docs.expo.dev/router/web/server-rendering

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
export const loader = createStaticLoader(async params => {
  const response = await fetch(`https://api.example.com/posts/${params.postId}`);
  return response.json();
});
```

### `createServerLoader` — request + params (throws if misused during static generation)
```tsx
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

### Server rendering / streaming SSR config
```json
{
  "expo": {
    "web": { "output": "server" },
    "plugins": [
      ["expo-router", { "unstable_useServerRendering": true }]
    ]
  }
}
```
The `unstable_useServerRendering` flag activates streaming server rendering; HTML is generated dynamically on each request rather than pre-generated.

### `generateMetadata` (server-side, before render)
```tsx
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
Export from a `_layout` route to customize the app-wide Suspense loading UI (see §1.6):
```tsx
export function SuspenseFallback() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <ActivityIndicator size="large" />
    </View>
  );
}
```

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
