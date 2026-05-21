# Video & Audio: Playback / Recording (Expo SDK 56)

Domain reference for the Expo SDK 56 knowledge base. Covers `expo-video`, `expo-audio` (playback + recording), the deprecated/removed `expo-av`, and `expo-screen-capture`.

> SDK 56 note: `expo-av` is no longer part of the SDK. Its v56 doc page (`/versions/v56.0.0/sdk/av/`) and the `latest` page both return HTTP 404 — see the [expo-av](#expo-av-deprecated--removed) section for the migration path. Treat `expo-video` and `expo-audio` as the supported APIs.

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
- `exitFullscreen(): Promise<void>`
- `startPictureInPicture(): Promise<void>`
- `stopPictureInPicture(): Promise<void>`

### Class: `VideoPlayer`

Properties:

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `allowsExternalPlayback` | boolean | `true` | AirPlay (iOS) |
| `audioMixingMode` | `'mixWithOthers' \| 'duckOthers' \| 'auto' \| 'doNotMix'` | `'auto'` | Audio interaction |
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

### Module static methods

- `createVideoPlayer(source, playerBuilderOptions?): VideoPlayer`
- `isPictureInPictureSupported(): boolean`
- `getCurrentVideoCacheSize(): number`
- `setVideoCacheSizeAsync(sizeBytes: number): Promise<void>`
- `clearVideoCacheAsync(): Promise<void>`

### Events (VideoPlayer)

`playingChange`, `playbackRateChange`, `mutedChange`, `volumeChange`, `statusChange`, `sourceChange`, `sourceLoad`, `playToEnd`, `audioTrackChange`, `subtitleTrackChange`, `videoTrackChange`, `availableAudioTracksChange`, `availableSubtitleTracksChange`, `timeUpdate`, `isExternalPlaybackActiveChange` (iOS).

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
  minBufferForPlayback?: number;            // seconds, Android default 2
  preferredForwardBufferDuration?: number;  // Android 20, iOS 0
  maxBufferBytes?: number;                  // Android
  waitsToMinimizeStalling?: boolean;        // iOS default true
  prioritizeTimeOverSizeThreshold?: boolean; // Android
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

interface SeekTolerance { toleranceBefore?: number; toleranceAfter?: number; } // seconds

interface ScrubbingModeOptions {
  scrubbingModeEnabled?: boolean;       // Android, iOS
  increaseCodecOperatingRate?: boolean; // Android
  allowSkippingMediaCodecFlush?: boolean; // Android
  useDecodeOnlyFlag?: boolean;          // Android
  enableDynamicScheduling?: boolean;    // Android
}

interface ButtonOptions {
  showPlayPause?: boolean;
  showSeekBackward?: boolean;
  showSeekForward?: boolean;
  showSettings?: boolean;
  showSubtitles?: boolean | null;
  showNext?: boolean;
  showPrevious?: boolean;
  showBottomBar?: boolean;
}

interface FullscreenOptions { enable: boolean; }

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

> `useAudioStream` (real-time PCM microphone capture) is documented in the streaming/recording domain and only summarized here.

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

Configurable properties: `microphonePermission` (iOS), `recordAudioAndroid` (Android RECORD_AUDIO), `enableBackgroundRecording`, `enableBackgroundPlayback` (default `true`).

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
- `setActiveForLockScreen(active, metadata?, options?)`
- `updateLockScreenMetadata(metadata)`
- `clearLockScreenControls()`

### Class: `AudioRecorder`

Properties: `currentTime` (seconds), `isRecording`, `uri` (`string | null`), `id`.

Methods:
- `prepareToRecordAsync(options?)`
- `record(options?)`
- `recordForDuration(seconds)` — deprecated
- `startRecordingAtTime(seconds)` — deprecated
- `pause()`, `stop()`
- `getStatus()` → `RecorderState`
- `getAvailableInputs()`, `getCurrentInput()`, `setInput(inputUid)`

### Module static methods (`AudioModule` / `Audio.*`)

- `setAudioModeAsync(mode)` — global audio behavior (see properties below).
- `setIsAudioActiveAsync(active)` — enable/disable audio subsystem.
- `requestRecordingPermissionsAsync()`, `getRecordingPermissionsAsync()`
- `requestNotificationPermissionsAsync()` (Android)
- `preload(source, options?)`, `getPreloadedSources()`, `clearPreloadedSource(source)`, `clearAllPreloadedSources()`
- `createAudioPlayer(source?, options?)` — manual creation; requires `release()`.
- `createAudioPlaylist(options?)` — manual creation.

`setAudioModeAsync(mode)` accepts a partial `AudioMode`:
- `playsInSilentMode` (boolean, default `true`)
- `shouldPlayInBackground` (boolean, default `false`)
- `allowsRecording` (boolean, iOS, default `false`)
- `allowsBackgroundRecording` (boolean, default `false`)
- `interruptionMode`: `'mixWithOthers' | 'doNotMix' | 'duckOthers'` (default `'mixWithOthers'`)
- `shouldRouteThroughEarpiece` (boolean, iOS)

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
interface AudioStatus {
  currentTime: number;
  duration: number;
  playing: boolean;
  isLoaded: boolean;
  isBuffering: boolean;
  error: string | null;
  didJustFinish: boolean;
  loop: boolean;
  mute: boolean;
  playbackRate: number;
  timeControlStatus: string;
  playbackState: string;
  isLive: boolean;
  currentOffsetFromLive: number | null;
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
  android?: RecordingOptionsAndroid;
  ios?: RecordingOptionsIos;
  web?: RecordingOptionsWeb;
}

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
```

### Enums

- `AudioQuality`: `MIN` (0), `LOW` (32), `MEDIUM` (64), `HIGH` (96), `MAX` (127)
- `IOSOutputFormat`: MPEG4AAC, LINEARPCM, APPLELOSSLESS, etc.
- `AndroidAudioEncoder`: `'default'`, `'amr_nb'`, `'amr_wb'`, `'aac'`, `'he_aac'`, `'aac_eld'`
- `AndroidOutputFormat`: `'default'`, `'3gp'`, `'mpeg4'`, `'amrnb'`, `'amrwb'`, `'aac_adts'`, `'mpeg2ts'`, `'webm'`
- `RecordingSource` (Android): `'camcorder'`, `'default'`, `'mic'`, `'unprocessed'`, `'voice_communication'`, `'voice_performance'`, `'voice_recognition'`
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

### SDK 56-specific audio changes

Per the SDK 56 changelog (https://expo.dev/changelog/sdk-56):
- New `useAudioStream` hook for real-time microphone buffer access.
- Live-stream improvements: iOS `isLiveStream` lock-screen option; Android `playsInSilentMode` support; `AudioStatus` now includes `isLive`, `currentOffsetFromLive`, and `error` fields.

---

## expo-av (deprecated / removed)

Source: `/versions/v56.0.0/sdk/av/` and `/versions/latest/sdk/av/` both return **HTTP 404** in SDK 56 — `expo-av` is no longer documented or shipped. Deprecation notice captured from the last release that documented it (SDK 54): https://docs.expo.dev/versions/v54.0.0/sdk/av/

Deprecation notice (verbatim, SDK 54 docs):

> **Deprecated:** The `Video` and `Audio` APIs from `expo-av` have now been deprecated and replaced by improved versions in `expo-video` and `expo-audio`. We recommend using those libraries instead. `expo-av` is not receiving patches and will be removed in SDK 55.

Timeline:
- Deprecated in SDK 53.
- SDK 54 was the last release shipping `expo-av` as part of the SDK; already removed from Expo Go.
- Removed from the SDK starting SDK 55. By SDK 56 the doc pages are gone (404).

Migration path:
- Video playback → migrate to **expo-video** (`useVideoPlayer` + `VideoView`).
- Audio playback and recording → migrate to **expo-audio** (`useAudioPlayer`, `useAudioRecorder`).
- Both replacement APIs are React hook-based and idiomatic for modern React. Recommended order: migrate video first, then audio.

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

- Android 14+ requires no permissions for blocking capture.
- Android 13 and below require `READ_MEDIA_IMAGES`, configured via `android.permissions` in app config.

---

## Source URLs

- expo-video: https://docs.expo.dev/versions/v56.0.0/sdk/video/
- expo-audio: https://docs.expo.dev/versions/v56.0.0/sdk/audio/
- expo-av (404 in SDK 56; deprecation notice from SDK 54 archive): https://docs.expo.dev/versions/v54.0.0/sdk/av/
- expo-screen-capture: https://docs.expo.dev/versions/v56.0.0/sdk/screen-capture/
- SDK 56 changelog (authoritative for version changes): https://expo.dev/changelog/sdk-56
