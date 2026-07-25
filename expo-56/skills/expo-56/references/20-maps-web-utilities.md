# Expo SDK 56 — Maps, Web Support & Utility APIs

Knowledge base reference covering the Expo-native maps module, web support / output modes, and utility APIs. All package docs target the SDK 56 path `https://docs.expo.dev/versions/v56.0.0/sdk/<pkg>/`; swap `v56.0.0` for `v57.0.0` for the SDK 57 page. SDK 57 differences are collected in [SDK 57 delta](#sdk-57-delta) at the end of this file.

---

## expo-maps

Source: https://docs.expo.dev/versions/v56.0.0/sdk/maps/

The newer Expo-native maps module. **Alpha status** — breaking changes are expected and it requires a development build (not available in Expo Go). Apple Maps is iOS-only; Google Maps is Android-only.

**This is not `react-native-maps`.** There is no `<MapView>`, no `region` / `initialRegion` prop, and no `<Marker>` / `<Polyline>` child components. You render `AppleMaps.View` / `GoogleMaps.View` and pass overlays as **array props** (`markers`, `polylines`, `polygons`, `circles`); the camera is `cameraPosition` (`{ coordinates, zoom }`) or the imperative `setCameraPosition` ref method.

### Installation

```sh
npx expo install expo-maps
```

Google Maps on Android requires a Google Cloud Maps API key, which goes in **app config**, not in the config plugin:

```json
{ "expo": { "android": { "config": { "googleMaps": { "apiKey": "YOUR_KEY" } } } } }
```

Creating a new development build is required for the key to take effect (`docs/pages/versions/v56.0.0/sdk/maps.mdx`, step 4).

The `expo-maps` config plugin handles **location permissions only**. Its entire props type is `{ requestLocationPermission?: boolean; locationPermission?: string }` (`packages/expo-maps/plugin/src/withMapsLocation.ts`). When `requestLocationPermission` is `true` it writes `NSLocationWhenInUseUsageDescription` (message from `locationPermission`) and adds `ACCESS_COARSE_LOCATION` + `ACCESS_FINE_LOCATION` on Android. It is a no-op if `requestLocationPermission` is not explicitly set.

### Basic usage

```tsx
import { AppleMaps, GoogleMaps } from 'expo-maps';
import { Platform } from 'react-native';

export default function App() {
  if (Platform.OS === 'ios') {
    return <AppleMaps.View style={{ flex: 1 }} />;
  } else if (Platform.OS === 'android') {
    return <GoogleMaps.View style={{ flex: 1 }} />;
  }
}
```

### Components

#### AppleMapsView (iOS only) — `AppleMaps.View`

Props:
- `annotations`: `AppleMapsAnnotation[]`
- `cameraPosition`: `CameraPosition`
- `circles`: `AppleMapsCircle[]`
- `colorScheme`: `AppleMapsColorScheme` (default `AUTOMATIC`)
- `markers`: `AppleMapsMarker[]`
- `polygons`: `AppleMapsPolygon[]`
- `polylines`: `AppleMapsPolyline[]`
- `properties`: `AppleMapsProperties`
- `style`: `ViewStyle`
- `uiSettings`: `AppleMapsUISettings`
- `ref`: `Ref<AppleMapsViewType>`
- Event handlers: `onAnnotationClick`, `onCameraMove`, `onCircleClick`, `onMapClick`, `onMarkerClick`, `onPolygonClick`, `onPolylineClick`

#### GoogleMapsView (Android only) — `GoogleMaps.View`

Props:
- `cameraPosition`: `CameraPosition`
- `circles`: `GoogleMapsCircle[]`
- `colorScheme`: `GoogleMapsColorScheme`
- `contentPadding`: `GoogleMapsContentPadding`
- `mapOptions`: `GoogleMapsMapOptions`
- `markers`: `GoogleMapsMarker[]`
- `polygons`: `GoogleMapsPolygon[]`
- `polylines`: `GoogleMapsPolyline[]`
- `properties`: `GoogleMapsProperties`
- `style`: `ViewStyle`
- `uiSettings`: `GoogleMapsUISettings`
- `userLocation`: `GoogleMapsUserLocation`
- `ref`: `Ref<GoogleMapsViewType>`
- Event handlers: `onCameraMove`, `onCircleClick`, `onMapClick`, `onMapLongClick`, `onMapLoaded`, `onMarkerClick`, `onPOIClick`, `onPolygonClick`, `onPolylineClick`

#### GoogleStreetView (Android only)

Props:
- `position`: `StreetViewCameraPosition`
- `style`: `ViewStyle`
- `isPanningGesturesEnabled`: `boolean`
- `isStreetNamesEnabled`: `boolean`
- `isUserNavigationEnabled`: `boolean`
- `isZoomGesturesEnabled`: `boolean`

### Markers & annotations

**AppleMapsMarker:**
- `coordinates`: `Coordinates`
- `id`: `string`
- `systemImage`: `string` (SF Symbol)
- `monogram`: `string` (iOS 17.0+, 1–2 characters)
- `tintColor`: `string`
- `title`: `string`

**AppleMapsAnnotation** (extends AppleMapsMarker):
- `backgroundColor`: `string`
- `icon`: `SharedRefType<'image'>`
- `text`: `string`
- `textColor`: `string`

**GoogleMapsMarker:**
- `coordinates`: `Coordinates`
- `id`: `string`
- `anchor`: `GoogleMapsAnchor`
- `draggable`: `boolean`
- `icon`: `SharedRefType<'image'>`
- `showCallout`: `boolean`
- `snippet`: `string`
- `title`: `string`
- `zIndex`: `number` (default `0`)

### Camera & coordinates

**CameraPosition:**
- `coordinates`: `Coordinates` (optional)
- `zoom`: `number` (optional)

**Coordinates:** both fields are **optional** — `{ latitude?: number; longitude?: number }` (`packages/expo-maps/src/shared.types.ts`).

**CameraMoveEvent:** `bearing`, `coordinates`, `latitudeDelta`, `longitudeDelta`, `tilt`, `zoom`.

### Shapes

**Circles** — shared: `center` and `radius` (required), plus optional `id`, `color`, `lineColor`, `lineWidth`. The two platforms are **not** identical:
- `AppleMapsCircle` additionally has `width?: number` (separate from `lineWidth`).
- `GoogleMapsCircle` additionally has `clickCoordinates?: Coordinates`.
- Colors are typed `ProcessedColorValue | string`, not plain `string`.

**Polygons** (both platforms): `coordinates`, `color`, `lineColor`, `lineWidth`, `id`.

**Polylines:**
- AppleMaps: `coordinates`, `color`, `width`, `contourStyle`, `id`
- GoogleMaps: `coordinates`, `color`, `width`, `geodesic`, `id`

### Hooks & methods

`AppleMaps` / `GoogleMaps` are namespaces and the permission helpers are top-level named exports — all from the same entry point (`packages/expo-maps/src/index.ts`):

```ts
import {
  AppleMaps,
  GoogleMaps,
  useLocationPermissions,
  getPermissionsAsync,
  requestPermissionsAsync,
} from 'expo-maps';

const [status, requestPermission] = useLocationPermissions();
```

- `getPermissionsAsync(): Promise<PermissionResponse>`
- `requestPermissionsAsync(): Promise<PermissionResponse>`

(`Maps.getPermissionsAsync(...)` only works if you wrote `import * as Maps from 'expo-maps'`.)

### Ref methods

**AppleMapsViewType:**
- `setCameraPosition(config?: CameraPosition): void` — animation duration is not supported on iOS
- `selectMarker(id?, options?): void` (iOS 18.0+)
- `selectAnnotation(id?, options?): void` (iOS 18.0+)
- `openLookAroundAsync(coordinates): Promise<void>`

**GoogleMapsViewType:**
- `setCameraPosition(config?: SetCameraPositionConfig): void`
- `selectMarker(id?, options?): Promise<void>` — rejects if a rapid follow-up call cancels the in-flight animation

`options` on `selectMarker` / `selectAnnotation` is `{ zoom?: number; moveCamera?: boolean }` (`moveCamera` defaults to `true`); passing `undefined` for `id` clears the selection.

`SetCameraPositionConfig = CameraPosition & { duration?: number }` (milliseconds, Android only — optional).

### SDK 56 notes

- Alpha — expect breaking changes; requires a development build.
- iOS 18.0+ required for annotation/marker selection and polygon/polyline click events.
- iOS 17.0+ required for marker `monogram`.

---

## Web support

### Overview

Source: https://docs.expo.dev/workflow/web/

Add web support dependencies:

```sh
npx expo install react-dom react-native-web @expo/metro-runtime
```

For an existing React Native app without Expo, install the `expo` package and switch the entry file to `registerRootComponent` instead of `AppRegistry.registerComponent`.

Run / export:

```sh
npx expo start --web              # development
npx expo export --platform web    # production export
```

Notes:
- React Native for web supplies cross-platform components (`<Text>`, `<View>`, etc.) wrapping React DOM primitives.
- Fast Refresh, debugging, env vars and bundling work universally across platforms.
- Metro is the JS bundler; Expo CLI optimizes with platform shaking.

### Web output modes

Expo Router supports three web output modes set via `expo.web.output` in `app.json`:

| Mode | Behavior |
|------|----------|
| `single` | Exports a single-page app with one `index.html`. Hosting must rewrite all URLs to `/index.html`. |
| `static` | Statically generated web app — HTML/CSS pre-rendered at build time. Deploy to any static host; do NOT add SPA-style `/index.html` redirects. |
| `server` | Server functions and API routes plus static pages; HTML generated dynamically per request. Requires a runtime server. |

Note: Expo Router does not support mixing server and static rendering in the same project — choose one output mode.

Sources: https://docs.expo.dev/router/web/static-rendering/ , https://docs.expo.dev/router/web/server-rendering/

#### Static rendering

Source: https://docs.expo.dev/router/web/static-rendering/

```json
{
  "expo": {
    "web": {
      "output": "static"
    }
  }
}
```

- Generates HTML and CSS at build time (good for SEO). Data loaders run during the build and their results are embedded in the output HTML.
- `npx expo export --platform web` produces a `dist` directory; files in a local `public` directory are copied over automatically.
- Dynamic routes require `generateStaticParams` to pre-generate known routes:

```tsx
export async function generateStaticParams(): Promise<Record<string, string>[]> {
  const posts = await getPosts();
  return posts.map(post => ({ id: post.id }));
}
```

This runs in Node.js at build-time (has `process.cwd()` and env vars) but cannot use browser APIs or native Expo modules.

- Metadata via the `<Head />` component:

```tsx
import Head from 'expo-router/head';

export default function Page() {
  return (
    <>
      <Head>
        <title>My Blog Website</title>
        <meta name="description" content="This is my blog." />
      </Head>
    </>
  );
}
```

- Customize the global HTML wrapper via `src/app/+html.tsx`.

#### Server rendering

Source: https://docs.expo.dev/router/web/server-rendering/

Requires SDK 55+; currently alpha.

```json
{
  "expo": {
    "web": {
      "output": "server"
    },
    "plugins": [
      [
        "expo-router",
        {
          "unstable_useServerRendering": true
        }
      ]
    ]
  }
}
```

- Generates HTML dynamically on each request. Dynamic routes render automatically at request time (no `generateStaticParams` needed). Data loaders run on the server per request and the result is embedded in the HTML response.
- `npx expo export --platform web` produces a `dist` directory with a `client` folder (JS/CSS bundles for hydration) and a `server` folder (routes manifest + `render.js`). Test locally with `npx expo serve`.
- Routes can export `generateMetadata`, which runs on the server before rendering and receives the request and route params.
- Requires a runtime server (cannot deploy to static hosts). Supported: EAS Hosting, Node.js/Express, Cloudflare Workers, Vercel, Netlify, Bun (via adapters).

| Aspect | Server | Static |
|--------|--------|--------|
| Generation | Per request | Build time |
| Dynamic routes | Automatic | Requires `generateStaticParams` |
| Hosting | Server required | Any static host |
| Time to First Byte | Slower | Fastest |

---

## Utility APIs

**SDK 56 baseline for every package in this section:** minimum iOS/tvOS 16.4, macOS 13.4 ([#43296](https://github.com/expo/expo/pull/43296)), listed as the only 🛠 breaking change in each package's `## 56.0.0 — 2026-05-05` changelog entry. Beyond that, only `expo-clipboard` broke a JS API (removed `setString`). Individual sections below call out anything extra.

Pick the right module:

| Goal | API |
|------|-----|
| Send a file/URL out to another app | `expo-sharing` → `shareAsync(url, options)` |
| Receive content shared *into* your app | `expo-sharing` → `useIncomingShare()` + its config plugin (experimental) |
| Copy/paste text, images, URLs | `expo-clipboard` |
| Generate a PDF, or print | `expo-print` → `printToFileAsync` / `printAsync` |
| Compose an email with attachments | `expo-mail-composer` → `composeAsync` |
| Prefill an SMS | `expo-sms` → `sendSMSAsync` |
| In-app store rating prompt | `expo-store-review` → `hasAction()` then `requestReview()` |
| iOS ATT prompt / IDFA | `expo-tracking-transparency` |

### expo-clipboard

Source: https://docs.expo.dev/versions/v56.0.0/sdk/clipboard/

Key methods:
- `getStringAsync(options?): Promise<string>`
- `setStringAsync(text, options?): Promise<boolean>` (returns `boolean` on web; always `true` on iOS/Android)
- `hasStringAsync(): Promise<boolean>`
- `getUrlAsync(): Promise<string | null>` / `setUrlAsync(url: string): Promise<void>` / `hasUrlAsync(): Promise<boolean>` — iOS-oriented (`ContentType.URL` is `@platform iOS`)
- `getImageAsync(options): Promise<ClipboardImage | null>`
- `setImageAsync(base64Image: string): Promise<void>` — one parameter only in SDK 56 and 57; the `options?: SetImageOptions` overload is unversioned/main-branch only
- `hasImageAsync(): Promise<boolean>`
- `addClipboardListener(listener): EventSubscription` — call `subscription.remove()` to detach; `removeClipboardListener(subscription)` still exists but is `@deprecated`
- `isPasteButtonAvailable: boolean` (a const, not a function — iOS 16+; `false` elsewhere) and the `<ClipboardPasteButton />` component, both exported from `expo-clipboard`

`ContentType` enum: `PLAIN_TEXT = 'plain-text'`, `HTML = 'html'`, `IMAGE = 'image'`, `URL = 'url'` (iOS).

```tsx
import * as Clipboard from 'expo-clipboard';

await Clipboard.setStringAsync('hello world');
const text = await Clipboard.getStringAsync();

const sub = Clipboard.addClipboardListener(({ contentTypes }) => {
  if (contentTypes.includes(Clipboard.ContentType.PLAIN_TEXT)) {
    Clipboard.getStringAsync().then(content => alert(content));
  }
});
// later: sub.remove();
```

**SDK 56 breaking:** the deprecated synchronous `Clipboard.setString()` was **removed** ([#41758](https://github.com/expo/expo/pull/41758)) — use `setStringAsync()`. (React Native core's `Clipboard` is also long gone; do not reach for either.)

Notes: Web uses the Async Clipboard API (browser-dependent). iOS 16+ requires user paste permission; denied access returns null/empty indistinguishably.

### expo-sharing

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sharing/

Key methods:
- `isAvailableAsync(): Promise<boolean>`
- `shareAsync(url: string, options?: SharingOptions): Promise<void>`

`SharingOptions`: `dialogTitle` (Android/Web), `mimeType` (Android), `UTI` (iOS), `anchor` (iOS iPad positioning).

```js
import * as Sharing from 'expo-sharing';

const available = await Sharing.isAvailableAsync();
if (available) {
  await Sharing.shareAsync('file:///path/to/file.pdf', {
    dialogTitle: 'Share Document',
    mimeType: 'application/pdf',
  });
}
```

#### Receiving shares (experimental)

`expo-sharing` also receives content shared *into* your app. Marked experimental; on iOS the share extension opens the main target rather than processing the share in a sharing `ViewController`, which Apple does not officially support and may break in a future iOS release.

Exports (`packages/expo-sharing/src/index.ts`):
- `getSharedPayloads(): SharePayload[]` — synchronous, available immediately
- `getResolvedSharedPayloadsAsync(): Promise<ResolvedSharePayload[]>`
- `clearSharedPayloads(): void`
- `useIncomingShare(): { sharedPayloads, resolvedSharedPayloads, clearSharedPayloads, isResolving, error, refreshSharePayloads }` — re-reads payloads on `AppState` change to `active`

`SharePayload`: `{ value: string; shareType: ShareType; mimeType?: string }` where `ShareType = 'text' | 'url' | 'audio' | 'image' | 'video' | 'file'`. A `ResolvedSharePayload` adds `contentUri`, `contentType` (`'text' | 'audio' | 'image' | 'video' | 'file' | 'website'`), `contentMimeType`, `originalName`, `contentSize`.

Config plugin props (all disabled by default):

| Prop | Default | Notes |
|------|---------|-------|
| `ios.enabled` | `false` | Adds a Share Extension target |
| `ios.extensionBundleIdentifier` | `{appBundleIdentifier}.ShareExtension` | |
| `ios.appGroupId` | `group.{appBundleIdentifier}` | Shares data app ↔ extension |
| `ios.activationRule` | `{}` | `ActivationRuleOptions` object or a raw `NSPredicate` string |
| `android.enabled` | `false` | Adds the `intent-filter` |
| `android.singleShareMimeTypes` | `[]` | `ACTION_SEND` |
| `android.multipleShareMimeTypes` | `[]` | `ACTION_SEND_MULTIPLE` |

`ActivationRuleOptions`: `supportsText?: boolean`, plus max-count numbers `supportsWebUrlWithMaxCount`, `supportsImageWithMaxCount`, `supportsMovieWithMaxCount`, `supportsFileWithMaxCount`, `supportsWebPageWithMaxCount`, `supportsAttachmentsWithMaxCount` (all default `0` = not accepted).

Routing: the OS deep-links into the app with hostname `expo-sharing`. With Expo Router, intercept it in `app/+native-intent.ts`:

```tsx
export async function redirectSystemPath({ path, initial }: { path: string; initial: boolean }) {
  try {
    if (new URL(path).hostname === 'expo-sharing') return '/handle-share';
    return path;
  } catch {
    return '/';
  }
}
```

Notes: Web requires HTTPS and has limited Web Share API support.

### expo-print

Source: https://docs.expo.dev/versions/v56.0.0/sdk/print/

Key methods:
- `printAsync(options: PrintOptions): Promise<void>` — prints a document/HTML (on web prints the page HTML)
- `printToFileAsync(options?: FilePrintOptions): Promise<FilePrintResult>` — prints HTML to a PDF in the app cache dir
- `selectPrinterAsync(): Promise<Printer>` — iOS only

`PrintOptions`: `uri?` (PDF only — remote, local, or a `data:application/pdf;base64,` URI), `html?`, `width?` (default `612`), `height?` (default `792`), `printerUrl?` (iOS), `useMarkupFormatter?` (iOS), `orientation?` (iOS — `Print.Orientation.portrait | .landscape`), `margins?: PageMargins` (iOS — `{ top, right, bottom, left }`).

`FilePrintOptions`: `html?`, `useMarkupFormatter?` (iOS), `width?`, `height?`, `margins?` (iOS), `base64?: boolean`, `textZoom?: number` (Android, percent, default `100`).

`FilePrintResult`: `{ uri: string; numberOfPages: number; base64?: string }` — `base64` only when `base64: true` was requested, and without the `data:application/pdf;base64,` prefix.

```jsx
import * as Print from 'expo-print';
import { shareAsync } from 'expo-sharing';

await Print.printAsync({ html, printerUrl: selectedPrinter?.url /* iOS only */ });

const { uri, numberOfPages } = await Print.printToFileAsync({ html });
await shareAsync(uri, { UTI: '.pdf', mimeType: 'application/pdf' });
```

Notes: `markupFormatterIOS` is deprecated (replaced by `useMarkupFormatter`).

### expo-mail-composer

Source: https://docs.expo.dev/versions/v56.0.0/sdk/mail-composer/

Key methods:
- `composeAsync(options: MailComposerOptions): Promise<MailComposerResult>` — opens the system mail UI
- `isAvailableAsync(): Promise<boolean>`
- `getClients(): MailClient[]` — `{ label: string; packageName?: string /* Android */; url?: string /* iOS */ }`

`MailComposerOptions`: `recipients`, `ccRecipients`, `bccRecipients` (string arrays), `subject`, `body`, `attachments` (file URIs), `isHtml` (boolean).

`MailComposerResult` is `{ status: MailComposerStatus }`. `MailComposerStatus` is a TypeScript **enum**, not a string-literal union — `UNDETERMINED = 'undetermined'`, `SENT = 'sent'`, `SAVED = 'saved'`, `CANCELLED = 'cancelled'` — so compare with `result.status === MailComposer.MailComposerStatus.SENT`. Note `SAVED` — the user saved a draft; it is neither sent nor cancelled.

```js
import * as MailComposer from 'expo-mail-composer';

if (await MailComposer.isAvailableAsync()) {
  const result = await MailComposer.composeAsync({
    recipients: ['user@example.com'],
    subject: 'Hello',
    body: 'This is a test email',
  });
}
```

On iOS, `getClients()` only sees clients whose URL schemes are declared in `LSApplicationQueriesSchemes`. The `expo-mail-composer` config plugin (no props) writes that list for you — add `"expo-mail-composer"` to `plugins` if you call `getClients()`.

Notes: Android, iOS (device only), Web, Expo Go.

### expo-sms

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sms/

Key methods:
- `isAvailableAsync(): Promise<boolean>` — always `false` on iOS simulator and browser
- `sendSMSAsync(addresses, message, options?: SMSOptions): Promise<SMSResponse>` — result is `'sent'`, `'cancelled'`, or `'unknown'`

```ts
import * as SMS from 'expo-sms';

const { result } = await SMS.sendSMSAsync(
  ['0123456789', '9876543210'],
  'My sample HelloWorld message',
  {
    attachments: {
      uri: 'path/myfile.png',
      mimeType: 'image/png',
      filename: 'myfile.png',
    },
  }
);
```

Notes: Android, iOS, Expo Go.

### expo-store-review

Source: https://docs.expo.dev/versions/v56.0.0/sdk/storereview/

Key methods:
- `requestReview(): Promise<void>` — native rating modal without leaving the app
- `isAvailableAsync(): Promise<boolean>` — iOS true unless via TestFlight; Android requires 5.0+
- `hasAction(): Promise<boolean>` — literally `!!storeUrl() || (await isAvailableAsync())`
- `storeUrl(): string | null` — reads `Constants.expoConfig.ios.appStoreUrl` on iOS and `Constants.expoConfig.android.playStoreUrl` on Android; `null` on web

```ts
import * as StoreReview from 'expo-store-review';

if (await StoreReview.hasAction()) {
  await StoreReview.requestReview();
}
```

If the native flow is unavailable **and** those `app.json` fields are missing, `requestReview()` silently does nothing but log `"StoreReview.requestReview(): Couldn't link to store, please make sure the android.playStoreUrl & ios.appStoreUrl fields are filled out in your app.json"`. That is why `hasAction()` is the correct guard.

Notes: Call `requestReview()` after meaningful user interactions, not from a button, and avoid time-sensitive moments.

### expo-tracking-transparency

Source: https://docs.expo.dev/versions/v56.0.0/sdk/tracking-transparency/

Key methods / hook:
- `requestTrackingPermissionsAsync(): Promise<PermissionResponse>`
- `getTrackingPermissionsAsync(): Promise<PermissionResponse>`
- `useTrackingPermissions()` hook → `[status, requestPermission]`
- `getAdvertisingId(): string | null` — IDFA on iOS / advertising ID on Android; `null` when permission is not granted
- `isAvailable(): boolean`

```ts
import { requestTrackingPermissionsAsync } from 'expo-tracking-transparency';

const { granted } = await requestTrackingPermissionsAsync();
if (granted) {
  // App is authorized to track the user or their device
}
```

Config plugin (set permission message):

```json
{
  "expo": {
    "plugins": [
      ["expo-tracking-transparency", { "userTrackingPermission": "Custom permission message" }]
    ]
  }
}
```

Notes: On iOS you must add `NSUserTrackingUsageDescription` to Info.plist or the app will be rejected by Apple. Android, iOS, tvOS, Expo Go.

### expo-blur

Source: https://docs.expo.dev/versions/v56.0.0/sdk/blur-view/

`BlurView` props:
- `intensity`: `number` (1–100, default `50`; animatable with reanimated)
- `tint`: `BlurTint` (default `'default'`) — `'light'`, `'dark'`, `'default'`, `'extraLight'`, `'regular'`, `'prominent'`, plus system material variants
- `blurMethod` (Android only): `BlurMethod` (default `'none'`) — `'none'`, `'dimezisBlurView'`, `'dimezisBlurViewSdk31Plus'`
- `blurReductionFactor` (Android only): `number` (default `4`) — fine-tune to match iOS
- `blurTarget` (Android only): `RefObject<View | null>` — connects to a `BlurTargetView` for background blurring
- `experimentalBlurMethod` (Android only): `@deprecated` alias of `blurMethod`; still accepted, but use `blurMethod`

```jsx
import { BlurView } from 'expo-blur';

<BlurView intensity={100} tint="dark" style={styles.blurContainer}>
  <Text style={styles.text}>Blurred content</Text>
</BlurView>
```

**On Android a bare `<BlurView>` blurs nothing.** You need `blurMethod` *and* a `BlurTargetView` wrapping the content to be blurred, with its ref handed to the `BlurView`:

```tsx
import { BlurView, BlurTargetView } from 'expo-blur';
import { useRef } from 'react';
import { View } from 'react-native';

const targetRef = useRef<View | null>(null);

<View style={styles.container}>
  <BlurTargetView ref={targetRef} style={styles.background}>
    {content}
  </BlurTargetView>
  <BlurView
    blurTarget={targetRef}
    intensity={100}
    blurMethod="dimezisBlurView"
    style={styles.blurContainer}
  />
</View>
```

A single `BlurTargetView` can back multiple `BlurView`s as long as they all fit inside its bounds — that is more efficient than creating several targets.

If touchables underneath a `BlurTargetView` are unresponsive on Android, that is a known bug fixed in **expo-blur 56.0.4** ([#47404](https://github.com/expo/expo/pull/47404)) — gesture handlers were treating those views as detached and cancelling their gestures. Bump the package (`npx expo install expo-blur`); no SDK upgrade needed.

Version attribution: the `BlurTargetView` requirement, the new Android blur API, `dimezisBlurViewSdk31Plus`, and the `experimentalBlurMethod` → `blurMethod` rename all shipped in **expo-blur 55.0.0** (2026-01-21), not 56. `expo-blur` 56.0.0's only breaking change is the min iOS/tvOS 16.4 / macOS 13.4 bump.

### expo-linear-gradient

Source: https://docs.expo.dev/versions/v56.0.0/sdk/linear-gradient/

`LinearGradient` props:

| Prop | Type | Platform | Notes |
|------|------|----------|-------|
| `colors` | `readonly [ColorValue, ColorValue, ...ColorValue[]]` | All | Required; min 2 colors |
| `start` | `LinearGradientPoint \| null` | All | Origin (0–1 range) |
| `end` | `LinearGradientPoint \| null` | All | Terminus (0–1 range) |
| `locations` | `readonly [number, number, ...number[]] \| null` | All | Color-stop positions, ascending |
| `dither` | `boolean` | Android only | Default `true`; disable to reduce banding/boost perf |

`colors` is a **tuple** type, not `ColorValue[]`. A variable typed `string[]` is a TypeScript error — pass the array inline or annotate it `as const`.

```tsx
import { LinearGradient } from 'expo-linear-gradient';

<LinearGradient
  colors={['#4c669f', '#3b5998', '#192f6a']}
  start={{ x: 0, y: 0 }}
  end={{ x: 1, y: 1 }}
  locations={[0, 0.5, 1]}
  style={styles.button}
>
  <Text>Gradient Button</Text>
</LinearGradient>
```

Notes: Android, iOS, tvOS, Web, Expo Go (docs frontmatter platform list, unchanged in 56 and 57). macOS (react-native-macos) support was added in expo-linear-gradient 56.0.4.

**Not deprecated.** React Native's `experimental_backgroundImage` style prop (Android/iOS) and `backgroundImage` (Web) accept CSS `linear-gradient()` / `radial-gradient()` syntax and are presented in the docs as an *optional alternative* that avoids the extra dependency — they do not replace `expo-linear-gradient`. Source: `docs/pages/versions/v56.0.0/sdk/linear-gradient.mdx:16`.

---

## SDK 57 delta

**Nothing in this file changed in SDK 57 — every API signature above is still correct, and there is no work to do here as part of a 56 → 57 upgrade.** On the `sdk-57` release branch, every `## 57.0.x` changelog entry for `expo-maps`, `expo-clipboard`, `expo-sharing`, `expo-print`, `expo-mail-composer`, `expo-sms`, `expo-store-review`, `expo-tracking-transparency`, `expo-blur` and `expo-linear-gradient` reads *"This version does not introduce any user-facing changes"* — with the single exception of expo-blur 57.0.1, which carries a fix that was also backported to SDK 56 (below). Web output modes are unchanged. There are **no new APIs and no breaking changes** for this domain.

### SDK 56 patch drift (not a reason to upgrade)

These look like 57 features but are already available on the SDK 56 line — `npx expo install` picks them up without changing SDK.

- **expo-blur: Android gesture fix** ([#47404](https://github.com/expo/expo/pull/47404)). Gesture handlers no longer treat views underneath a `BlurTargetView` as detached and cancel their gestures — the bug that made buttons under a `BlurTargetView` unresponsive on Android. Shipped in **expo-blur 56.0.4** (2026-07-15) *and* 57.0.1 on the same day. **Needs >= 56.0.4** on SDK 56; the SDK 56 pin is already `~56.0.4`. Do not upgrade SDK for this.
- **expo-clipboard: macOS support** ([#46479](https://github.com/expo/expo/pull/46479)). Shipped in **expo-clipboard 56.0.4** (2026-06-05); `origin/sdk-56`'s `src/Clipboard.ts` already carries all three `@platform macos` tags and already documents `addClipboardListener` as *"a no-op on Web and macOS."* **Needs >= 56.0.4**, which is the SDK 56 pin, so every SDK 56 project has it. The only 56 → 57 difference is a docs-freeze artifact: `docs/public/static/data/v56.0.0/expo-clipboard.json` was generated before 56.0.4 shipped, so the macOS tags appear only in the v57 data. (macOS here means react-native-macos — the docs frontmatter platform list is still `['android','ios','web','expo-go']` in both.)

### Not in 57

`expo-clipboard`'s `android.isSensitive` option on `setStringAsync` / `setImageAsync` ([#43291](https://github.com/expo/expo/pull/43291)) **never landed on the `sdk-57` release branch** — it is still under `## Unpublished` on `main` (SDK 58 in progress) and `origin/sdk-57:packages/expo-clipboard/src/Clipboard.types.ts` contains zero occurrences of `isSensitive`. The published tarball confirms it: `expo-clipboard@57.0.1`'s `build/Clipboard.d.ts` declares `setImageAsync(base64Image: string): Promise<void>` — one parameter, no `SetImageOptions`. Do not emit it for SDK 56 or 57.

### Version pins (56 → 57)

From `packages/expo/bundledNativeModules.json` on the release branches (`git show origin/sdk-56:…` / `origin/sdk-57:…`) — this is what `expo install` resolves against. The SDK 56 column is the current patch line, which keeps moving.

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-maps` | `~56.0.7` | `~57.0.1` |
| `expo-clipboard` | `~56.0.4` | `~57.0.1` |
| `expo-sharing` | `~56.0.23` | `~57.0.7` |
| `expo-print` | `~56.0.4` | `~57.0.1` |
| `expo-mail-composer` | `~56.0.4` | `~57.0.1` |
| `expo-sms` | `~56.0.3` | `~57.0.1` |
| `expo-store-review` | `~56.0.3` | `~57.0.1` |
| `expo-tracking-transparency` | `~56.0.5` | `~57.0.1` |
| `expo-blur` | `~56.0.4` | `~57.0.2` |
| `expo-linear-gradient` | `~56.0.4` | `~57.0.1` |
| `@expo/metro-runtime` | `~56.0.18` | `~57.0.7` |
| `react-native-web` | `~0.21.0` | `~0.21.0` (unchanged) |
| `react-dom` | `19.2.3` | `19.2.3` (unchanged) |

The SDK 57 pins are **not** a flat `~57.0.0` — `expo-sharing` and `@expo/metro-runtime` are already at `~57.0.7`, `expo-blur` at `~57.0.2`. Because `react-dom` and `react-native-web` did not move, `npx expo install react-dom react-native-web @expo/metro-runtime` needs no version-specific handling.

> Two sources to avoid. (1) `packages/<pkg>/package.json` on the expo `main` branch — `main` is SDK 58 in progress; every one of these still reads `56.0.x` there and their 57 content sits under `## Unpublished`. (2) `docs/public/static/schemas/v5{6,7}.0.0/native-modules.json` — frozen at the docs cut and already stale (it lists `expo-blur` at `~56.0.3`, shipped is `~56.0.4`). Use `bundledNativeModules.json` on the release branch.

### Unchanged, verified

- **expo-maps** is still `isAlpha: true` and still *"not available in the Expo Go app"* in `docs/pages/versions/v57.0.0/sdk/maps.mdx`. The v56 and v57 `.mdx` pages differ only in `sourceCodeUrl` (`sdk-56` → `sdk-57`), an added `exampleName: 'with-maps'`, and one versioned link. The "alpha, development build required" framing above applies to both SDKs. `setCameraPosition` still returns `void` on both branches (`origin/sdk-57:packages/expo-maps/src/google/GoogleMapsView.tsx`) — there is no promise to await or catch.
- **Web output modes**: `expo.web.output` is still exactly `["single", "static", "server"]` with an identical description in both `app-config-schema.json` files. Server rendering is still opt-in via the expo-router plugin prop `unstable_useServerRendering` (`packages/expo-router/plugin/src/withRouter.ts`) and still alpha / SDK 55+ (`docs/pages/router/web/server-rendering.mdx`).
- **Zero API changes** in `expo-maps`, `expo-clipboard`, `expo-sharing`, `expo-print`, `expo-mail-composer`, `expo-sms`, `expo-store-review`, `expo-tracking-transparency` and `expo-linear-gradient` across the whole `## 57.0.x` range on `origin/sdk-57`. Do not go hunting for churn here.
