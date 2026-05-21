# Expo SDK 56 — Camera & Visual Media

Domain reference covering camera capture, image rendering, image picking, image manipulation, low-level GL rendering, and video thumbnails. All content sourced from the official Expo SDK 56 docs (`https://docs.expo.dev/versions/v56.0.0/sdk/...`) and cross-checked against the SDK 56 changelog (`https://expo.dev/changelog/sdk-56`).

> Changelog note: The SDK 56 changelog does not list package-specific breaking changes for any of the six packages below. The version-pinned doc pages are treated as authoritative for the API surface.

---

## 1. expo-camera

Source: https://docs.expo.dev/versions/v56.0.0/sdk/camera/

### Installation

```sh
npx expo install expo-camera
```

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
| `videoQuality` | Quality enum | — | Android, iOS, Web |
| `videoBitrate` | `number` | — | Android, iOS, Web |
| `videoStabilizationMode` | `'off' \| 'standard' \| 'cinematic' \| 'auto'` | `'auto'` | Android, iOS, Web |
| `barcodeScannerSettings` | `{ barcodeTypes: BarcodeType[] }` | — | Android, iOS, Web |
| `pictureSize` | `string` | — | Android, iOS, Web |
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

- `takePictureAsync(options?)` → `Promise<CameraCapturedPicture>`. Options: `quality` (0–1), `base64`, `exif`, `skipProcessing`, `imageType` (web), `shutterSound`, `pictureRef`, `mirror`, `additionalExif`. Result: `{ uri, width, height, base64?, exif?, format }`.
- `takePictureAsync({ pictureRef: true })` → `Promise<PictureRef>` (native image ref). `PictureRef` has `width`, `height`, `nativeRefType`, and `savePictureAsync({ quality?, base64?, metadata? })`.
- `recordAsync(options?)` → `Promise<{ uri: string } | undefined>`. Options: `maxDuration` (s), `maxFileSize` (bytes), `codec` (iOS: `'avc1' | 'hvc1' | 'jpeg' | 'apcn' | 'ap4h'`), `mirror`.
- `stopRecording()` — stops active recording.
- `toggleRecordingAsync()` — pause/resume recording (iOS 18+; gate behind `getSupportedFeatures()`).
- `pausePreview()` / `resumePreview()` — control preview state.
- `getAvailablePictureSizesAsync()` → `Promise<string[]>`.
- `getAvailableLensesAsync()` → `Promise<string[]>` (iOS).
- `getSupportedFeatures()` → `{ isModernBarcodeScannerAvailable, toggleRecordingAsyncAvailable }`.

#### Static methods

- `CameraView.isAvailableAsync()` → `Promise<boolean>` (Web).
- `CameraView.scanFromURLAsync(url, barcodeTypes?)` → `Promise<BarcodeScanningResult[]>`.
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

`BarcodeScanningResult`: `{ type, data, bounds, cornerPoints, extra? }` where `bounds = { origin: { x, y }, size: { width, height } }`. Corner point order differs by platform (Android/Web: top-left → bottom-left; iOS: bottom-left → top-right).

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

Manual (non-CNG): Android manifest needs `CAMERA` + `RECORD_AUDIO` and the `expo-camera/android/maven` repo; iOS Info.plist needs `NSCameraUsageDescription` + `NSMicrophoneUsageDescription`.

#### Minimal usage

```tsx
import { CameraView, useCameraPermissions } from 'expo-camera';
import { useState } from 'react';
import { Button, View } from 'react-native';

export default function App() {
  const [facing, setFacing] = useState('back');
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

### Installation

```sh
npx expo install expo-image
```

### Exported components

- **`Image`** — cross-platform image component (Android, iOS, tvOS, Web, Expo Go). Disk + memory caching, animated formats (GIF, WebP, APNG), BlurHash/ThumbHash placeholders. Backed by SDWebImage (iOS) and Glide (Android).
- **`ImageBackground`** — image as a background with children rendered on top. All `Image` props plus separate `style` (container) and `imageStyle` (background).

### Key props

- Source/loading: `source` (URL, local, `require()`, or array for responsive selection), `cachePolicy` (`'none' | 'disk' | 'memory' | 'memory-disk'`, default `'disk'`), `priority` (`'normal' | 'low' | 'high'`), `autoplay` (default `true`).
- Sizing/position: `contentFit` (`'cover' | 'contain' | 'fill' | 'none' | 'scale-down'`, default `'cover'`), `contentPosition`, `allowDownscaling` (default `true`).
- Placeholders: `placeholder` (string/number/ImageSource/array), `placeholderContentFit` (default `'scale-down'`).
- Transitions: `transition` (ms number or `ImageTransition` object; effects include `'cross-dissolve'`, `'flip-from-top'`, `'curl-up'`, SF Symbol animations).
- Visual: `blurRadius` (default `0`), `tintColor`, `preferHighDynamicRange` (iOS/tvOS 17+).
- Caching/perf: `recyclingKey` (list recycling / FlashList), `decodeFormat` (Android: `'argb' | 'rgb'`), `enforceEarlyResizing` (iOS).
- Accessibility/Web: `accessibilityLabel`/`alt`, `accessible`, `loading` (`'lazy' | 'eager'`, Web), `draggable` (Web), `responsivePolicy` (`'static' | 'initial' | 'live'`, Web).
- Advanced: `enableLiveTextInteraction` (iOS 16+), `sfEffect` (iOS 17+), `useAppleWebpCodec` (iOS, default `true`).
- Callbacks: `onLoadStart`, `onProgress({ loaded, total })`, `onLoad({ cacheType, source })`, `onLoadEnd`, `onDisplay`, `onError`.

### Hook

`useImage(source, options?, dependencies?)` → `ImageRef | null`. Options: `maxWidth`, `maxHeight`, `tintColor`, `onError(error, retry)`.

```jsx
const image = useImage('https://picsum.photos/1000/800', {
  maxWidth: 800,
  onError(error, retry) { console.error(error.message); }
});
```

### Static methods

- `Image.prefetch(urls, cachePolicy?)` / `Image.prefetch(urls, options?)` (options support `headers`) → `Promise<boolean>`
- `Image.loadAsync(source, options?)` → `Promise<ImageRef>`
- `Image.clearMemoryCache()`
- `Image.clearDiskCache()`
- `Image.getCachePathAsync(cacheKey)` → `Promise<string | null>`
- `Image.generateBlurhashAsync(source, numberOfComponents?)`
- `Image.generateThumbhashAsync(source)`
- `Image.configureCache(config)` (iOS only)

### Instance methods

`getAnimatableRef()`, `startAnimating()`, `stopAnimating()`, `lockResourceAsync()`, `unlockResourceAsync()`, `reloadAsync()`.

### Key types

`ImageSource`: `{ uri?, width?, height?, scale?, blurhash?, thumbhash?, headers?, cacheKey?, isAnimated?, webMaxViewportWidth? }`.
`ImageTransition`: `{ duration, effect, timing }`.
`ImageRef` (read-only): `width`, `height`, `scale`, `isAnimated`, `mediaType` (iOS), `nativeRefType`.

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

Bare RN: set `EXPO_IMAGE_DISABLE_LIBDAV1D=1` before `pod install`.

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

---

## 3. expo-image-picker

Source: https://docs.expo.dev/versions/v56.0.0/sdk/imagepicker/

### Installation

```sh
npx expo install expo-image-picker
```

### Permission hooks

- `useCameraPermissions(options?)` → `[PermissionResponse | null, requestPermission, getPermission]`
- `useMediaLibraryPermissions(options?)` → `[MediaLibraryPermissionResponse | null, requestPermission, getPermission]`. Options: `PermissionHookOptions<{ writeOnly: boolean }>`.

### Methods

- `launchImageLibraryAsync(options?)` → `Promise<ImagePickerResult>`
- `launchCameraAsync(options?)` → `Promise<ImagePickerResult>`
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

(Plugin also accepts `colors` / `dark.colors` for Android crop-tool theming.) iOS manual setup needs `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSMicrophoneUsageDescription`.

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

### Installation

```sh
npx expo install expo-image-manipulator
```

### APIs

**New context-based API (recommended):**
- `useImageManipulator(source)` — hook returning an `ImageManipulatorContext`
- `ImageManipulator.manipulate(source)` — returns an `ImageManipulatorContext`

**Deprecated:** `ImageManipulator.manipulateAsync(uri, actions, saveOptions)` — replaced by the new contextual, object-oriented API.

### `ImageManipulatorContext` (chainable)

- `rotate(degrees)` — clockwise for positive, counter-clockwise for negative.
- `flip(flipType)` — `'vertical'` or `'horizontal'` (`FlipType.Vertical` / `FlipType.Horizontal`).
- `resize(size)` — `{ width, height }`; a missing dimension is auto-computed to preserve aspect ratio.
- `crop(rect)` — `{ originX, originY, width, height }`.
- `extent(options)` — Web only; `{ size, offset, backgroundColor }`.
- `reset()` — revert to the original image.
- `renderAsync()` — awaits scheduled transforms, returns an `ImageRef`.

### `ImageRef`

- `saveAsync(options)` — persists to the cache directory and returns the saved image (`uri`, dimensions, optional `base64`).

### `SaveOptions`

- `format` — `SaveFormat.JPEG | SaveFormat.PNG | SaveFormat.WEBP`
- `compress` — `0.0`–`1.0`
- `base64` — boolean

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

### Installation

```sh
npx expo install expo-gl
```

### `GLView` component

Platforms: Android, iOS, Web, Expo Go. Extends `ViewProps`.

#### Props

| Prop | Type | Platform | Default | Description |
|------|------|----------|---------|-------------|
| `onContextCreate` | `(gl: ExpoWebGLRenderingContext) => void` | All | required | Called when the OpenGL ES context is created. |
| `msaaSamples` | `number` | iOS | `4` | Multisampling sample count; `0` disables. |
| `enableExperimentalWorkletSupport` | `boolean` | All | `false` | Enables Reanimated worklet-thread integration. |

#### Static methods

- `GLView.createContextAsync()` → `Promise<ExpoWebGLRenderingContext>` (headless context, no view).
- `GLView.destroyContextAsync(exgl)` → `Promise<boolean>`.
- `GLView.takeSnapshotAsync(exgl, options)` → `Promise<GLSnapshot>`.
- `GLView.getWorkletContext(contextId)` → `ExpoWebGLRenderingContext | undefined`.

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
- Legacy AR session methods (`startARSessionAsync`) are no longer part of the documented API surface.

---

## 6. expo-video-thumbnails

Source: https://docs.expo.dev/versions/v56.0.0/sdk/video-thumbnails/

### Installation

```sh
npx expo install expo-video-thumbnails
```

> **Deprecated:** This library is no longer receiving patches. Use `generateThumbnailsAsync` from **expo-video** instead.

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

## Deprecation summary

- **expo-image-picker**: `MediaTypeOptions` enum deprecated → use the `MediaType` string union; `videoExportPreset` deprecated by Apple.
- **expo-image**: `defaultSource`, `fadeDuration`, `loadingIndicatorSource`, `resizeMode` deprecated → `placeholder` / `transition` / `contentFit`+`contentPosition`.
- **expo-image-manipulator**: `manipulateAsync()` deprecated → context-based `manipulate()` / `useImageManipulator()`.
- **expo-video-thumbnails**: whole library deprecated → migrate to `expo-video`'s `generateThumbnailsAsync`.
- **expo-gl**: legacy AR session methods removed from the documented surface.
