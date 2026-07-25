# Video & Audio: Playback / Recording (Expo SDK 56)

Domain reference for the Expo SDK 56 knowledge base. Covers `expo-video`, `expo-audio` (playback + recording), the deprecated/removed `expo-av`, and `expo-screen-capture`.

## Read this first — what models get wrong from memory

**Do not emit `expo-av`.** It is not in the SDK. `import { Audio, Video } from 'expo-av'` will not resolve. There is no `av.mdx` under `docs/pages/versions/v55.0.0/`, `v56.0.0/` or `v57.0.0/`, and `expo-av` is absent from `packages/expo/bundledNativeModules.json` on both `origin/sdk-56` and `origin/sdk-57`. Use `expo-video` and `expo-audio`.

expo-av → expo-video / expo-audio symbol map:

| expo-av (do not use) | Replacement |
|------|-------------|
| `Audio.Sound.createAsync(source)` | `useAudioPlayer(source)` / `createAudioPlayer(source)` |
| `sound.playAsync()` / `pauseAsync()` | `player.play()` / `player.pause()` |
| `sound.setPositionAsync(positionMillis)` | `player.seekTo(seconds)` — **unit change: ms → seconds** |
| `status.positionMillis` / `status.durationMillis` | `status.currentTime` / `status.duration` — **seconds, not ms** |
| `setOnPlaybackStatusUpdate(cb)` | `useAudioPlayerStatus(player)` / `useEvent(player, ...)` |
| `Audio.Recording.createAsync()` | `useAudioRecorder(preset)` + `prepareToRecordAsync()` + `record()` |
| `playsInSilentModeIOS` | `playsInSilentMode` |
| `staysActiveInBackground` | `shouldPlayInBackground` |
| `interruptionModeIOS` / `interruptionModeAndroid` (numeric enums) | `interruptionMode` (string union, both platforms) |
| `<Video resizeMode={ResizeMode.CONTAIN} />` | `<VideoView contentFit="contain" />` |
| `<Video useNativeControls />` | `<VideoView nativeControls />` |

Other traps in this domain:

- **Seconds by default.** Positions and durations (`currentTime`, `duration`, `seekTo`, `seekBy`, `bufferedPosition`) are **seconds**; expo-av used milliseconds. This is the most common silent migration bug. The milliseconds hold-outs are the ones whose names say so: `RecorderState.durationMillis`, `AudioPlayerOptions.updateInterval` / `AudioPlaylistOptions.updateInterval`, `useAudioRecorderState(recorder, intervalMs)`, and `seekTo(seconds, toleranceMillisBefore?, toleranceMillisAfter?)`.
- **Lifecycle.** `useVideoPlayer` / `useAudioPlayer` / `useAudioPlaylist` auto-release on unmount. The `createVideoPlayer` / `createAudioPlayer` / `createAudioPlaylist` factories **do not** — you must call `release()` yourself (`AudioPlaylist` also exposes `destroy()`).
- **iOS `audioMixingMode` docs/native mismatch (56 *and* 57).** The TSDoc says `@default 'auto'`, but the iOS native default is `.doNotMix` — video playback silently interrupts other audio. Set it explicitly on iOS. Verified `var audioMixingMode: AudioMixingMode = .doNotMix` in `ios/VideoPlayer.swift` of both `expo-video@56.1.4` and `expo-video@57.0.2` (published tarballs). The flip to `.auto` (#47363) is on `main` only — **not in 57**; do not upgrade expecting it.
- Overlapping `VideoView`s with `contentFit="cover"` may render out of bounds; set `surfaceType="textureView"` on Android.
- Multiple `VideoView`s sharing one player is unsupported on Android (platform limitation).
- On Android the JS runtime is **paused** while a `VideoView` is in fullscreen, so `ref.exitFullscreen()` only works when called from a native listener.

### Version pins

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-video` | `~56.1.4` | `~57.0.2` |
| `expo-audio` | `~56.0.13` | `~57.0.3` |
| `expo-screen-capture` | `~56.0.4` | `~57.0.1` |
| `expo-av` | — (not in the SDK) | — (not in the SDK) |

Source: `packages/expo/bundledNativeModules.json` on `origin/sdk-56` / `origin/sdk-57` — this is what `expo install` resolves against. (Do **not** use `docs/public/static/schemas/v5x.0.0/native-modules.json`; those snapshots are stale and under-report the shipped patch levels.) Note `expo-video` moved to a **56.1.x** minor inside SDK 56 (56.1.0 added Android `exitFullscreen`) — pin `~56.1.4`, not `~56.0.x`. The `expo-*` pins are **not** flat `~57.0.0` in SDK 57.

---

## expo-video

Source: https://docs.expo.dev/versions/v56.0.0/sdk/video/

A modern, hook-based video playback library replacing the Video API of `expo-av`. Supported platforms: Android, iOS, tvOS, Web.

### Installation

```sh
npx expo install expo-video
```

For existing React Native apps, install the `expo` package as well.

### Configuration (config plugin)

```json
{
  "expo": {
    "plugins": [
      [
        "expo-video",
        {
          "supportsBackgroundPlayback": true,
          "supportsPictureInPicture": true
        }
      ]
    ]
  }
}
```

Configurable properties:
- `supportsBackgroundPlayback` (boolean, default undefined): enables background audio on iOS and a foreground service on Android.
- `supportsPictureInPicture` (boolean, default undefined): enables PiP mode on Android/iOS.

### Hook: `useVideoPlayer(source, setup?, playerBuilderOptions?)`

Creates a `VideoPlayer` with automatic lifecycle management.

```jsx
const player = useVideoPlayer(videoSource, player => {
  player.loop = true;
  player.play();
});
```

Parameters:
- `source`: `VideoSource` (string, number, null, or `VideoSourceObject`)
- `setup`: optional configuration callback
- `playerBuilderOptions`: Android-specific player builder options

### Component: `VideoView`

Platforms: Android, iOS, tvOS, Web.

```jsx
<VideoView
  player={player}
  style={styles.video}
  fullscreenOptions={{ enable: true }}
  allowsPictureInPicture
/>
```

Props:

| Prop | Type | Platform | Description |
|------|------|----------|-------------|
| `player` | `VideoPlayer \| null` | All | Player instance |
| `contentFit` | `'contain' \| 'cover' \| 'fill'` | All | Video scaling (default `'contain'`) |
| `nativeControls` | boolean | All | Show native controls (default `true`) |
| `fullscreenOptions` | `FullscreenOptions` | All | Fullscreen configuration |
| `allowsPictureInPicture` | boolean | Android, iOS, Web | Enable PiP |
| `onFirstFrameRender` | `() => void` | All | After first frame renders |
| `onFullscreenEnter` | `() => void` | All | Fullscreen entry |
| `onFullscreenExit` | `() => void` | All | Fullscreen exit |
| `onPictureInPictureStart` | `() => void` | Android, iOS, Web | PiP start |
| `onPictureInPictureStop` | `() => void` | Android, iOS, Web | PiP stop |
| `requiresLinearPlayback` | boolean | Android, iOS | Prevent user skip (default `false`) |
| `showsTimecodes` | boolean | iOS | Show timecodes (default `true`) |
| `startsPictureInPictureAutomatically` | boolean | Android 12+, iOS | Auto-PiP on background (default `false`) |
| `surfaceType` | `'surfaceView' \| 'textureView'` | Android | Rendering surface (default `'surfaceView'`) |
| `useAudioNodePlayback` | boolean | Web | Use Audio Nodes (default `false`) |
| `buttonOptions` | `ButtonOptions` | Android | Control button visibility |
| `contentPosition` | `{ dx: number, dy: number }` | iOS | Video offset in container |
| `crossOrigin` | `'anonymous' \| 'use-credentials'` | Web | CORS policy |
| `allowsVideoFrameAnalysis` | boolean | iOS 16.0+ | Enable Live Text (default `true`) |
| `playsInline` | boolean | Web | Play within element area |
| `useExoShutter` | boolean | Android | Use ExoPlayer shutter (default `false`) |

`VideoView` imperative methods (via ref):
- `enterFullscreen(): Promise<void>`
- `exitFullscreen(): Promise<void>` — **Android caveat:** the JS runtime is paused while the `VideoView` is in fullscreen, so this only works when called from a native listener. Use `useEventListener(player, 'playToEnd', () => ref.current?.exitFullscreen())`, not a `setTimeout` or a plain JS timer. Android support requires `expo-video >= 56.1.0` (before that it threw `MethodUnsupportedException`) — one more reason the SDK 56 pin is `~56.1.4`.
- `startPictureInPicture(): Promise<void>`
- `stopPictureInPicture(): Promise<void>`

### Class: `VideoPlayer`

Properties:

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `allowsExternalPlayback` | boolean | `true` | AirPlay (iOS) |
| `audioMixingMode` | `'mixWithOthers' \| 'duckOthers' \| 'auto' \| 'doNotMix'` | `'auto'` (TSDoc/Android) | **iOS in 56 *and* 57:** the native default is actually `doNotMix`, not `auto` — set it explicitly on iOS. The `.auto` fix (#47363) is not in either release line. |
| `audioTrack` | `AudioTrack \| null` | `null` | Current audio track |
| `availableAudioTracks` | `AudioTrack[]` | — | |
| `availableSubtitleTracks` | `SubtitleTrack[]` | — | |
| `availableVideoTracks` | `VideoTrack[]` | — | |
| `bufferedPosition` | number (read-only) | — | seconds |
| `bufferOptions` | `BufferOptions` | — | |
| `currentLiveTimestamp` | `number \| null` (read-only) | — | Live server timestamp |
| `currentOffsetFromLive` | `number \| null` (read-only) | — | Live latency |
| `currentTime` | number | — | seconds |
| `duration` | number (read-only) | — | seconds |
| `isExternalPlaybackActive` | boolean (read-only) | — | AirPlay active (iOS) |
| `isLive` | boolean (read-only) | — | |
| `keepScreenOnWhilePlaying` | boolean | `true` | |
| `loop` | boolean | `false` | |
| `muted` | boolean | `false` | |
| `playbackRate` | number | `1.0` | 0–16 |
| `playing` | boolean (read-only) | — | |
| `preservesPitch` | boolean | `true` | |
| `scrubbingModeOptions` | `ScrubbingModeOptions` | — | |
| `seekTolerance` | `SeekTolerance` | — | |
| `showNowPlayingNotification` | boolean | `false` | OS notification |
| `status` | `VideoPlayerStatus` | — | |
| `staysActiveInBackground` | boolean | `false` | |
| `subtitleTrack` | `SubtitleTrack \| null` | `null` | |
| `targetOffsetFromLive` | number | — | seconds |
| `timeUpdateEventInterval` | number | `0` | seconds |
| `videoTrack` | `VideoTrack \| null` (read-only) | `null` | |
| `volume` | number | `1.0` | 0–1 |

Methods:
- `pause(): void`
- `play(): void`
- `replay(): void`
- `seekBy(seconds: number): void`
- `replace(source: VideoSource, disableWarning?: boolean): void`
- `replaceAsync(source: VideoSource): Promise<void>`
- `generateThumbnailsAsync(times: number | number[], options?: VideoThumbnailOptions): Promise<VideoThumbnail[]>`
- `release(): void` — inherited from `SharedObject`. Required for players made with `createVideoPlayer`; `useVideoPlayer` releases automatically on unmount.

### Module static methods

- `createVideoPlayer(source, playerBuilderOptions?): VideoPlayer` — **you own the lifecycle; call `player.release()` when done.**
- `isPictureInPictureSupported(): boolean`
- `getCurrentVideoCacheSize(): number`
- `setVideoCacheSizeAsync(sizeBytes: number): Promise<void>`
- `clearVideoCacheAsync(): Promise<void>`

### Events (VideoPlayer)

All 15 events and their payload shapes (source: `packages/expo-video/src/VideoPlayerEvents.types.ts`). Every `old*` field is optional.

| Event | Payload |
|-------|---------|
| `statusChange` | `{ status: VideoPlayerStatus; oldStatus?; error?: PlayerError }` |
| `playingChange` | `{ isPlaying: boolean; oldIsPlaying? }` |
| `playbackRateChange` | `{ playbackRate: number; oldPlaybackRate? }` |
| `volumeChange` | `{ volume: number; oldVolume? }` |
| `mutedChange` | `{ muted: boolean; oldMuted? }` |
| `playToEnd` | *(no payload)* |
| `timeUpdate` | `{ currentTime; currentLiveTimestamp: number \| null; currentOffsetFromLive: number \| null; bufferedPosition }` |
| `sourceChange` | `{ source: VideoSource; oldSource? }` |
| `sourceLoad` | `{ videoSource: VideoSource \| null; duration; availableVideoTracks; availableSubtitleTracks; availableAudioTracks }` |
| `audioTrackChange` | `{ audioTrack: AudioTrack \| null; oldAudioTrack? }` |
| `subtitleTrackChange` | `{ subtitleTrack: SubtitleTrack \| null; oldSubtitleTrack? }` |
| `videoTrackChange` | `{ videoTrack: VideoTrack \| null; oldVideoTrack? }` |
| `availableAudioTracksChange` | `{ availableAudioTracks: AudioTrack[]; oldAvailableAudioTracks? }` |
| `availableSubtitleTracksChange` | `{ availableSubtitleTracks: SubtitleTrack[]; oldAvailableSubtitleTracks? }` |
| `isExternalPlaybackActiveChange` (iOS) | `{ isExternalPlaybackActive: boolean; oldIsExternalPlaybackActive? }` |

Listening patterns:

```jsx
// useEvent — re-renders with current value
const { isPlaying } = useEvent(player, 'playingChange', { isPlaying: player.playing });

// useEventListener — side-effect callback
useEventListener(player, 'statusChange', ({ status, error }) => {
  console.log('Status:', status);
});

// player.addListener — manual subscription
useEffect(() => {
  const subscription = player.addListener('statusChange', ({ status, error }) => {
    setPlayerStatus(status);
  });
  return () => subscription.remove();
}, []);
```

### Types

```typescript
type VideoSource = string | number | null | VideoSourceObject;

interface VideoSourceObject {
  uri?: string;
  assetId?: number;
  contentType?: 'auto' | 'progressive' | 'hls' | 'dash' | 'smoothStreaming';
  drm?: DRMOptions;
  headers?: Record<string, string>;
  metadata?: VideoMetadata;
  useCaching?: boolean;
}

interface DRMOptions {
  type: 'clearkey' | 'fairplay' | 'playready' | 'widevine';
  licenseServer: string;
  headers?: Record<string, string>;
  certificateUrl?: string;       // iOS
  base64CertificateData?: string; // iOS
  contentId?: string;            // iOS
  multiKey?: boolean;            // Android
}

interface VideoMetadata { title?: string; artist?: string; artwork?: string; }

interface BufferOptions {
  minBufferForPlayback?: number;             // seconds, Android default 2
  preferredForwardBufferDuration?: number;   // Android 20, iOS 0
  maxBufferBytes?: number | null;            // Android, default 0 = player picks the buffer size automatically
  waitsToMinimizeStalling?: boolean;         // iOS default true
  prioritizeTimeOverSizeThreshold?: boolean; // Android, default false
}

interface AudioTrack { label: string; language: string; name?: string; id?: string; isDefault?: boolean; autoSelect?: boolean; }
interface SubtitleTrack { label: string; language: string; name?: string; id?: string; isDefault?: boolean; autoSelect?: boolean; }

interface VideoTrack {
  id: string;
  url: string | null;
  mimeType: string | null;
  size: VideoSize;
  frameRate: number | null;
  bitrate: number | null;
  averageBitrate: number | null;
  peakBitrate: number | null;
  videoRange: 'sdr' | 'hlg' | 'pq';
  isSupported: boolean; // Android
}
interface VideoSize { width: number; height: number; }

interface SeekTolerance { toleranceBefore?: number; toleranceAfter?: number; } // seconds, both default 0

interface ScrubbingModeOptions {
  scrubbingModeEnabled?: boolean;         // Android, iOS — default FALSE; gates all the others
  increaseCodecOperatingRate?: boolean;   // Android, default true
  allowSkippingMediaCodecFlush?: boolean; // Android, default true
  useDecodeOnlyFlag?: boolean;            // Android, default true
  enableDynamicScheduling?: boolean;      // Android, default true
}
// On Android, playback is suppressed while scrubbingModeEnabled is true — set it back to false when the gesture ends.

interface ButtonOptions {           // Android
  showPlayPause?: boolean;          // default true
  showSeekBackward?: boolean;       // default true
  showSeekForward?: boolean;        // default true
  showSettings?: boolean;           // default true
  showSubtitles?: boolean | null;   // default undefined
  showNext?: boolean;               // default false
  showPrevious?: boolean;           // default false
  showBottomBar?: boolean;          // default true; always visible in fullscreen regardless
}

// FullscreenOptions has FOUR fields, not one. typedoc does not expand it in the
// generated docs data, so it is commonly mis-remembered as `{ enable }` alone.
type FullscreenOptions = {
  enable: boolean;                                          // default true; false hides the fullscreen button
  orientation?: FullscreenOrientation;                      // default 'default'; Android, iOS
  autoExitOnRotate?: boolean;                               // default false; Android, iOS; no-op when orientation === 'default'
  keepFullscreenOnPiPStop?: KeepFullscreenOnPiPStopBehavior; // default 'autoEnter'; iOS
};

type FullscreenOrientation =
  | 'default' | 'portrait' | 'portraitUp' | 'portraitDown'
  | 'landscape' | 'landscapeLeft' | 'landscapeRight';

type KeepFullscreenOnPiPStopBehavior = 'always' | 'autoEnter' | 'never';

type VideoPlayerStatus = 'idle' | 'loading' | 'readyToPlay' | 'error';
type VideoContentFit = 'contain' | 'cover' | 'fill';

interface VideoThumbnailOptions { maxWidth?: number; maxHeight?: number; }

interface PlayerBuilderOptions {
  seekForwardIncrement?: number;  // seconds
  seekBackwardIncrement?: number; // seconds
}
```

### Component: `VideoAirPlayButton` (iOS)

```jsx
<VideoAirPlayButton
  tint="blue"
  activeTint="red"
  prioritizeVideoDevices
  onBeginPresentingRoutes={() => {}}
  onEndPresentingRoutes={() => {}}
/>
```

Props: `tint`, `activeTint`, `prioritizeVideoDevices` (default `true`), `onBeginPresentingRoutes`, `onEndPresentingRoutes`.

### Video caching (Android, iOS)

```typescript
const videoSource: VideoSource = {
  uri: 'https://example.com/video.mp4',
  useCaching: true,
};

await Video.setVideoCacheSizeAsync(1024 * 1024 * 1024); // 1GB
const size = Video.getCurrentVideoCacheSize();
await Video.clearVideoCacheAsync();
```

Caching is unavailable for HLS sources (iOS) and unsupported for DRM-protected content.

### Known issues / workarounds

- Overlapping `VideoView`s with `contentFit="cover"` may render out of bounds; set `surfaceType="textureView"` on Android.
- Multiple `VideoView`s sharing one player is unsupported on Android (platform limitation).

### Intercepting native asset loading (iOS, advanced)

`expo-video` exposes a native iOS extension point for taking over how the underlying `AVURLAsset` is created — URL rewriting, a custom `AVAssetResourceLoaderDelegate`, a local proxy server, DASH→HLS translation. Requires a custom/local native module, so it is **not available in Expo Go**.

- Conform a Swift class to `VideoAssetTransportProvider`: `identifier` (stable name), `priority` (higher wins), `makeLoadPlan(for: VideoAssetSourceDescriptor) -> VideoAssetLoadPlan?` (return `nil` to ignore a source).
- Register it in your module's `OnCreate` block via `VideoAssetTransportRegistry.registerProvider(...)`; unregister in `OnDestroy`.
- `VideoAssetLoadPlan` fields: `assetURL`, `assetOptions`, `reportedContentTypeHint`, `resourceLoaderDelegate`, `resourceLoaderQueue`, `prepareAsset`, `retainedObjects`, `attachErrorHandler`, `onAssetDeinit`.

Full walkthrough + Swift examples: `docs/pages/versions/v56.0.0/sdk/video.mdx` § "Intercepting native asset loading" (unchanged in v57).

### Full example: basic playback with controls

```jsx
import { useEvent } from 'expo';
import { useVideoPlayer, VideoView } from 'expo-video';
import { StyleSheet, View, Button } from 'react-native';

const videoSource = 'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4';

export default function VideoScreen() {
  const player = useVideoPlayer(videoSource, player => {
    player.loop = true;
    player.play();
  });

  const { isPlaying } = useEvent(player, 'playingChange', { isPlaying: player.playing });

  return (
    <View style={styles.contentContainer}>
      <VideoView style={styles.video} player={player} fullscreenOptions={{ enable: true }} allowsPictureInPicture />
      <View style={styles.controlsContainer}>
        <Button title={isPlaying ? 'Pause' : 'Play'} onPress={() => isPlaying ? player.pause() : player.play()} />
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  contentContainer: { flex: 1, padding: 10, alignItems: 'center', justifyContent: 'center', paddingHorizontal: 50 },
  video: { width: 350, height: 275 },
  controlsContainer: { padding: 10 },
});
```

---

## expo-audio

Source: https://docs.expo.dev/versions/v56.0.0/sdk/audio/

Modern, hook-based audio playback and recording library replacing the Audio API of `expo-av`. Supported platforms: Android, iOS, tvOS, Web, Expo Go.

> `useAudioStream` (real-time PCM microphone capture) is only summarized here — the full `AudioStream` class and its `AudioStreamOptions` / `AudioStreamResult` / `AudioStreamBuffer` / `AudioStreamEncoding` types live in `references/07-media-device-apis.md`.

### Installation

```sh
npx expo install expo-audio
```

### Configuration (config plugin)

```json
{
  "expo": {
    "plugins": [
      [
        "expo-audio",
        {
          "microphonePermission": "Allow $(PRODUCT_NAME) to access your microphone.",
          "enableBackgroundPlayback": true,
          "enableBackgroundRecording": false
        }
      ]
    ]
  }
}
```

Configurable properties (source: `packages/expo-audio/plugin/src/withAudio.ts`):
- `microphonePermission?: string | false` (iOS) — `NSMicrophoneUsageDescription` text; pass `false` to omit the key entirely. Default `"Allow $(PRODUCT_NAME) to access your microphone"`.
- `recordAudioAndroid?: boolean` (Android `RECORD_AUDIO`) — default `true`.
- `enableBackgroundRecording?: boolean` — default `false`.
- `enableBackgroundPlayback?: boolean` — default `true`.

### Hooks

- `useAudioPlayer(source?, options?)` → `AudioPlayer`. Auto lifecycle management.
- `useAudioPlayerStatus(player)` → `AudioStatus`. Real-time playback status.
- `useAudioRecorder(options, statusListener?)` → `AudioRecorder`.
- `useAudioRecorderState(recorder, interval?)` → `RecorderState`. Polls at `interval` ms (default 500).
- `useAudioSampleListener(player, listener)` — sampling for visualization/analysis.
- `useAudioPlaylist(options?)` → `AudioPlaylist`.
- `useAudioPlaylistStatus(playlist)` → `AudioPlaylistStatus`.
- `useAudioStream(options?)` → `AudioStreamResult` (real-time PCM mic capture; SDK 56 addition — see streaming domain).

```jsx
const player = useAudioPlayer(require('./audio.mp3'));
player.play();

const status = useAudioPlayerStatus(player); // status.playing, status.currentTime, status.duration

const recorder = useAudioRecorder(RecordingPresets.HIGH_QUALITY);
await recorder.prepareToRecordAsync();
recorder.record();

const state = useAudioRecorderState(recorder); // state.isRecording, state.durationMillis, state.canRecord
```

### Class: `AudioPlayer`

Properties: `currentTime`, `duration`, `playing`, `paused`, `muted`, `loop`, `volume` (0.0–1.0), `playbackRate` (0.1–2.0, platform-dependent), `isLoaded`, `isBuffering`, `isAudioSamplingSupported`, `shouldCorrectPitch`, `id`.

Methods:
- `play()`, `pause()`
- `seekTo(seconds, toleranceMillisBefore?, toleranceMillisAfter?)`
- `replace(source)`
- `remove()`
- `setPlaybackRate(rate, pitchCorrectionQuality?)`
- `setActiveForLockScreen(active: boolean, metadata?: AudioMetadata, options?: AudioLockScreenOptions)`
- `updateLockScreenMetadata(metadata)`
- `clearLockScreenControls()`
- `release(): void` — inherited from `SharedObject`. Required for players made with `createAudioPlayer`; `useAudioPlayer` releases automatically on unmount.

### Class: `AudioRecorder`

Properties: `currentTime` (seconds), `isRecording`, `uri` (`string | null`), `id`.

Methods:
- `prepareToRecordAsync(options?)`
- `record(options?)`
- `recordForDuration(seconds)` — **deprecated**, use `record({ forDuration: seconds })`
- `startRecordingAtTime(seconds)` — **deprecated**, use `record({ atTime: seconds })` (iOS only)
- `pause()`, `stop()`
- `getStatus()` → `RecorderState`
- `getAvailableInputs()`, `getCurrentInput()`, `setInput(inputUid)`

### Class: `AudioPlaylist`

Obtained from `useAudioPlaylist(options?)` (auto-released) or `createAudioPlaylist(options?)` (you release it).

Properties: `id`, `currentIndex` (read-only), `trackCount` (read-only), `sources: AudioSourceInfo[]` (read-only), `playing`, `muted`, `isLoaded`, `isBuffering`, `currentTime` (seconds), `duration` (seconds), `volume` (0.0–1.0), `playbackRate`, `loop: AudioPlaylistLoopMode`.

Methods:
- `play()`, `pause()`
- `next()`, `previous()` — wrap around only when `loop === 'all'`; no-ops at the ends when `loop === 'none'`
- `skipTo(index)`
- `seekTo(seconds): Promise<void>`
- `add(source)`, `insert(source, index)`, `remove(index)`, `clear()`
- `destroy()` — frees native resources; `release()` is also available via `SharedObject`

```ts
interface AudioPlaylistOptions {
  sources?: AudioSource[];                        // default []
  updateInterval?: number;                        // ms, default 500
  loop?: AudioPlaylistLoopMode;                   // default 'none'
  crossOrigin?: 'anonymous' | 'use-credentials';  // web only, default undefined
}

interface AudioPlaylistStatus {   // returned by useAudioPlaylistStatus(playlist)
  id: string;
  currentIndex: number;
  trackCount: number;
  currentTime: number;   // seconds
  duration: number;      // seconds
  playing: boolean;
  isBuffering: boolean;
  isLoaded: boolean;
  playbackRate: number;
  muted: boolean;
  volume: number;
  loop: AudioPlaylistLoopMode;
  didJustFinish: boolean;
}

interface AudioSourceInfo { uri?: string; name?: string; }
```

> `AudioPlaylist` has **no** lock-screen methods — only `AudioPlayer` does. Still true in SDK 57: `setActiveForLockScreen` appears exactly once in the published `expo-audio@57.0.3` type surface (`build/AudioModule.types.d.ts`, on `AudioPlayer`), same as `56.0.13`. Playlist lock-screen controls (#46020) are `main`-only.

### Module static methods (`AudioModule` / `Audio.*`)

- `setAudioModeAsync(mode)` — global audio behavior (see properties below).
- `setIsAudioActiveAsync(active)` — enable/disable audio subsystem.
- `requestRecordingPermissionsAsync()`, `getRecordingPermissionsAsync()`
- `requestNotificationPermissionsAsync()` (Android)
- `preload(source, options?: PreloadOptions)`, `getPreloadedSources()`, `clearPreloadedSource(source)`, `clearAllPreloadedSources()`
  - `PreloadOptions { preferredForwardBufferDuration?: number }` — seconds, default `10`, Android + iOS. On iOS maps to `AVPlayerItem.preferredForwardBufferDuration` (`0` = let the system decide); no-op on web.
- `createAudioPlayer(source?, options?)` — manual creation; **you must call `release()`**.
- `createAudioPlaylist(options?: AudioPlaylistOptions)` — manual creation; **you must call `release()` / `destroy()`**.

`setAudioModeAsync(mode)` accepts a partial `AudioMode`:
- `playsInSilentMode` (boolean, default `true`)
- `shouldPlayInBackground` (boolean, default `false`)
- `allowsRecording` (boolean, iOS, default `false`)
- `allowsBackgroundRecording` (boolean, default `false`)
- `interruptionMode`: `'mixWithOthers' | 'doNotMix' | 'duckOthers'` (default `'mixWithOthers'`). Must be `'doNotMix'` when using `setActiveForLockScreen`.
- `interruptionModeAndroid` (`InterruptionModeAndroid`, Android) — **deprecated**; `InterruptionModeAndroid` is just an alias of `InterruptionMode`. Use `interruptionMode`, which now works on both platforms.
- `shouldRouteThroughEarpiece` (boolean, **all platforms**, default `false`) — not iOS-only. On iOS it only takes effect when `allowsRecording: true` (i.e. the session category is `.playAndRecord`); otherwise audio routes through the speaker.

```jsx
await setAudioModeAsync({
  playsInSilentMode: true,
  shouldPlayInBackground: true,
  interruptionMode: 'doNotMix'
});
```

### Recording presets

```ts
RecordingPresets.HIGH_QUALITY = {
  extension: '.m4a',
  sampleRate: 44100,
  numberOfChannels: 2,
  bitRate: 128000,
  android: { outputFormat: 'mpeg4', audioEncoder: 'aac' },
  ios: {
    outputFormat: IOSOutputFormat.MPEG4AAC,
    audioQuality: AudioQuality.MAX,
    linearPCMBitDepth: 16,
    linearPCMIsBigEndian: false,
    linearPCMIsFloat: false
  },
  web: { mimeType: 'audio/webm', bitsPerSecond: 128000 }
};

RecordingPresets.LOW_QUALITY = {
  extension: '.m4a',
  sampleRate: 44100,
  numberOfChannels: 2,
  bitRate: 64000,
  android: { extension: '.3gp', outputFormat: '3gp', audioEncoder: 'amr_nb' },
  ios: {
    audioQuality: AudioQuality.MIN,
    outputFormat: IOSOutputFormat.MPEG4AAC,
    linearPCMBitDepth: 16,
    linearPCMIsBigEndian: false,
    linearPCMIsFloat: false
  },
  web: { mimeType: 'audio/webm', bitsPerSecond: 128000 }
};
```

### Types

```ts
interface AudioStatus {          // 18 fields
  id: string;
  currentTime: number;           // seconds
  duration: number;              // seconds
  playing: boolean;
  isLoaded: boolean;
  isBuffering: boolean;
  error: string | null;
  didJustFinish: boolean;
  loop: boolean;
  mute: boolean;                 // note: `mute`, not `muted`
  playbackRate: number;
  shouldCorrectPitch: boolean;   // default true
  timeControlStatus: string;
  playbackState: string;
  reasonForWaitingToPlay: string;
  isLive: boolean;
  currentOffsetFromLive: number | null;
  mediaServicesDidReset?: boolean; // iOS
}

interface RecorderState {
  isRecording: boolean;
  canRecord: boolean;
  durationMillis: number;
  metering?: number;
  url: string | null;
  mediaServicesDidReset: boolean;
}

interface AudioPlayerOptions {
  updateInterval?: number;  // ms, default 500
  downloadFirst?: boolean;
  preferredForwardBufferDuration?: number;
  keepAudioSessionActive?: boolean;
  crossOrigin?: 'anonymous' | 'use-credentials';
}

interface RecordingOptions {
  extension: string;
  sampleRate: number;
  numberOfChannels: number;
  bitRate: number;
  isMeteringEnabled?: boolean;
  directory?: RecordingDirectory;  // needs expo-audio >= 56.0.12; see patch-drift note
  android?: RecordingOptionsAndroid;
  ios?: RecordingOptionsIos;
  web?: RecordingOptionsWeb;
}

type RecordingDirectory = 'cache' | 'document';  // default 'cache'; Android + iOS

interface RecordingStartOptions {
  forDuration?: number; // seconds
  atTime?: number;      // seconds (iOS only)
}

// AudioSource: string URI | require() asset number | null | object:
interface AudioSourceObject {
  uri?: string;
  assetId?: number;
  headers?: Record<string, string>;
  name?: string;
}

interface AudioMetadata {
  title?: string;
  artist?: string;
  albumTitle?: string;
  artworkUrl?: string;
}

interface AudioLockScreenOptions {   // SDK 56: 3 fields
  showSeekForward?: boolean;
  showSeekBackward?: boolean;
  isLiveStream?: boolean;            // hides duration + scrub bar, disables seek
}
```

### Enums

- `AudioQuality`: `MIN` (0), `LOW` (32), `MEDIUM` (64), `HIGH` (96), `MAX` (127)
- `IOSOutputFormat`: MPEG4AAC, LINEARPCM, APPLELOSSLESS, etc.
- `AndroidAudioEncoder`: `'default'`, `'amr_nb'`, `'amr_wb'`, `'aac'`, `'he_aac'`, `'aac_eld'`
- `AndroidOutputFormat`: `'default'`, `'3gp'`, `'mpeg4'`, `'amrnb'`, `'amrwb'`, `'aac_adts'`, `'mpeg2ts'`, `'webm'`
- `RecordingSource` (Android, 8 values): `'camcorder'`, `'default'`, `'mic'`, `'remote_submix'`, `'unprocessed'`, `'voice_communication'`, `'voice_performance'`, `'voice_recognition'`
- `InterruptionMode`: `'mixWithOthers'`, `'doNotMix'`, `'duckOthers'`
- `AudioPlaylistLoopMode`: `'none'`, `'single'`, `'all'`
- `PitchCorrectionQuality` (iOS): `'low'`, `'medium'`, `'high'`

### Examples

Playback:

```jsx
import { useAudioPlayer } from 'expo-audio';
import { Button, View } from 'react-native';

export default function App() {
  const player = useAudioPlayer(require('./audio.mp3'));
  return (
    <View>
      <Button title="Play" onPress={() => player.play()} />
      <Button title="Pause" onPress={() => player.pause()} />
      <Button title="Seek to 0" onPress={() => player.seekTo(0)} />
    </View>
  );
}
```

Recording:

```jsx
import { useAudioRecorder, RecordingPresets } from 'expo-audio';
import { Button, View } from 'react-native';

export default function App() {
  const recorder = useAudioRecorder(RecordingPresets.HIGH_QUALITY);

  const startRecording = async () => {
    await recorder.prepareToRecordAsync();
    recorder.record();
  };

  const stopRecording = async () => {
    await recorder.stop();
    console.log('Recording saved at:', recorder.uri);
  };

  return (
    <View>
      <Button title="Record" onPress={startRecording} />
      <Button title="Stop" onPress={stopRecording} />
    </View>
  );
}
```

Background playback + lock screen:

```jsx
import { useAudioPlayer, setAudioModeAsync } from 'expo-audio';
import { useEffect } from 'react';
import { Button } from 'react-native';

export default function App() {
  const player = useAudioPlayer(require('./audio.mp3'));

  useEffect(() => {
    setAudioModeAsync({
      playsInSilentMode: true,
      shouldPlayInBackground: true,
      interruptionMode: 'doNotMix'
    });
  }, []);

  const play = () => {
    player.setActiveForLockScreen(true, { title: 'Track Title', artist: 'Artist Name' });
    player.play();
  };

  return <Button title="Play" onPress={play} />;
}
```

Audio visualization:

```jsx
import { useAudioPlayer, useAudioSampleListener } from 'expo-audio';

export default function App() {
  const player = useAudioPlayer(require('./audio.mp3'));
  useAudioSampleListener(player, (sample) => {
    const frames = sample.channels[0].frames;
    // use frames for visualization
  });
  return <Button title="Play" onPress={() => player.play()} />;
}
```

### Platform notes

- Audio stops automatically on headphone/Bluetooth device disconnection.
- Lock screen controls require `interruptionMode: 'doNotMix'`.
- Background recording significantly impacts battery life.
- Web requires a secure (HTTPS) context for microphone access.
- Android requires explicit `setActiveForLockScreen()` for sustained background playback (≈3-minute OS limit otherwise).

### What arrived in SDK 56 (baseline — still present in 57)

Per the SDK 56 changelog (https://expo.dev/changelog/sdk-56):
- `useAudioStream` hook for real-time microphone buffer access.
- Live-stream support: iOS `isLiveStream` lock-screen option; Android `playsInSilentMode` support; `AudioStatus` gained `isLive`, `currentOffsetFromLive`, and `error`.

### SDK 56 patch drift — things that arrived *inside* the 56 line

The SDK 56 line is still being patched. These are **not** reasons to upgrade to 57; they only need a patch bump on the line you are already on. Minimum versions from the release-branch CHANGELOGs.

| Item | Package | Needs |
|------|---------|-------|
| Android `VideoView` ref `exitFullscreen()` (was `MethodUnsupportedException`) — #41836 | `expo-video` | `>= 56.1.0` |
| `availableVideoTracks` deduplicated for HLS sources with multiple audio renditions — #46691. Track-picker UIs built against an earlier 56 patch will render fewer rows. | `expo-video` | `>= 56.1.3` |
| iOS failed players recovered instead of leaving a broken playback placeholder — #46681 | `expo-video` | `>= 56.1.3` |
| iOS shared remote command center commands re-enabled so lock-screen controls keep working after `expo-audio` playback — #46753 | `expo-video` | `>= 56.1.4` |
| Android `RemoteServiceException` crash when the system starts `AudioControlsService` via `startForegroundService()` — #46147 | `expo-audio` | `>= 56.0.10` |
| `RecordingOptions.directory?: 'cache' \| 'document'` (default `'cache'`; Android + iOS). `'document'` keeps recordings out of reach of OS storage-pressure eviction — #46189 | `expo-audio` | `>= 56.0.12` |
| Android recording crash in apps wrapped with Microsoft Intune — #47005 | `expo-audio` | `>= 56.0.13` |

---

## expo-av (deprecated / removed)

Source: `/versions/v56.0.0/sdk/av/` and `/versions/latest/sdk/av/` both return **HTTP 404** in SDK 56 — `expo-av` is no longer documented or shipped. Deprecation notice captured from the last release that documented it (SDK 54): https://docs.expo.dev/versions/v54.0.0/sdk/av/

Deprecation notice (verbatim, SDK 54 docs):

> **Deprecated:** The `Video` and `Audio` APIs from `expo-av` have now been deprecated and replaced by improved versions in `expo-video` and `expo-audio`. We recommend using those libraries instead. `expo-av` is not receiving patches and will be removed in SDK 55.

Timeline: deprecated in SDK 53, last shipped in SDK 54, removed from the SDK in SDK 55; doc pages 404 from SDK 55 onward (v55, v56 and v57 all lack `sdk/av.mdx`).

Migration path: video → **expo-video** (`useVideoPlayer` + `VideoView`); audio playback and recording → **expo-audio** (`useAudioPlayer`, `useAudioRecorder`). Symbol-by-symbol mapping is in the [table at the top of this file](#read-this-first--what-models-get-wrong-from-memory) — mind the milliseconds → seconds unit change. Recommended order: migrate video first, then audio.

---

## expo-screen-capture

Source: https://docs.expo.dev/versions/v56.0.0/sdk/screen-capture/

Protects app screens from being captured/recorded and notifies on screenshots. Use cases: protecting sensitive data (passwords, credit cards) and preventing paid content recording. Supported platforms: Android, iOS, Expo Go.

### Installation

```sh
npx expo install expo-screen-capture
```

### Hooks

- `usePreventScreenCapture(key?)` — prevents capture while the component is mounted. Optional `key` (string) to manage multiple instances. Returns `void`.
- `useScreenshotListener(listener)` — runs `listener` on screenshot detection while mounted. Returns `void`.
- `usePermissions(options?)` — manages screenshot-detection permissions. Returns `[PermissionResponse | null, RequestPermissionMethod, GetPermissionMethod]`.

### Methods

- `preventScreenCaptureAsync(key?)` — block screenshots/recordings until `allowScreenCaptureAsync()` is called. Optional key avoids conflicts.
- `allowScreenCaptureAsync(key?)` — re-enable capture; requires matching key if one was set.
- `addScreenshotListener(listener)` — fires when the user screenshots while the app is foregrounded. Returns `EventSubscription`.
- `removeScreenshotListener(subscription)` — unregister a screenshot listener.
- `getPermissionsAsync()` — check screenshot-detection permissions (iOS always grants; Android varies by version).
- `requestPermissionsAsync()` — request screenshot-detection permissions.
- `isAvailableAsync()` — whether the Screen Capture API is available on the device.
- `enableAppSwitcherProtectionAsync(blurIntensity?)` (iOS) — apply a blur overlay in the app switcher/background (0.0–1.0, default 0.5).
- `disableAppSwitcherProtectionAsync()` (iOS) — remove the app-switcher blur.

### Types

- `PermissionResponse`: `status` (`PermissionStatus`), `granted` (boolean), `canAskAgain` (boolean), `expires` (`PermissionExpiration`).
- `PermissionStatus`: `GRANTED`, `DENIED`, `UNDETERMINED`.
- `EventSubscription`: provides `remove()`.

### Platform notes

- **Blocking capture** (`preventScreenCaptureAsync` / `usePreventScreenCapture`) never requires a permission on any Android version.
- **The screenshot callback** (`addScreenshotListener` / `useScreenshotListener`) works permission-free on Android 14+. On Android 13 and lower it requires `READ_MEDIA_IMAGES`, declared via `android.permissions` in app config.
- Google Play warning: `READ_MEDIA_IMAGES` may only be declared by apps that genuinely need broad photo access (Play Photo and Video Permissions policy) — do not add it just for screenshot detection if you ship to Play.
- Testing: on Android Emulator run `adb shell input keyevent 120` to trigger a screenshot; on iOS Simulator use **Device → Trigger Screenshot**.

---

## Source URLs

- expo-video: https://docs.expo.dev/versions/v56.0.0/sdk/video/ · https://docs.expo.dev/versions/v57.0.0/sdk/video/
- expo-audio: https://docs.expo.dev/versions/v56.0.0/sdk/audio/ · https://docs.expo.dev/versions/v57.0.0/sdk/audio/
- expo-av (404 in SDK 55/56/57; deprecation notice from SDK 54 archive): https://docs.expo.dev/versions/v54.0.0/sdk/av/
- expo-screen-capture: https://docs.expo.dev/versions/v56.0.0/sdk/screen-capture/ · https://docs.expo.dev/versions/v57.0.0/sdk/screen-capture/
- SDK 56 changelog (feature narrative): https://expo.dev/changelog/sdk-56
- Authoritative for pins: `packages/expo/bundledNativeModules.json` on `origin/sdk-56` / `origin/sdk-57`. Authoritative for API surface: the published npm tarballs (`npm pack expo-video@57.0.2`).

---

## SDK 57 delta

**There is no JS API delta in this domain.** Diffing the published tarballs, `expo-video@56.1.4/build` vs `expo-video@57.0.2/build` and `expo-audio@56.0.13/build` vs `expo-audio@57.0.3/build` are **byte-for-byte identical** (`diff -r -q` reports no differing files). Everything in the SDK 56 body above applies verbatim to SDK 57. Nothing in this domain is a reason to upgrade — and nothing in it will break when you do.

> How this was verified: `npm pack expo-video@57.0.2 expo-video@56.1.4 expo-audio@57.0.3 expo-audio@56.0.13`, then recursive diff of the `package/` trees. Do not use `docs/public/static/data/v57.0.0/expo-*.json` or the `native-modules.json` schemas — both are stale snapshots. Do not use `packages/*/CHANGELOG.md` on `main`: `main` is SDK 58 in progress, and its `## Unpublished` section is the single biggest source of false "new in 57" claims for this domain.

### What actually changed (native only)

Outside `build/`, the only differing sources between the two tarball pairs:

- **`expo-video` 56.1.4 → 57.0.2**, iOS only: `VideoPlayer.swift` and `VideoPlayerObserver.swift` changed, plus a new `ios/Utils/RunOnMainThread.swift`. Per the `origin/sdk-57` CHANGELOG these are 57.0.2's two bug fixes — a race when registering video player observer delegates (#47976) and `VideoPlayer` release adapted to the modified `SharedObject` lifecycle (#47828). `57.0.0` and `57.0.1` are both marked *"does not introduce any user-facing changes."*
- **`expo-audio` 56.0.13 → 57.0.3**, iOS only: `AudioModule.swift` (audio session deactivated off the main thread to avoid app hangs, #47066, in 57.0.0) and `AudioPlaylist.swift` (iOS playlist `currentIndex` no longer freezes after the first auto-advance, #47257, in 57.0.1). 57.0.3's Intune recording fix (#47005) is also in `expo-audio@56.0.13`.
- **`android/build.gradle`** for both packages differs only in the `version` / `versionName` strings. `androidxMedia3Version` is **`1.9.0` in both SDK 56 and SDK 57** — media3 did not move.
- **`expo-screen-capture`**: 57.0.0 and 57.0.1 are both *"does not introduce any user-facing changes."* Only the pin moved.
- **`expo-av`**: still not in the SDK. No `docs/pages/versions/v57.0.0/sdk/av.mdx`, and it is absent from `packages/expo/bundledNativeModules.json` on `origin/sdk-57`. Everything in the expo-av section above carries over verbatim.
- The iOS `VideoAssetTransportProvider` extension point is unchanged — the only v56→v57 diff in `sdk/video.mdx` is the `sourceCodeUrl` frontmatter line.

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-video` | `~56.1.4` | `~57.0.2` |
| `expo-audio` | `~56.0.13` | `~57.0.3` |
| `expo-screen-capture` | `~56.0.4` | `~57.0.1` |

Source: `packages/expo/bundledNativeModules.json` on `origin/sdk-56` and `origin/sdk-57`. Note these are **not** flat `~57.0.0`.

### Not in 57 — do not attribute these to the upgrade

All of the following are on `main` (SDK 58 in progress) only. Each was verified absent from `origin/sdk-57` and from the published 57.x tarballs.

| Claim | Reality |
|-------|---------|
| iOS `audioMixingMode` default flipped `doNotMix` → `auto` (#47363) | **Not in 57.** `ios/VideoPlayer.swift` in the shipped `expo-video@57.0.2` still reads `= .doNotMix`. The docs/native mismatch persists in 57 — keep setting it explicitly on iOS. |
| `AudioPlaylist` lock-screen controls `setActiveForLockScreen` / `updateLockScreenMetadata` / `clearLockScreenControls` (#46020) | **Not in 57.** `AudioPlayer` only, in both lines. |
| `AudioLockScreenOptions` gains `showNextTrack` / `showPreviousTrack` (5 fields) | **Not in 57.** `build/AudioConstants.d.ts` is identical in 56.0.13 and 57.0.3: still the 3 fields `showSeekForward`, `showSeekBackward`, `isLiveStream`. |
| Android `VideoView` prop `controllerAutoShow` (#46665) | **Not in 57.** The string does not appear anywhere in `expo-video@57.0.2`. |
| Video caching keyed on `Authorization` / auth request headers (#45995) | **Not in 57.** |
| AndroidX Media3 bumped to `1.9.1` (#45368) | **Not in 57.** Both lines pin `1.9.0`. |
| expo-video iOS player-registry thread-safety fix (#46930); `VideoView` no longer strongly retaining a detached `VideoPlayer` (#46453) | **Not in 57.** |
| expo-audio unique lock-screen `MediaSession` IDs (#47101); stale artwork on metadata update without `artworkUrl` (#45738); Android audio-focus-denied guard (#46957) | **Not in 57.** |
| expo-video `maxResolution` player option (#46992); `useVideoPlayer` using `replaceAsync` instead of re-creating the player on source change (#46495) | **Not in 57.** |
| expo-audio `RecordingOptions.fileName` (#47265); `RecorderState.fileSize` (#46808); `AudioStream.startFileRecordingAsync` / `stopFileRecordingAsync` (#46771) | **Not in 57.** |

Also **not** 57 deltas because they shipped in the SDK 56 patch line too — see "SDK 56 patch drift" above for the minimum versions: `RecordingOptions.directory` (#46189, `expo-audio >= 56.0.12`), `availableVideoTracks` deduplication (#46691, `expo-video >= 56.1.3`), failed-player recovery (#46681, `expo-video >= 56.1.3`), remote command center re-enable (#46753, `expo-video >= 56.1.4`), Intune recording crash (#47005, `expo-audio >= 56.0.13`).
