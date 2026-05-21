# Expo SDK 56 — Maps, Web Support & Utility APIs

Knowledge base reference covering the Expo-native maps module, web support / output modes, and utility APIs. All package docs target the SDK 56 path `https://docs.expo.dev/versions/v56.0.0/sdk/<pkg>/`.

---

## expo-maps

Source: https://docs.expo.dev/versions/v56.0.0/sdk/maps/

The newer Expo-native maps module. **Alpha status** — breaking changes are expected and it requires a development build (not available in Expo Go). Apple Maps is iOS-only; Google Maps is Android-only.

### Installation

```sh
npx expo install expo-maps
```

Google Maps on Android requires a Google Cloud Maps API key configured via the config plugin. Location permissions are configured via the config plugin with `requestLocationPermission` and `locationPermission` props.

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

**Coordinates:**
- `latitude`: `number`
- `longitude`: `number`

**CameraMoveEvent:** `bearing`, `coordinates`, `latitudeDelta`, `longitudeDelta`, `tilt`, `zoom`.

### Shapes

**Circles** (both platforms): `center`, `radius`, `color`, `lineColor`, `lineWidth`, `id`.

**Polygons** (both platforms): `coordinates`, `color`, `lineColor`, `lineWidth`, `id`.

**Polylines:**
- AppleMaps: `coordinates`, `color`, `width`, `contourStyle`, `id`
- GoogleMaps: `coordinates`, `color`, `width`, `geodesic`, `id`

### Hooks & methods

```ts
const [status, requestPermission] = useLocationPermissions(options?);
```

- `Maps.getPermissionsAsync(): Promise<PermissionResponse>`
- `Maps.requestPermissionsAsync(): Promise<PermissionResponse>`

### Ref methods

**AppleMapsViewType:**
- `setCameraPosition(config: CameraPosition): void`
- `selectMarker(id, options): void` (iOS 18.0+)
- `selectAnnotation(id, options): void` (iOS 18.0+)
- `openLookAroundAsync(coordinates): Promise<void>`

**GoogleMapsViewType:**
- `setCameraPosition(config: SetCameraPositionConfig): void`
- `selectMarker(id, options): Promise<void>`

`SetCameraPositionConfig` extends `CameraPosition` with `duration: number` (milliseconds, Android only).

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

### expo-clipboard

Source: https://docs.expo.dev/versions/v56.0.0/sdk/clipboard/

Key methods:
- `getStringAsync(options?): Promise<string>`
- `setStringAsync(text, options?): Promise<boolean>` (returns `boolean` on web; always `true` on iOS/Android)
- `hasStringAsync(): Promise<boolean>`
- `getImageAsync(options): Promise<ClipboardImage | null>`
- `setImageAsync(base64Image): Promise<void>`
- `hasImageAsync(): Promise<boolean>`
- `addClipboardListener(listener): EventSubscription`

```tsx
await Clipboard.setStringAsync('hello world');
const text = await Clipboard.getStringAsync();

Clipboard.addClipboardListener(({ contentTypes }) => {
  if (contentTypes.includes(Clipboard.ContentType.PLAIN_TEXT)) {
    Clipboard.getStringAsync().then(content => alert(content));
  }
});
```

Notes: Web uses the Async Clipboard API (browser-dependent). iOS 16+ requires user paste permission; denied access returns null/empty indistinguishably. No SDK 56 breaking changes.

### expo-sharing

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sharing/

Key methods:
- `isAvailableAsync(): Promise<boolean>`
- `shareAsync(url: string, options?: SharingOptions): Promise<void>`

`SharingOptions`: `dialogTitle` (Android/Web), `mimeType` (Android), `UTI` (iOS), `anchor` (iOS iPad positioning).

```js
const available = await Sharing.isAvailableAsync();
if (available) {
  await Sharing.shareAsync('file:///path/to/file.pdf', {
    dialogTitle: 'Share Document',
    mimeType: 'application/pdf',
  });
}
```

Notes: Web requires HTTPS and has limited Web Share API support. No SDK 56 breaking changes.

### expo-print

Source: https://docs.expo.dev/versions/v56.0.0/sdk/print/

Key methods:
- `printAsync(options: PrintOptions): Promise<void>` — prints a document/HTML (on web prints the page HTML)
- `printToFileAsync(options?: FilePrintOptions): Promise<FilePrintResult>` — prints HTML to a PDF in the app cache dir
- `selectPrinterAsync(): Promise<Printer>` — iOS only

```jsx
await Print.printAsync({ html, printerUrl: selectedPrinter?.url /* iOS only */ });

const { uri } = await Print.printToFileAsync({ html });
await shareAsync(uri, { UTI: '.pdf', mimeType: 'application/pdf' });
```

Notes: `markupFormatterIOS` is deprecated (replaced by `useMarkupFormatter`). No SDK 56 breaking changes.

### expo-mail-composer

Source: https://docs.expo.dev/versions/v56.0.0/sdk/mail-composer/

Key methods:
- `composeAsync(options: MailComposerOptions)` — opens the system mail UI
- `isAvailableAsync(): Promise<boolean>`

`MailComposerOptions`: `recipients`, `ccRecipients`, `bccRecipients` (string arrays), `subject`, `body`, `attachments` (file URIs), `isHtml` (boolean).

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

Notes: Android, iOS (device only), Web, Expo Go. No SDK 56 breaking changes.

### expo-sms

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sms/

Key methods:
- `isAvailableAsync(): Promise<boolean>` — always `false` on iOS simulator and browser
- `sendSMSAsync(addresses, message, options?: SMSOptions): Promise<SMSResponse>` — result is `'sent'`, `'cancelled'`, or `'unknown'`

```ts
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

Notes: Android, iOS, Expo Go. No SDK 56 breaking changes.

### expo-store-review

Source: https://docs.expo.dev/versions/v56.0.0/sdk/storereview/

Key methods:
- `requestReview(): Promise<void>` — native rating modal without leaving the app
- `isAvailableAsync(): Promise<boolean>` — iOS true unless via TestFlight; Android requires 5.0+
- `hasAction(): Promise<boolean>` — whether the review flow is available
- `storeUrl(): string | null` — store URL from app config

```ts
if (await StoreReview.hasAction()) {
  await StoreReview.requestReview();
}
```

Notes: Call `requestReview()` after meaningful user interactions, not from a button, and avoid time-sensitive moments. No SDK 56 breaking changes.

### expo-tracking-transparency

Source: https://docs.expo.dev/versions/v56.0.0/sdk/tracking-transparency/

Key methods / hook:
- `requestTrackingPermissionsAsync()`
- `getTrackingPermissionsAsync()`
- `useTrackingPermissions()` hook → `[status, requestPermission]`

```ts
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

Notes: On iOS you must add `NSUserTrackingUsageDescription` to Info.plist or the app will be rejected by Apple. Android, iOS, tvOS, Expo Go. No SDK 56 breaking changes.

### expo-blur

Source: https://docs.expo.dev/versions/v56.0.0/sdk/blur-view/

`BlurView` props:
- `intensity`: `number` (1–100, default `50`; animatable with reanimated)
- `tint`: `BlurTint` (default `'default'`) — `'light'`, `'dark'`, `'default'`, `'extraLight'`, `'regular'`, `'prominent'`, plus system material variants
- `blurMethod` (Android only): `BlurMethod` (default `'none'`) — `'none'`, `'dimezisBlurView'`, `'dimezisBlurViewSdk31Plus'`
- `blurReductionFactor` (Android only): `number` (default `4`) — fine-tune to match iOS
- `blurTarget` (Android only): `RefObject<View | null>` — connects to a `BlurTargetView` for background blurring

```jsx
<BlurView intensity={100} tint="dark" style={styles.blurContainer}>
  <Text style={styles.text}>Blurred content</Text>
</BlurView>
```

SDK 56 change: Android blur support is now stable, requiring a `BlurTargetView` wrapper for proper background blurring.

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

```tsx
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

Notes: Android, iOS, tvOS, Web, Expo Go. No SDK 56 breaking changes.
