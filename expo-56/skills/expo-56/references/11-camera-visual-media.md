# Expo SDK 56 — Camera & Visual Media

Domain reference covering camera capture, image rendering, image picking, image manipulation, low-level GL rendering, and video thumbnails. Verified against the frozen SDK 56 API data (`docs/public/static/data/v56.0.0/expo-*.json`), the versioned doc pages (`docs/pages/versions/v56.0.0/sdk/...`), and package `src` + `CHANGELOG.md` on the `sdk-56` release branch.

```sh
npx expo install expo-camera expo-image expo-image-picker expo-image-manipulator expo-gl expo-video-thumbnails
```

## SDK 56 version pins

`expo-camera` `~56.0.8` · `expo-image` `~56.0.11` · `expo-image-picker` `~56.0.22` · `expo-image-manipulator` `~56.0.23` · `expo-gl` `~56.0.6` · `expo-video-thumbnails` `~56.0.3`.

Source: `git show origin/sdk-56:packages/expo/bundledNativeModules.json` — this file ships inside the `expo` package and is what `expo install` resolves against. Do **not** use `docs/public/static/schemas/v56.0.0/native-modules.json`; it is frozen at the SDK 56 docs cut and is now several patches stale. The SDK 56 line is still being patched, so these floors climb; the `~` ranges absorb it. For SDK 57 pins see [SDK 57 delta](#sdk-57-delta).

## SDK 56 breaking changes (all six packages)

- **All six** bumped the minimum iOS/tvOS deployment target to **16.4** (macOS 13.4) in `56.0.0` (#43296).
- **expo-camera 56.0.0** removed `RECORD_AUDIO` from the Android manifest — audio in video recording now depends entirely on the `recordAudioAndroid: true` plugin flag. This silently breaks video-with-audio on Android for anyone who relied on the implicit manifest permission. (#45132, filed under bug fixes but behaviourally breaking.)
- **expo-camera 56.0.0** extracted barcode scanning (ZXingObjC) into a separate `ExpoCameraBarcodeScanning` companion pod, so `barcodeScannerEnabled: false` now actually excludes ZXingObjC from precompiled builds (#44766). Filed under 💡 Others, not 🛠 Breaking changes — it is a build/binary-size change with no behavioural break; listed here only because it changes what the iOS build produces.

## Corrections to common assumptions

A model reproducing this domain from memory typically gets these wrong:

- `scanFromURLAsync` is a **module-level export**, not a `CameraView` static — and its runtime default is `['qr']` only.
- `ImageSource` has **no** `scale` field. `scale` is a read-only field on `ImageRef`.
- `Image.clearMemoryCache()` / `Image.clearDiskCache()` are **async** (`Promise<boolean>`).
- `ImageManipulator.manipulateAsync()` is deprecated; the context API (`manipulate` / `useImageManipulator`) replaces it.
- `ImagePicker.MediaTypeOptions` is deprecated; use the `MediaType` string union.
- `PictureRef` and both `ImageRef` types are `SharedRef`s; `ImageManipulatorContext` is a plain `SharedObject` (no `nativeRefType`). All four expose `release()` — see [Native image refs & memory](#native-image-refs--memory).
- `GLView.getWorkletContext` is deprecated and documented as broken; use the top-level `getWorkletContext` export.

---

## 1. expo-camera

Source: https://docs.expo.dev/versions/v56.0.0/sdk/camera/

### Core component: `CameraView`

A React component rendering the device front/back camera preview with configurable zoom, torch, flash, and focus. Only **one** active camera preview is allowed at a time — unmount when the screen is unfocused.

#### Key props

| Prop | Type | Default | Platforms |
|------|------|---------|-----------|
| `facing` | `'front' \| 'back'` | `'back'` | Android, iOS, Web |
| `flash` | `'off' \| 'on' \| 'auto' \| 'screen'` | `'off'` | Android, iOS, Web |
| `enableTorch` | `boolean` | `false` | Android, iOS, Web |
| `zoom` | `number` (0–1) | `0` | Android, iOS, Web |
| `mode` | `'picture' \| 'video'` | `'picture'` | Android, iOS, Web |
| `ratio` | `'4:3' \| '16:9' \| '1:1'` | — | Android |
| `mirror` | `boolean` | `false` | Android, iOS, Web |
| `mute` | `boolean` | `false` | Android, iOS, Web |
| `active` | `boolean` | `true` | iOS |
| `autofocus` | `'on' \| 'off'` | `'off'` | iOS |
| `videoQuality` | `VideoQuality` = `'2160p' \| '1080p' \| '720p' \| '480p' \| '4:3'` | — | Android, iOS, Web |
| `videoBitrate` | `number` | — | Android, iOS, Web |
| `videoStabilizationMode` | `'off' \| 'standard' \| 'cinematic' \| 'auto'` | `'auto'` | Android, iOS, Web |
| `barcodeScannerSettings` | `{ barcodeTypes: BarcodeType[] }` | — | Android, iOS, Web |
| `pictureSize` | `string` | — (iOS: `High`) | Android, iOS, Web |
| `selectedLens` | `string` | `'builtInWideAngleCamera'` | iOS |
| `animateShutter` | `boolean` | `true` | Android, iOS, Web |
| `poster` | `string` (URL) | — | Web |
| `responsiveOrientationWhenOrientationLocked` | `boolean` | — | iOS |

#### Callbacks

- `onCameraReady: () => void`
- `onMountError: (event: { message: string }) => void`
- `onBarcodeScanned: (result: BarcodeScanningResult) => void`
- `onAvailableLensesChanged: (event: { lenses: string[] }) => void`
- `onResponsiveOrientationChanged: (event: { orientation: CameraOrientation }) => void`

#### Capture / instance methods (ref)

- `takePictureAsync(options?: CameraPictureOptions)` → `Promise<CameraCapturedPicture>`. Options: `quality` (0–1, default `1`), `base64`, `exif`, `additionalExif` (android/ios), `onPictureSaved(picture)` — if set, the promise resolves immediately with no data and the result arrives in the callback — `skipProcessing`, `shutterSound` (default `true`), `pictureRef`, `scale` (web), `imageType` (web), `isImageMirror` (web), ~~`mirror`~~ (**deprecated** — use the `mirror` prop on `<CameraView>`). Result: `{ uri, width, height, base64?, exif?, format }`.
- `takePictureAsync({ pictureRef: true })` → `Promise<PictureRef>` (native image ref). `PictureRef extends SharedRef<'image'>`: `width`, `height`, `nativeRefType`, `savePictureAsync({ quality?, base64?, metadata? })`, and `release()` — see [Native image refs & memory](#native-image-refs--memory).
- `recordAsync(options?: CameraRecordingOptions)` → `Promise<{ uri: string } | undefined>`. Options: `maxDuration` (s), `maxFileSize` (bytes), `codec` (iOS: `'avc1' | 'hvc1' | 'jpeg' | 'apcn' | 'ap4h'`), ~~`mirror`~~ (**deprecated** — use the `mirror` prop).
- `stopRecording()` — stops active recording.
- `toggleRecordingAsync()` — pause/resume recording (iOS 18+; gate behind `getSupportedFeatures()`).
- `pausePreview()` / `resumePreview()` — control preview state.
- `getAvailablePictureSizesAsync()` → `Promise<string[]>`.
- `getAvailableLensesAsync()` → `Promise<string[]>` (iOS).
- `getSupportedFeatures()` → `{ isModernBarcodeScannerAvailable, toggleRecordingAsyncAvailable }`.

#### Module-level function (not a `CameraView` static)

```ts
import { scanFromURLAsync } from 'expo-camera';

scanFromURLAsync(url: string, barcodeTypes: BarcodeType[] = ['qr']): Promise<BarcodeScanningResult[]>
```

**Trap:** `scanFromURLAsync` is a top-level named export — `CameraView.scanFromURLAsync` does not exist. The runtime default for `barcodeTypes` is `['qr']` **only** (the JSDoc's "defaults to all supported bar code types" is wrong), so pass the array explicitly to scan anything else. Two further limits from the same docblock: **only QR codes are supported on iOS**, and on Android the barcode should fill most of the image. Source: `packages/expo-camera/src/index.ts:76-91`.

#### Static methods

- `CameraView.isAvailableAsync()` → `Promise<boolean>` (Web).
- `CameraView.launchScanner(options?)` — native scanner (Android / iOS 16+). Options: `{ barcodeTypes, isGuidanceEnabled?, isHighlightingEnabled?, isPinchToZoomEnabled? }`.
- `CameraView.dismissScanner()` — closes scanner (iOS).
- `CameraView.getAvailableVideoCodecsAsync()` → `Promise<VideoCodec[]>` (iOS).
- `CameraView.onModernBarcodeScanned(listener)` — subscription with `.remove()`.

#### Permission hooks

- `useCameraPermissions(options?)` → `[PermissionResponse | null, requestPermission, getPermission]`
- `useMicrophonePermissions(options?)` → `[PermissionResponse | null, requestPermission, getPermission]`

`PermissionResponse`: `{ status, granted, expires, canAskAgain }`. Status enum: `GRANTED | DENIED | UNDETERMINED`.

#### Barcode scanning

Supported types: `aztec`, `ean13`, `ean8`, `qr`, `pdf417`, `upc_e`, `datamatrix`, `code39`, `code93`, `itf14`, `codabar`, `code128`, `upc_a`.

`BarcodeScanningResult`: `{ type, data, bounds, cornerPoints, extra? }` where `bounds = { origin: { x, y }, size: { width, height } }`.

Corner point order differs by platform (`packages/expo-camera/src/Camera.types.ts:318-335`):

- **Android / Web**: `topLeft, topRight, bottomRight, bottomLeft` (MLKit / `BarcodeDetector` order).
- **iOS**: `bottomLeft, bottomRight, topLeft, topRight`.

Two caveats the docs bury: on iOS `cornerPoints` is **not** returned for `code39` and `pdf417`; and `bounds` may be an empty rectangle and does not necessarily bound the whole barcode — for some types it is the scanner's search area.

#### Config plugin (app.json)

```json
{
  "expo": {
    "plugins": [
      [
        "expo-camera",
        {
          "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera",
          "microphonePermission": "Allow $(PRODUCT_NAME) to access your microphone",
          "recordAudioAndroid": true,
          "barcodeScannerEnabled": true
        }
      ]
    ]
  }
}
```

Plugin prop types (`packages/expo-camera/plugin/src/withCamera.ts:16-39`): `cameraPermission` and `microphonePermission` are `string | false` — pass `false` to omit the Info.plist key entirely. `recordAudioAndroid` defaults to `true`; `barcodeScannerEnabled` defaults to `true`.

Manual (non-CNG): Android manifest needs `CAMERA` + `RECORD_AUDIO` and the `expo-camera/android/maven` repo; iOS Info.plist needs `NSCameraUsageDescription` + `NSMicrophoneUsageDescription`. Since 56.0.0 the library no longer merges `RECORD_AUDIO` into the manifest itself.

#### Minimal usage

```tsx
import { CameraView, useCameraPermissions, type CameraType } from 'expo-camera';
import { useState } from 'react';
import { Button, View } from 'react-native';

export default function App() {
  // Must be annotated — bare useState('back') infers `string`, which fails the `facing` prop.
  const [facing, setFacing] = useState<CameraType>('back');
  const [permission, requestPermission] = useCameraPermissions();

  if (!permission?.granted) {
    return <Button onPress={requestPermission} title="Grant access" />;
  }

  return (
    <View style={{ flex: 1 }}>
      <CameraView style={{ flex: 1 }} facing={facing} />
      <Button onPress={() => setFacing(f => f === 'back' ? 'front' : 'back')}
              title="Flip" />
    </View>
  );
}
```

#### Constraints / gotchas

- Call `takePictureAsync` only after `onCameraReady` fires.
- Avoid capture during paused preview (Android throws; iOS captures last frame).
- Web returns base64 URIs rather than file paths.
- Chrome iframes need `allow="microphone; camera;"`.

---

## 2. expo-image

Source: https://docs.expo.dev/versions/v56.0.0/sdk/image/

### Exported components

- **`Image`** — cross-platform image component (Android, iOS, tvOS, Web, Expo Go). Disk + memory caching, animated formats (GIF, WebP, APNG), BlurHash/ThumbHash placeholders. Backed by SDWebImage (iOS) and Glide (Android).
- **`ImageBackground`** — image as a background with children rendered on top. All `Image` props plus separate `style` (container) and `imageStyle` (background).

### Key props

- Source/loading: `source` (URL, local, `require()`, or array for responsive selection), `cachePolicy` (`'none' | 'disk' | 'memory' | 'memory-disk'`, default `'disk'`), `priority` (`'normal' | 'low' | 'high'`), `autoplay` (default `true`).
- Sizing/position: `contentFit` (`'cover' | 'contain' | 'fill' | 'none' | 'scale-down'`, default `'cover'`), `contentPosition` (default `'center'`), `allowDownscaling` (default `true`).
- Placeholders: `placeholder` (string/number/ImageSource/array), `placeholderContentFit` (default `'scale-down'`).
- Transitions: `transition` (ms number or `ImageTransition` object; effects include `'cross-dissolve'`, `'flip-from-top'`, `'curl-up'`, SF Symbol animations).
- Visual: `blurRadius` (default `0`), `tintColor`, `preferHighDynamicRange` (iOS/tvOS 17+).
- Caching/perf: `recyclingKey` (list recycling / FlashList), `decodeFormat` (Android: `'argb' | 'rgb'`, default `'argb'`), `enforceEarlyResizing` (iOS).
- Accessibility/Web: `accessibilityLabel`/`alt`, `accessible`, `focusable` (Android, default `false`), `loading` (`'lazy' | 'eager'`, Web), `draggable` (Web), `responsivePolicy` (`'static' | 'initial' | 'live'`, default `'static'`, Web).
- Advanced: `enableLiveTextInteraction` (iOS 16+), `sfEffect` (iOS 17+), `useAppleWebpCodec` (iOS, default `true`).
- Callbacks: `onLoadStart`, `onProgress({ loaded, total })`, `onLoad({ cacheType, source })`, `onLoadEnd`, `onDisplay`, `onError`.

> **Web `loading` default changed in 56.0.10 (#46425).** When `responsivePolicy` is `'static'` (the default), `loading` now defaults to **`'lazy'`**, not `'eager'`. The generated `sizes` leads with the `auto` keyword so the browser picks the source from the rendered layout size, and `sizes="auto"` only applies to lazily-loaded images. Opt out with `loading="eager"`, which also disables `sizes="auto"`. The frozen v56 doc page still prints `@default 'eager'` because it was cut before 56.0.10 — the source docblock (`packages/expo-image/src/Image.types.ts:224-237`) is correct.

### Hook

`useImage(source, options?, dependencies?)` → `ImageRef | null`. Options: `maxWidth`, `maxHeight`, `tintColor`, `onError(error, retry)`. The hook calls `image.release()` in its effect cleanup, so refs it returns are managed for you; `Image.loadAsync()` refs are not.

```jsx
const image = useImage('https://picsum.photos/1000/800', {
  maxWidth: 800,
  onError(error, retry) { console.error(error.message); }
});
```

### Static methods

- `Image.prefetch(urls, cachePolicy?)` / `Image.prefetch(urls, options?)` (options support `headers`) → `Promise<boolean>`
- `Image.loadAsync(source, options?)` → `Promise<ImageRef>`
- `Image.clearMemoryCache()` → `Promise<boolean>` (**async** — not a void call)
- `Image.clearDiskCache()` → `Promise<boolean>` (**async**)
- `Image.getCachePathAsync(cacheKey)` → `Promise<string | null>`
- `Image.generateBlurhashAsync(source, numberOfComponents?)`
- `Image.generateThumbhashAsync(source)`
- `Image.configureCache(config)` (iOS only)

### Instance methods

`getAnimatableRef()`, `startAnimating()`, `stopAnimating()`, `lockResourceAsync()`, `unlockResourceAsync()`, `reloadAsync()`.

### Key types

`ImageSource`: `{ uri?, headers?, width?: number | null, height?: number | null, blurhash?, thumbhash?, cacheKey?, webMaxViewportWidth? (web, deprecated), isAnimated? (android/ios) }`.

**Trap:** `ImageSource` has **no `scale` field** — passing one is a TS error. Expo's own JSDoc on the `source` prop misleadingly says "it is important to provide `width`, `height` and `scale` properties"; `scale` lives on `ImageRef`. Source: `packages/expo-image/src/Image.types.ts:16-77`.

`ImageTransition`: `{ duration, effect, timing }`.

`ImageRef extends SharedRef<'image'>` — readonly `width`, `height`, `scale`, `mediaType` (iOS, `string | null`), `isAnimated?`; plus `nativeRefType` and `release()` from the base class. See [Native image refs & memory](#native-image-refs--memory).

### Supported formats

WebP, PNG, APNG, AVIF, JPEG, GIF, SVG, ICO: all platforms. HEIC: Android + iOS (not Web). ICNS, PSD preview: iOS only.

### Config plugin (app.json)

```json
{
  "expo": {
    "plugins": [
      ["expo-image", { "disableLibdav1d": true }]
    ]
  }
}
```

`disableLibdav1d` (iOS, default `false`): when `true`, skips adding the bundled `libavif`/`libdav1d` Pod. Use it when another dependency (for example FFmpegKit) already links `libdav1d`. **Disabling the Pod removes AVIF support on iOS** unless another decoder is available — which contradicts the "AVIF: all platforms" row above. Source: `docs/pages/versions/v56.0.0/sdk/image.mdx:78-91`.

Bare RN: set `EXPO_IMAGE_DISABLE_LIBDAV1D=1` (or `=true`) before `pod install`, or add `ENV['EXPO_IMAGE_DISABLE_LIBDAV1D'] ||= '1'` to the top of the Podfile. **The upstream v56 doc page prints `||= '0'` — that is a typo and a no-op**: the podspec only disables the Pod when the value is exactly `'1'` or `'true'`, so `'0'` leaves libdav1d linked. Note also that the `Podfile.properties.json` key `expo-image.disable-libdav1d` (`'true'`/`'false'`) is what the config plugin actually writes, and it **takes precedence over the env var** — if the key is present the env var is ignored entirely. Source: `packages/expo-image/ios/ExpoImage.podspec:4-12`.

### Minimal usage

```jsx
import { Image } from 'expo-image';
import { StyleSheet, View } from 'react-native';

const blurhash = '|rF?hV%2WCj[ayj[a|j[az_NaeWBj@ayfRayfQfQM{M|...';

export default function App() {
  return (
    <View style={styles.container}>
      <Image
        style={styles.image}
        source="https://picsum.photos/seed/696/3000/2000"
        placeholder={{ blurhash }}
        contentFit="cover"
        transition={1000}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff', justifyContent: 'center' },
  image: { flex: 1, width: '100%' },
});
```

### Deprecations

- `defaultSource` → use `placeholder`
- `fadeDuration` → use `transition`
- `loadingIndicatorSource` → use `placeholder`
- `resizeMode` → use `contentFit` + `contentPosition` (note: `'repeat'` is unsupported)
- `webMaxViewportWidth` (deprecated in 56.0.10) → with layout-based selection it now only emits fallback `sizes` breakpoints for browsers without `sizes="auto"` support

---

## 3. expo-image-picker

Source: https://docs.expo.dev/versions/v56.0.0/sdk/imagepicker/

### Permission hooks

- `useCameraPermissions(options?)` → `[PermissionResponse | null, requestPermission, getPermission]`
- `useMediaLibraryPermissions(options?)` → `[MediaLibraryPermissionResponse | null, requestPermission, getPermission]`. Options: `PermissionHookOptions<{ writeOnly: boolean }>`.

### Methods

- `launchImageLibraryAsync(options?)` → `Promise<ImagePickerResult>`
- `launchCameraAsync(options?)` → `Promise<ImagePickerResult>` (since 56.0.16 this can also be invoked on the **iOS Simulator**, #45923)
- `getPendingResultAsync()` → `Promise<ImagePickerErrorResult | ImagePickerResult | null>` (Android — recovers data lost after MainActivity is destroyed)
- `requestCameraPermissionsAsync()` → `Promise<CameraPermissionResponse>`
- `requestMediaLibraryPermissionsAsync(writeOnly?)` → `Promise<MediaLibraryPermissionResponse>`
- `getCameraPermissionsAsync()` → `Promise<CameraPermissionResponse>`
- `getMediaLibraryPermissionsAsync(writeOnly?)` → `Promise<MediaLibraryPermissionResponse>`

### `ImagePickerOptions`

| Property | Type | Notes |
|----------|------|-------|
| `allowsEditing` | `boolean` | Default `false`. Android/iOS. |
| `allowsMultipleSelection` | `boolean` | Default `false`. Android, iOS 14+, Web. Mutually exclusive with `allowsEditing`. |
| `aspect` | `[number, number]` | Android crop ratio. |
| `base64` | `boolean` | Include Base64 data. |
| `cameraType` | `CameraType` | `back` (default) or `front`. |
| `defaultTab` | `'photos' \| 'albums'` | Android. Default `'photos'`. |
| `exif` | `boolean` | Android/iOS. |
| `legacy` | `boolean` | Android. Default `false`. |
| `mediaTypes` | `MediaType \| MediaType[] \| MediaTypeOptions` | Default `'images'`. |
| `orderedSelection` | `boolean` | iOS 15+. Default `false`. |
| `preferredAssetRepresentationMode` | enum | iOS 14+. Default `Automatic`. |
| `presentationStyle` | enum | iOS. Default `Automatic`. |
| `quality` | `number` | 0–1. Default `1.0`. |
| `selectionLimit` | `number` | iOS 14+, Android. Default `0` (no limit). |
| `shape` | `'rectangle' \| 'oval'` | Android. Default `'rectangle'`. |
| `shouldDownloadFromNetwork` | `boolean` | iOS. Default `false`. |
| `videoExportPreset` | `VideoExportPreset` | iOS 11+. Default `Passthrough`. (Deprecated by Apple.) |
| `videoMaxDuration` | `number` | Seconds; `0` = unlimited. |
| `videoQuality` | enum | iOS. |

### Media types

`MediaType`: `'images'`, `'videos'`, `'livePhotos'` (iOS only; ignored elsewhere).
`MediaTypeOptions` (**deprecated**, still available): `All`, `Images`, `Videos`.

### Result shapes

`ImagePickerResult = ImagePickerSuccessResult | ImagePickerCanceledResult`

```typescript
// success
{ canceled: false, assets: ImagePickerAsset[] }
// canceled
{ canceled: true, assets: null }
```

`ImagePickerAsset` core fields: `uri` (string), `width`/`height` (number), `assetId` (string|null), `fileName` (string|null), `fileSize` (number), `type` (`'image' | 'video' | 'livePhoto' | 'pairedVideo' | null`), `duration` (ms, video), `base64` (string|null), `exif` (Record|null), `mimeType` (string), `file` (Web only `File`), `pairedVideoAsset` (ImagePickerAsset|null, iOS live photo).

### Enums

- `CameraType`: `back`, `front`
- `VideoExportPreset`: `Passthrough`, `LowQuality`, `MediumQuality`, `HighestQuality`, `H264_640x480`, `H264_960x540`, `H264_1280x720`, `H264_1920x1080`, `H264_3840x2160`, `HEVC_1920x1080`, `HEVC_3840x2160`
- `UIImagePickerControllerQualityType`: `High`, `Medium`, `Low`, `VGA640x480`, `IFrame1280x720`, `IFrame960x540`
- `UIImagePickerPreferredAssetRepresentationMode` (iOS): `Automatic`, `Compatible`, `Current`
- `UIImagePickerPresentationStyle` (iOS): `AUTOMATIC`, `CURRENT_CONTEXT`, `FORM_SHEET`, `FULL_SCREEN`, `OVER_CURRENT_CONTEXT`, `OVER_FULL_SCREEN`, `PAGE_SHEET`, `POPOVER`
- `PermissionStatus`: `GRANTED`, `DENIED`, `UNDETERMINED`

### Config plugin (app.json)

```json
{
  "expo": {
    "plugins": [
      [
        "expo-image-picker",
        {
          "photosPermission": "The app accesses your photos...",
          "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera",
          "microphonePermission": "Allow $(PRODUCT_NAME) to access your microphone"
        }
      ]
    ]
  }
}
```

Plugin prop types (`packages/expo-image-picker/plugin/src/withImagePicker.ts:18-58`):

- `photosPermission | cameraPermission | microphonePermission` — each `string | false`. Pass `false` to omit the Info.plist key entirely (and to block the Android permission for camera/microphone).
- `colors` and `dark: { colors }` (Android crop UI), where `ImagePickerColors = { cropToolbarColor?, cropToolbarIconColor?, cropToolbarActionTextColor?, cropBackButtonIconColor?, cropBackgroundColor? }` — all hex strings.

iOS manual setup needs `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSMicrophoneUsageDescription`.

### Minimal usage

```tsx
import { useState } from 'react';
import { Alert, Button, Image, View, StyleSheet } from 'react-native';
import * as ImagePicker from 'expo-image-picker';

export default function ImagePickerExample() {
  const [image, setImage] = useState<string | null>(null);

  const pickImage = async () => {
    const permissionResult =
      await ImagePicker.requestMediaLibraryPermissionsAsync();

    if (!permissionResult.granted) {
      Alert.alert('Permission required',
        'Permission to access the media library is required.');
      return;
    }

    let result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ['images', 'videos'],
      allowsEditing: true,
      aspect: [4, 3],
      quality: 1,
    });

    if (!result.canceled) {
      setImage(result.assets[0].uri);
    }
  };

  return (
    <View style={styles.container}>
      <Button title="Pick an image from camera roll" onPress={pickImage} />
      {image && <Image source={{ uri: image }} style={styles.image} />}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  image: { width: 200, height: 200 },
});
```

### Notes / known issues

- `MediaTypeOptions` is deprecated — use the `MediaType` string union (`'images'`, `'videos'`, `'livePhotos'`).
- iOS cropping of high-resolution camera-roll images may produce an incorrect crop rectangle (underlying `UIImagePickerController` bug).
- Web permission dialogs must follow an immediate user interaction (e.g. a button press).

---

## 4. expo-image-manipulator

Source: https://docs.expo.dev/versions/v56.0.0/sdk/imagemanipulator/

### APIs

**New context-based API (recommended):**
- `useImageManipulator(source: string | SharedRef<'image'>)` — hook returning an `ImageManipulatorContext`
- `ImageManipulator.manipulate(source: string | SharedRef<'image'>)` — returns an `ImageManipulatorContext`

`source` is **not URI-only**. Any `SharedRef<'image'>` works, so an expo-image `ImageRef`, an expo-camera `PictureRef`, or an expo-video `VideoThumbnail` can be handed in directly with no round trip through the filesystem. Source: `packages/expo-image-manipulator/src/ImageManipulator.types.ts:142`.

`useImageManipulator` is built on `useReleasingSharedObject`, so it releases the context for you. `ImageManipulator.manipulate()` does **not** — call `context.release()` (and `image.release()` on the rendered `ImageRef`) yourself.

**Deprecated:** `ImageManipulator.manipulateAsync(uri, actions, saveOptions)` — replaced by the new contextual, object-oriented API.

### `ImageManipulatorContext` (chainable)

- `rotate(degrees)` — clockwise for positive, counter-clockwise for negative.
- `flip(flipType)` — `'vertical'` or `'horizontal'` (`FlipType.Vertical` / `FlipType.Horizontal`).
- `resize(size)` — `{ width, height }`; a missing dimension is auto-computed to preserve aspect ratio.
- `crop(rect)` — `{ originX, originY, width, height }`.
- `extent(options)` — **Web only**; `{ originX?: number, originY?: number, width: number, height: number, backgroundColor?: string | null }`. Sets the canvas size and offsets the image inside it; enlarged/unfilled areas are painted with `backgroundColor`. (There is no `size` or `offset` key.)
- `reset()` — revert to the original image.
- `renderAsync()` — awaits scheduled transforms, returns an `ImageRef`.
- `release()` — inherited from `SharedObject`; free the native context when you created it with `manipulate()`.

### `ImageRef` (image-manipulator)

`ImageRef extends SharedRef<'image'>` with `width`, `height`, `release()`, and:

```ts
saveAsync(options?: SaveOptions): Promise<ImageResult>
// ImageResult = { uri: string; width: number; height: number; base64?: string }
```

Persists into the app cache directory. On Android this is the *scoped* cache directory since 56.0.8 (#45708).

### `SaveOptions`

- `format` — `SaveFormat.JPEG | SaveFormat.PNG | SaveFormat.WEBP`. **Defaults to `SaveFormat.JPEG`** when omitted.
- `compress` — `0.0`–`1.0` (`1` = no compression / highest quality)
- `base64` — boolean; when truthy, `ImageResult.base64` carries the JPEG/PNG data (prepend `data:image/xxx;base64,` for a data URI)

On Web, requesting an unsupported mime type throws since 56.0.14 (#46165) instead of silently falling back.

### Enums

- `FlipType`: `Horizontal`, `Vertical`
- `SaveFormat`: `JPEG`, `PNG`, `WEBP`

### Minimal usage

```jsx
const context = useImageManipulator(imageUri);
context.rotate(90).flip(FlipType.Vertical);
const renderedImage = await context.renderAsync();
const result = await renderedImage.saveAsync({ format: SaveFormat.PNG });
```

---

## 5. expo-gl (GLView)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/gl-view/

### `GLView` component

Platforms: Android, iOS, Web, Expo Go. Extends `ViewProps`.

#### Props

| Prop | Type | Platform | Default | Description |
|------|------|----------|---------|-------------|
| `onContextCreate` | `(gl: ExpoWebGLRenderingContext) => void` | All | required | Called when the OpenGL ES context is created. |
| `msaaSamples` | `number` | iOS | `4` | Multisampling sample count; `0` disables. |
| `enableExperimentalWorkletSupport` | `boolean` | All | `false` | Enables Reanimated worklet-thread integration. |

#### Static methods

- `GLView.createContextAsync()` → `Promise<ExpoWebGLRenderingContext>` (headless context, no view). A context created this way **must** be destroyed with `GLView.destroyContextAsync()`, and you must set up your own viewport, framebuffer and texture before taking a snapshot.
- `GLView.destroyContextAsync(exgl?)` → `Promise<boolean>`. Intended for headless contexts from `createContextAsync`.
- `GLView.takeSnapshotAsync(exgl?, options?)` → `Promise<GLSnapshot>`.
- ~~`GLView.getWorkletContext(contextId)`~~ — **deprecated (since SDK 54 or earlier, not new in 56)**: "This method doesn't work inside of the worklets with new reanimated versions." Use the top-level export instead: `import { getWorkletContext } from 'expo-gl'`. Source: `packages/expo-gl/src/GLView.tsx:32, 87-93`.

#### Instance methods

- `takeSnapshotAsync(options?)` → `Promise<GLSnapshot>`
- `createCameraTextureAsync(cameraRefOrHandle)` → `Promise<WebGLTexture>`
- `destroyObjectAsync(glObject)` → `Promise<boolean>`

### `ExpoWebGLRenderingContext`

Extends `WebGL2RenderingContext`. Property: `contextId: number`. Methods: `endFrameEXP()` (present the frame), `flushEXP()`, `__expoSetLogging(option)`, `_expo_texImage2D(...)`, `_expo_texSubImage2D(...)`.

### `SnapshotOptions`

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `format` | `'jpeg' \| 'png' \| 'webp'` | `'jpeg'` | WebP produces PNG on iOS. |
| `compress` | `number` | `1.0` | 0–1.0. |
| `flip` | `boolean` | `false` | Vertical flip. |
| `rect` | `{ x, y, width, height }` | — | Crop region. |
| `framebuffer` | `WebGLFramebuffer` | — | Defaults to view framebuffer. |

### `GLSnapshot`

`{ uri: string | Blob | null, localUri: string, width: number, height: number }`

### Minimal usage

```jsx
import { View } from 'react-native';
import { GLView } from 'expo-gl';

export default function App() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <GLView style={{ width: 300, height: 300 }} onContextCreate={onContextCreate} />
    </View>
  );
}

function onContextCreate(gl) {
  gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
  gl.clearColor(0, 1, 1, 1);
  gl.clear(gl.COLOR_BUFFER_BIT);
  gl.endFrameEXP();
}
```

### Notes / deprecations

- Some WebGL2 methods are unimplemented (e.g. `getFramebufferAttachmentParameter()`, `compressedTexImage2D()`, `fenceSync()`).
- Legacy AR session methods (`startARSessionAsync`) were **removed entirely in SDK 55** — not merely undocumented. They are absent from `packages/expo-gl/src` on `sdk-56`/`sdk-57` and from the v55+ API data, so there is nothing to call at runtime.

---

## 6. expo-video-thumbnails

Source: https://docs.expo.dev/versions/v56.0.0/sdk/video-thumbnails/

> **Deprecated:** the library is not receiving patches; use `generateThumbnailsAsync` from **expo-video** instead. The docs' own wording ("will be removed in SDK 56", still printed verbatim on the v57 page at `docs/pages/versions/v57.0.0/sdk/video-thumbnails.mdx:14`) is stale — the package still ships in SDK 56 (`~56.0.3`) and SDK 57 (`~57.0.1`).
>
> The replacement is a **`VideoPlayer` instance method**, not a module function:
> `player.generateThumbnailsAsync(times: number | number[], options?: VideoThumbnailOptions): Promise<VideoThumbnail[]>` (Android/iOS only). The returned `VideoThumbnail`s are native image refs, usable directly as an expo-image `source`. Source: `packages/expo-video/src/VideoPlayer.types.ts:306-315`.

### Method

`VideoThumbnails.getThumbnailAsync(sourceFilename, options?)` → `Promise<VideoThumbnailsResult>`. Generates an image thumbnail from a video file. Supported on Android, iOS, tvOS. No special permission hooks; reads local or remote URIs.

#### Options

| Property | Type | Notes |
|----------|------|-------|
| `time` | `number` | Frame position in milliseconds. |
| `quality` | `number` | `0.0`–`1.0` (1.0 = highest). |
| `headers` | `Record<string, string>` | Custom headers for remote URIs. |

#### `VideoThumbnailsResult`

`{ uri: string, width: number, height: number }`

### Minimal usage

```jsx
const { uri } = await VideoThumbnails.getThumbnailAsync(videoUrl, {
  time: 15000,
});
```

---

## Native image refs & memory

Three of the six packages hand back native handles rather than file URIs (as does expo-video's `VideoThumbnail`). All of them are `SharedRef` / `SharedObject` subclasses (`packages/expo-modules-core/src/ts-declarations/SharedObject.ts`, `SharedRef.ts`), which means they carry `nativeRefType`, the `EventEmitter` surface (`addListener`, `removeListener`, `emit`, `listenerCount`, `removeAllListeners`, `startObserving`, `stopObserving`), and:

```ts
release(): void  // detaches the JS object from its native counterpart
```

| Ref | Package | Auto-released? |
|-----|---------|----------------|
| `PictureRef` | expo-camera (`takePictureAsync({ pictureRef: true })`) | No |
| `ImageRef` | expo-image (`Image.loadAsync`) | No |
| `ImageRef` | expo-image (`useImage`) | **Yes** — effect cleanup calls `release()` |
| `ImageRef` | expo-image-manipulator (`context.renderAsync()`) | No |
| `ImageManipulatorContext` | expo-image-manipulator (`manipulate()`) | No |
| `ImageManipulatorContext` | expo-image-manipulator (`useImageManipulator()`) | **Yes** — `useReleasingSharedObject` |

Rules of thumb:

- Any call to `release()` detaches the native object permanently; subsequent native calls on that ref throw. Only release when nothing else holds the ref.
- In loops and lists, release refs you no longer render — native bitmaps otherwise accumulate until GC catches up, which is the usual cause of "memory grows until the app is killed" reports in this domain.
- The deprecated `manipulateAsync()` already releases both its context and its rendered image internally (`packages/expo-image-manipulator/src/ImageManipulator.ts:50-52`); the context API leaves it to you.
- Related but distinct: `<Image>` instances expose `lockResourceAsync()` / `unlockResourceAsync()` (Android/iOS) to prevent and re-allow reloading of the displayed resource, and `recyclingKey` to invalidate an image when a recycled list cell is reused.

---

## SDK 56 patch drift

Behaviour that changed after the initial SDK 56 release and is therefore missing from the frozen v56 doc pages. The "min patch" column is the version you must be on to get the change **without leaving the SDK 56 line**. Verified against `git show origin/sdk-56:packages/<pkg>/CHANGELOG.md`.

| Min patch | Change |
|-----------|--------|
| expo-image **56.0.10** | Web `loading` defaults to `'lazy'` under the default `'static'` `responsivePolicy`; `webMaxViewportWidth` deprecated (#46425) |
| expo-image 56.0.11 | Android: image no longer stays blank when `source` changes mid-`transition` (#46752) |
| expo-image 56.0.9 | iOS: `placeholder` loads from the asset catalog (xcassets) (#46170) |
| expo-image 56.0.8 | Android: `useImage` no longer crashes on SVG sources; `maxWidth`/`maxHeight` preserve SVG aspect ratio (#46077) |
| expo-image 56.0.6 | Android: `tintColor` in `loadAsync`/`useImage` no longer requires API 26 (#45981) |
| expo-image 56.0.5 | iOS: `recyclingKey` clears the cached `placeholderImage` (no more stale blurhash in recycled cells) (#44762); Xcode asset-catalog images load by resource name (#45686) |
| expo-image-picker **56.0.16** | iOS: `launchCameraAsync` can be invoked on the Simulator (#45923) |
| expo-image-picker 56.0.7 | Android: crop activity follows the system day/night theme (#44944) |
| expo-image-manipulator 56.0.14 | Web: throws on an unsupported requested mime type (#46165) |
| expo-image-manipulator 56.0.8 | Android: saves into the scoped cache directory (#45708) |
| expo-gl 56.0.6 | Docs-only clarification of `createContextAsync`/`destroyContextAsync` (#47200) — **also** shipped as expo-gl `57.0.1`, so it is a backport, not an SDK 57 feature |
| expo-gl 56.0.5 | `WebGLRenderingContext`/`WebGL2RenderingContext` globals installed at module init (#45865); `WebGL2RenderingContext` now inherits from `WebGLRenderingContext`, so `gl2 instanceof WebGLRenderingContext` is `true` (#45871) |
| expo-camera | No user-facing changes in 56.0.6–56.0.8 (56.0.7 was a config-plugin ESM import fix) |
| expo-image-picker 56.0.17–56.0.22, expo-image-manipulator 56.0.15–56.0.23 | All "does not introduce any user-facing changes" — version churn only |

---

## Deprecation summary

Migration table — old symbol → replacement → deprecated since.

| Package | Deprecated | Replacement | Since |
|---------|-----------|-------------|-------|
| expo-camera | `CameraPictureOptions.mirror`, `CameraRecordingOptions.mirror` | `mirror` prop on `<CameraView>` | ≤54 |
| expo-image | `defaultSource`, `loadingIndicatorSource` | `placeholder` | ≤56 |
| expo-image | `fadeDuration` | `transition` | ≤56 |
| expo-image | `resizeMode` | `contentFit` + `contentPosition` (`'repeat'` unsupported) | ≤56 |
| expo-image | `webMaxViewportWidth` | layout-based `sizes="auto"` selection | 56.0.10 |
| expo-image-picker | `MediaTypeOptions` enum | `MediaType` string union (`'images' \| 'videos' \| 'livePhotos'`) | ≤56 |
| expo-image-picker | `videoExportPreset` | — (deprecated by Apple) | ≤56 |
| expo-image-manipulator | `manipulateAsync()` | `manipulate()` / `useImageManipulator()` | ≤56 |
| expo-gl | `GLView.getWorkletContext()` | top-level `getWorkletContext` export | ≤54 |
| expo-gl | `startARSessionAsync` and friends | — (**removed outright in SDK 55**; not callable in 56/57) | ≤54 |
| expo-video-thumbnails | whole library | `player.generateThumbnailsAsync()` from expo-video | ≤56 |

---

## SDK 57 delta

Verified against `git show origin/sdk-57:packages/<pkg>/CHANGELOG.md` and cross-checked against `origin/sdk-56` to weed out backports. Almost nothing changed here: five of the six packages shipped `_This version does not introduce any user-facing changes._` at `57.0.0`. The only API-surface addition in the whole domain is two `expo-image` cache statics; everything else is expo-camera iOS behaviour landing in the `57.0.1` / `57.0.3` patches. Both are genuine 57-only deltas — neither appears anywhere on the SDK 56 line (expo-camera stops at `56.0.8`, expo-image at `56.0.11`).

### Breaking

Nothing was filed under breaking changes, but two expo-camera patches change observable output on iOS:

- **`pictureSize` now defaults to `photo` on iOS instead of `high`** (expo-camera `57.0.1`, #47173). Captures are full-resolution by default, so `takePictureAsync()` returns larger files and different `width`/`height` than on SDK 56. Set `pictureSize="High"` explicitly to keep SDK 56 sizing and latency — the iOS value strings are case-sensitive (`"3840x2160"`, `"1920x1080"`, `"1280x720"`, `"640x480"`, `"352x288"`, `"Photo"`, `"High"`, `"Medium"`, `"Low"`, exactly what `getAvailablePictureSizesAsync()` returns); an unrecognised string like `"high"` throws `EnumNoSuchValueException` natively. Implementation: `packages/expo-camera/ios/CameraViewModule.swift:124-125` on `origin/sdk-57` (`if pictureSize == nil && view.pictureSize != .photo { view.pictureSize = .photo }`); enum raw values in `packages/expo-camera/ios/Current/CameraRecordingOptions.swift:56-65`. Filed under 🐛 Bug fixes, not 🛠 Breaking changes.
- **iOS capture orientation is preserved as EXIF metadata instead of rotating the pixels** (expo-camera `57.0.3`, #47824). `takePictureAsync` and `PictureRef.savePictureAsync` now return the real orientation tag and native dimensions, and no longer save captures above their native resolution. Code that assumed pixels were pre-rotated, or that read `width`/`height` without consulting EXIF orientation, needs re-testing.

### New in 57

**expo-image `57.0.0`** (#46620) — the only new API in this domain. Both are Android + iOS only (`packages/expo-image/src/Image.tsx:156-188`):

```ts
Image.writeToCacheAsync(source: string | ImageRef, cacheKey: string): Promise<void>
Image.readFromCacheAsync(cacheKey: string): Promise<ImageRef | null>
```

Seed the disk cache from an image you already have on device (an `expo-image-picker` result, a file you downloaded with `expo-file-system`) so a later render whose `source.cacheKey` matches is served straight from cache. `readFromCacheAsync` resolves `null` when nothing is cached, and returns an `ImageRef` you can pass directly as a `source`.

> Gotcha: caching an **animated** image (GIF, APNG, animated WebP) from an `ImageRef` flattens it to a single frame, because the ref holds the decoded image rather than the encoded bytes. Pass the local file URI instead to seed animation losslessly.

**expo-camera `57.0.1` / `57.0.3`** — iOS preview and capture work (#47172, #47173, #47477, #47816, #47818):

- Responsive capture + fast capture prioritization for lower shutter lag on successive captures.
- Preview is hosted on the view's backing layer, so it no longer zooms into place on launch.
- Deprecated `videoOrientation` replaced with `AVCaptureDevice.RotationCoordinator`.
- The session is fully configured before it starts running — fixes dark first frames and the preview rotating into place.
- Photos are processed off the main thread, and a redundant full-resolution JPEG re-encode was removed (fixes UI hangs on older devices).
- Deferred photo delivery disabled — responsive capture was hanging the capture promise on iOS 17+.
- Android: `onMountError` fires instead of crashing when the camera can't start (for example a device reporting zero cameras).

### Unchanged in 57

Verified package by package — the SDK 56 sections above apply verbatim:

- **expo-image-picker** — `57.0.0` through `57.0.6` are all "does not introduce any user-facing changes". (The Simulator `launchCameraAsync` support is an SDK **56** feature, shipped in 56.0.16, merely inherited by 57.)
- **expo-image-manipulator** — `57.0.0`–`57.0.6`, no user-facing changes.
- **expo-gl** — no API change. `57.0.1` was a docs-only clarification (#47200) of `createContextAsync`/`destroyContextAsync`, already folded into the expo-gl section above — and it was **backported to expo-gl `56.0.6`**, so it is not a reason to upgrade.
- **expo-video-thumbnails** — `57.0.0`–`57.0.1`, no user-facing changes. Still deprecated, still shipping.

Not in SDK 57 despite appearing on `main`: expo-camera document scanning (`CameraView.isDocumentScannerAvailable`, `CameraView.scanDocumentAsync`, #47359 merged 2026-07-08) and the expo-image `imageLoaded` module event (#47337) both landed after the SDK 57 docs cut, are absent from `docs/public/static/data/v57.0.0/expo-camera.json` and `.../expo-image.json`, and are absent from every `57.0.x` CHANGELOG section on `origin/sdk-57`. Treat them as unreleased / post-57.

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-camera` | `~56.0.8` | `~57.0.3` |
| `expo-image` | `~56.0.11` | `~57.0.1` |
| `expo-image-picker` | `~56.0.22` | `~57.0.6` |
| `expo-image-manipulator` | `~56.0.23` | `~57.0.6` |
| `expo-gl` | `~56.0.6` | `~57.0.2` |
| `expo-video-thumbnails` | `~56.0.3` | `~57.0.1` |

Source: `git show origin/sdk-56:packages/expo/bundledNativeModules.json` and `git show origin/sdk-57:packages/expo/bundledNativeModules.json`. **Do not** read these from `docs/public/static/schemas/v5*.0.0/native-modules.json` — those are frozen at the docs cut and understate every pin here. Note the SDK 57 pins are **not** a flat `~57.0.0`: each package tracks its own patch line, and both lines keep moving, so re-read `bundledNativeModules.json` rather than hard-coding these.
