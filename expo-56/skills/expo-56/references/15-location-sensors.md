# Location, Sensors & Device-State — Expo SDK 56 & 57

Knowledge-base reference for Expo location, sensors, and device-state modules. Body targets SDK 56 (`https://docs.expo.dev/versions/v56.0.0/sdk/<pkg>/`); SDK 57 (`.../v57.0.0/sdk/<pkg>/`) is covered in the trailing [SDK 57 delta](#sdk-57-delta).

**Verdict: SDK 57 changes nothing in this domain.** All seven packages ship `## 57.0.0 — 2026-06-25 — _This version does not introduce any user-facing changes._` on `origin/sdk-57`, and no 57.0.x patch since has added an entry either. Everything a model might be tempted to call "new in 57" here — Motion Activity, the expo-sensors iOS motion-permission fix, expo-network macOS — was **published on the SDK 56 patch line** and is one `npx expo install` away without upgrading. See [SDK 57 delta](#sdk-57-delta) for the minimum 56.x patch versions.

---

## Table of Contents
1. [expo-location](#1-expo-location)
2. [expo-sensors](#2-expo-sensors)
3. [expo-screen-orientation](#3-expo-screen-orientation)
4. [expo-brightness](#4-expo-brightness)
5. [expo-battery](#5-expo-battery)
6. [expo-cellular](#6-expo-cellular)
7. [expo-network](#7-expo-network)
8. [SDK 57 delta](#sdk-57-delta)

---

## Common wrong assumptions

Things models routinely get wrong in this domain. All verified against SDK 56 **and** 57 source.

- `Battery.addPowerStateListener` **does not exist**. Only `addBatteryLevelListener`, `addBatteryStateListener`, `addLowPowerModeListener`.
- expo-cellular has **no synchronous constants** (`Cellular.carrier`, `Cellular.isoCountryCode`, …). Everything is `…Async()`.
- iOS `pausesUpdatesAutomatically` is documented as `false` but the native default is **`true`** on 56 and 57. Pass `false` explicitly.
- `Pedometer.getStepCountAsync` is **iOS-only** and can only read the past 7 days.
- `LightSensor` is **Android-only** — the iOS implementation is a stub whose `isAvailableAsync()` always returns `false`.
- `Network.getMacAddressAsync()` was removed in an earlier SDK; it is gone in 56 and 57.
- expo-brightness has **no web support** at all (`platforms: ['android','ios','expo-go']`).
- `expo-location` Motion Activity does **not** require location permissions — it has its own permission family.

### Platform matrix

| Package | Android | iOS | Web | tvOS |
|---------|:-------:|:---:|:---:|:----:|
| expo-location | ✅ | ✅ | partial (no background/geofencing) | ❌ |
| expo-sensors — Accelerometer / Gyroscope / DeviceMotion | ✅ | ✅ | ✅ | ❌ |
| expo-sensors — Magnetometer / Barometer / Pedometer | ✅ | ✅ | ❌ | ❌ |
| expo-sensors — LightSensor | ✅ | ❌ | ❌ | ❌ |
| expo-screen-orientation | ✅ | ✅ | ✅ | ❌ |
| expo-brightness | ✅ | ✅ | ❌ | ❌ |
| expo-battery | ✅ | ✅ | Chromium only | ❌ |
| expo-cellular | ✅ | partial | partial | ❌ |
| expo-network | ✅ | ✅ | ✅ | ✅ |

Docs frontmatter marks Accelerometer, Gyroscope, Barometer and expo-battery as `ios*` — **device only**, they do not work in the iOS Simulator. Identical in v56 and v57.

> **Version pins:** the authoritative source is `packages/expo/bundledNativeModules.json` on the release branch — it ships inside the `expo` package and is exactly what `npx expo install <pkg>` resolves against. See the [pin table](#version-pins-56--57). Do **not** quote `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json`: those snapshots are stale (they say expo-location `~56.0.16` / `~57.0.0`; the shipped pins are `~56.0.22` / `~57.0.6`). Patch-level facts below were confirmed against the published npm tarballs, not against `main`.

---

## 1. expo-location

Source: https://docs.expo.dev/versions/v56.0.0/sdk/location/

Provides foreground & background location, geofencing, geocoding, and compass heading.

### Installation
```sh
npx expo install expo-location
```
For existing React Native apps, ensure `expo` is installed via `npx expo install expo`.

### Config Plugin (app.json)
```json
{
  "expo": {
    "plugins": [
      [
        "expo-location",
        {
          "locationAlwaysAndWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location."
        }
      ]
    ]
  }
}
```

| Property | Default | Platform | Purpose |
|----------|---------|----------|---------|
| `locationAlwaysAndWhenInUsePermission` | `"Allow $(PRODUCT_NAME) to use your location"` | iOS | Sets `NSLocationAlwaysAndWhenInUseUsageDescription` |
| `locationAlwaysPermission` | `"Allow $(PRODUCT_NAME) to use your location"` | iOS (Deprecated) | Sets `NSLocationAlwaysUsageDescription` |
| `locationWhenInUsePermission` | `"Allow $(PRODUCT_NAME) to use your location"` | iOS | Sets `NSLocationWhenInUseUsageDescription` |
| `isIosBackgroundLocationEnabled` | `false` | iOS | Enables `location` in `UIBackgroundModes` |
| `isAndroidBackgroundLocationEnabled` | `false` | Android | Enables `ACCESS_BACKGROUND_LOCATION` |
| `isAndroidForegroundServiceEnabled` | based on background setting | Android | Enables foreground service permissions |
| `isAndroidMotionActivityEnabled` | `false` | Android | Enables `android.permission.ACTIVITY_RECOGNITION` + `com.google.android.gms.permission.ACTIVITY_RECOGNITION`, required by `getMotionActivityAsync` / `watchMotionActivityAsync` |
| `motionUsagePermission` | `"Allow $(PRODUCT_NAME) to detect your current motion activity"` | iOS | Sets `NSMotionUsageDescription`, required by Motion Activity on iOS |
| `androidForegroundServiceIcon` | — | Android | Path to 96x96 white PNG for notification icon |

> `isAndroidMotionActivityEnabled` and `motionUsagePermission` are real but **undocumented** — both are destructured in `plugin/build/withLocation.js` of the published tarballs (verified in `expo-location@56.0.22` and `@57.0.6`) yet neither appears in the v56 or v57 `location.mdx` plugin tables. Same situation as `android.usePrecompiledHeaders` in `SKILL.md`.

### iOS manual Info.plist (no CNG)
```xml
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>Allow $(PRODUCT_NAME) to use your location</string>
<key>NSLocationAlwaysUsageDescription</key>
<string>Allow $(PRODUCT_NAME) to use your location</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Allow $(PRODUCT_NAME) to use your location</string>
```
Background location on iOS (manual), add to Expo.plist:
```xml
<key>UIBackgroundModes</key>
  <array>
    <string>location</string>
  </array>
```

### Permissions

**Android (automatic):** `ACCESS_COARSE_LOCATION`, `ACCESS_FINE_LOCATION`
**Android (optional via plugin):** `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_LOCATION` (API 34+), `ACCESS_BACKGROUND_LOCATION`

**iOS keys:**
| Key | Purpose |
|-----|---------|
| `NSLocationAlwaysAndWhenInUseUsageDescription` | Both When In Use and Always access |
| `NSLocationWhenInUseUsageDescription` | Foreground (When In Use) access |
| `NSLocationAlwaysUsageDescription` | Deprecated; use Always variant |

iOS behavior notes:
- "Allow Once" and "Allow While Using the App" both return When In Use; no way to distinguish them.
- If user picked "Allow Once" and you request background perms in the same session, the request silently fails.
- Use `Linking.openURL('app-settings:')` to direct users to Settings for background access.
- You can request foreground first, then background later (incremental).

### Hooks
```ts
// Android, iOS, Web
const [status, requestPermission] = Location.useForegroundPermissions();
const [status, requestPermission] = Location.useBackgroundPermissions();
```
Return: `[PermissionResponse | null, RequestPermissionMethod, GetPermissionMethod]`.

### Core Methods

#### `getCurrentPositionAsync(options?: LocationOptions)` → `Promise<LocationObject>`
One-time current location. (Android, iOS, Web)
```ts
let location = await Location.getCurrentPositionAsync({});
```

#### `getLastKnownPositionAsync(options?: LocationLastKnownOptions)` → `Promise<LocationObject | null>`
Last known position without new request.
```ts
let location = await Location.getLastKnownPositionAsync({
  maxAge: 600000,
  requiredAccuracy: 100
});
```

#### `watchPositionAsync(options, callback, errorHandler?)` → `Promise<LocationSubscription>`
Foreground location updates.
```ts
const subscription = await Location.watchPositionAsync(
  { accuracy: Location.Accuracy.High },
  (location) => console.log(location),
  (error) => console.log(error)
);

subscription.remove();
```
- `callback`: `(location: LocationObject) => any`
- `errorHandler?`: `(reason: string) => void`

#### `getHeadingAsync()` → `Promise<LocationHeadingObject>` (Android, iOS, Web)

#### `watchHeadingAsync(callback, errorHandler?)` → `Promise<LocationSubscription>` (Android, iOS)
```ts
const subscription = await Location.watchHeadingAsync(
  (heading) => console.log(heading),
  (error) => console.log(error)
);
```

#### `geocodeAsync(address: string)` → `Promise<LocationGeocodedLocation[]>` (Android, iOS)
Requires foreground permission on Android.
```ts
const locations = await Location.geocodeAsync("Baker Street London");
```

#### `reverseGeocodeAsync(location)` → `Promise<LocationGeocodedAddress[]>` (Android, iOS)
Requires foreground permission on Android.
```ts
const addresses = await Location.reverseGeocodeAsync({
  latitude: 37.7749,
  longitude: -122.4194
});
```

### Permission Methods
- `requestForegroundPermissionsAsync()` → `Promise<LocationPermissionResponse>`
- `getForegroundPermissionsAsync()` → `Promise<LocationPermissionResponse>`
- `requestBackgroundPermissionsAsync()` → `Promise<PermissionResponse>` (must request foreground first; Android 11+ opens system settings)
- `getBackgroundPermissionsAsync()` → `Promise<PermissionResponse>`

### Location Services Status
- `hasServicesEnabledAsync()` → `Promise<boolean>`
- `getProviderStatusAsync()` → `Promise<LocationProviderStatus>`
- `isBackgroundLocationAvailableAsync()` → `Promise<boolean>`
- `enableNetworkProviderAsync()` → `Promise<void>` (Android; prompts high-accuracy mode via Play services)

### Background Location Tasks (via expo-task-manager)

#### `startLocationUpdatesAsync(taskName, options?: LocationTaskOptions)` → `Promise<void>`
```ts
import * as TaskManager from 'expo-task-manager';

TaskManager.defineTask('YOUR_TASK_NAME',
  ({ data: { locations }, error }) => {
    if (error) return;
    console.log('Received new locations', locations);
  }
);

await Location.startLocationUpdatesAsync('YOUR_TASK_NAME', {
  accuracy: Location.Accuracy.High,
  deferredUpdatesInterval: 1000
});
```
Requirements: location permissions granted; task defined via `TaskManager.defineTask`; iOS requires a development build (not in Expo Go). See `14-notifications-background.md` for the expo-task-manager API these calls depend on.

- `stopLocationUpdatesAsync(taskName)` → `Promise<void>`
- `hasStartedLocationUpdatesAsync(taskName)` → `Promise<boolean>`

> **Known bug (56 and 57, still unfixed):** on Android, `timeInterval` / `distanceInterval` passed to `startLocationUpdatesAsync` are **ignored** for background updates — only `accuracy` is applied. Use `deferredUpdatesInterval` / `deferredUpdatesDistance` instead. Root cause is in `android/src/main/java/expo/modules/location/records/LocationArguments.kt`, where the task-options constructor does `timeInterval = map["timeInterval"] as? Long` / `distanceInterval = map["distanceInterval"] as? Int?` — a JS number does not arrive as a Kotlin `Long`, so both coerce to `null`. Verified identical: the entire `android/src` tree of `expo-location@56.0.22` and `expo-location@57.0.6` is byte-for-byte the same. The fix (#46790) is in **neither** release branch — post-57, do not upgrade for it.

### Geofencing

#### `startGeofencingAsync(taskName, regions?: LocationRegion[])` → `Promise<void>`
```ts
import { GeofencingEventType } from 'expo-location';

TaskManager.defineTask('GEOFENCE_TASK',
  ({ data: { eventType, region }, error }) => {
    if (error) return;
    if (eventType === GeofencingEventType.Enter) {
      console.log("Entered region:", region);
    } else if (eventType === GeofencingEventType.Exit) {
      console.log("Left region:", region);
    }
  }
);

await Location.startGeofencingAsync('GEOFENCE_TASK', [
  {
    identifier: 'home',
    latitude: 37.7749,
    longitude: -122.4194,
    radius: 100,
    notifyOnEnter: true,
    notifyOnExit: true
  }
]);
```
Platform limits: Android max 100 active geofences; iOS max 20 monitored regions.

- `stopGeofencingAsync(taskName)` → `Promise<void>`
- `hasStartedGeofencingAsync(taskName)` → `Promise<boolean>`

### Web
- `installWebGeolocationPolyfill()` → `void` — polyfills `navigator.geolocation`.

### Types
Shape summaries, not literal declarations — add `type`/`export` yourself when copying.
```ts
LocationObject = {
  coords: LocationObjectCoords;
  mocked?: boolean;  // Android only
  timestamp: number; // ms since epoch
}

LocationObjectCoords = {
  latitude: number;
  longitude: number;
  accuracy: number | null;       // meters
  altitude: number | null;       // meters WGS 84
  altitudeAccuracy: number | null;
  heading: number | null;        // degrees 0-359.99
  speed: number | null;          // m/s
}

LocationHeadingObject = {
  magHeading: number;   // magnetic north degrees
  trueHeading: number;  // true north degrees (requires permissions)
  accuracy: number;     // 0=none,1=low,2=medium,3=high
}

LocationGeocodedLocation = {
  latitude: number; longitude: number;
  accuracy?: number; altitude?: number;
}

LocationGeocodedAddress = {
  city: string | null;
  country: string | null;
  district: string | null;
  formattedAddress: string | null;   // Android
  isoCountryCode: string | null;
  name: string | null;
  postalCode: string | null;
  region: string | null;
  street: string | null;
  streetNumber: string | null;
  subregion: string | null;
  timezone: string | null;           // iOS
}

LocationOptions = {
  accuracy?: Accuracy;            // default Balanced
  distanceInterval?: number;      // meters
  timeInterval?: number;          // Android only, ms
  mayShowUserSettingsDialog?: boolean; // Android only, default true
}

LocationLastKnownOptions = { maxAge?: number; requiredAccuracy?: number; }

LocationTaskOptions extends LocationOptions = {
  activityType?: ActivityType;            // iOS, default Other
  deferredUpdatesDistance?: number;       // meters, default 0
  deferredUpdatesInterval?: number;       // ms, default 0
  deferredUpdatesTimeout?: number;
  pausesUpdatesAutomatically?: boolean;   // iOS — TS docs claim default false, the native default is TRUE (see note)
  showsBackgroundLocationIndicator?: boolean; // iOS, default false
  foregroundService?: LocationTaskServiceOptions;
}

LocationTaskServiceOptions = {
  notificationTitle: string;
  notificationBody: string;
  notificationColor?: string;   // #RRGGBB or #AARRGGBB
  killServiceOnDestroy?: boolean;
}

LocationRegion = {
  identifier?: string;          // auto UUID if omitted
  latitude: number; longitude: number;
  radius: number;               // meters
  notifyOnEnter?: boolean;      // default true
  notifyOnExit?: boolean;       // default true
  state?: GeofencingRegionState;
}

LocationPermissionResponse extends PermissionResponse = {
  status; granted; expires; canAskAgain;
  android?: { accuracy: 'fine' | 'coarse' | 'none' };
  ios?: { scope: 'whenInUse' | 'always' | 'none'; accuracy: 'full' | 'reduced' /*iOS14+*/ };
}

LocationProviderStatus = {
  locationServicesEnabled: boolean;
  backgroundModeEnabled: boolean;
  gpsAvailable?: boolean;       // Android
  networkAvailable?: boolean;   // Android
  passiveAvailable?: boolean;   // Android
}

LocationSubscription = { remove: () => void; }
```

### Enums
```ts
Accuracy.Lowest = 1            // ~3 km
Accuracy.Low = 2              // ~1 km
Accuracy.Balanced = 3        // ~100 m (default)
Accuracy.High = 4            // ~10 m
Accuracy.Highest = 5
Accuracy.BestForNavigation = 6

ActivityType.Other = 1
ActivityType.AutomotiveNavigation = 2
ActivityType.Fitness = 3
ActivityType.OtherNavigation = 4
ActivityType.Airborne = 5     // iOS only, falls back to Other

GeofencingEventType.Enter = 1
GeofencingEventType.Exit = 2

GeofencingRegionState.Unknown = 0
GeofencingRegionState.Inside = 1
GeofencingRegionState.Outside = 2

PermissionStatus.GRANTED = "granted"
PermissionStatus.DENIED = "denied"
PermissionStatus.UNDETERMINED = "undetermined"
```

### Minimal Example
```tsx
import { useEffect, useState } from 'react';
import * as Location from 'expo-location';

const [location, setLocation] = useState<Location.LocationObject | null>(null);

useEffect(() => {
  (async () => {
    const { status } = await Location.requestForegroundPermissionsAsync();
    if (status !== 'granted') return;            // handle denial in your UI
    setLocation(await Location.getCurrentPositionAsync({}));
  })();
}, []);
```

### SDK 56 Notes
- `NSLocationAlwaysUsageDescription` is deprecated; use `NSLocationAlwaysAndWhenInUseUsageDescription`.
- Background location is not supported in Expo Go on iOS; use a development build.
- Deferred updates batch location changes when backgrounded to reduce battery use.
- Geofencing limits: iOS 20, Android 100 per app.
- **`pausesUpdatesAutomatically` really defaults to `true` on iOS.** The TS/doc comment says `false`, but `ios/TaskConsumers/EXLocationTaskConsumer.m:73` reads `defaultValue:true` in **both** published `expo-location@56.0.22` and `@57.0.6`. Background updates therefore auto-pause unless you pass `pausesUpdatesAutomatically: false`. The correction (#47008) is in neither release branch — post-57. Pass the value explicitly on both SDKs.
- Motion Activity (`getMotionActivityAsync` and friends) is present at runtime from **expo-location 56.0.10** onward but is not in the v56 docs — see [SDK 57 delta](#sdk-57-delta) for the full API. Verified by tarball: `getMotionActivityAsync` has 0 hits in `expo-location@56.0.9/build`, 6 in `@56.0.10/build`.
- Patch-level: 56.0.12 fixed an ES-module import error in the typed config plugin (#46089) — unrelated to Motion Activity. The SDK 56 pin is `~56.0.22`; every 56.0.13 → 56.0.22 release is "does not introduce any user-facing changes".

---

## 2. expo-sensors

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sensors/
Per-sensor sources: `/sdk/accelerometer/`, `/sdk/gyroscope/`, `/sdk/magnetometer/`, `/sdk/barometer/`, `/sdk/pedometer/`, `/sdk/devicemotion/`

### Installation & Import
```sh
npx expo install expo-sensors
```
```js
import * as Sensors from 'expo-sensors';
// or individual sensors:
import { Accelerometer, Barometer, DeviceMotion, Gyroscope,
         LightSensor, Magnetometer, MagnetometerUncalibrated, Pedometer }
from 'expo-sensors';
```

### Config Plugin
```json
{
  "expo": {
    "plugins": [
      ["expo-sensors", {
        "motionPermission": "Allow $(PRODUCT_NAME) to access your device motion"
      }]
    ]
  }
}
```
- `motionPermission` (iOS only) — custom message, or `false` to disable.

### Permissions
- **Android (API 31+):** sensor updates capped at 200 Hz. For higher rates add `HIGH_SAMPLING_RATE_SENSORS`:
  ```xml
  <uses-permission android:name="android.permission.HIGH_SAMPLING_RATE_SENSORS" />
  ```
- **iOS:** requires `NSMotionUsageDescription` in Info.plist for motion data.
- **Android:** expo-sensors' own `AndroidManifest.xml` declares `android.permission.ACTIVITY_RECOGNITION`, so it is merged into your app automatically — `Pedometer.requestPermissionsAsync()` requests it at runtime on Android 10+.

### Shared sensor API
Each of Accelerometer / Gyroscope / Magnetometer / MagnetometerUncalibrated / Barometer / DeviceMotion / LightSensor exposes the same class surface (platform support differs per sensor — see the [platform matrix](#platform-matrix)):
- `addListener(listener)` → `EventSubscription` (call `.remove()`)
- `setUpdateInterval(intervalMs)` — Android 12+ capped at 200 Hz
- `isAvailableAsync()` → `Promise<boolean>`
- `getListenerCount()` → `number`
- `hasListeners()` → `boolean`
- `removeSubscription(subscription)` — **deprecated**, use `subscription.remove()`
- `getPermissionsAsync()` / `requestPermissionsAsync()`
- `removeAllListeners()` — not deprecated; still the supported way to drop every listener at once

### Accelerometer (Android, iOS, Web)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/accelerometer/
Measurement (one g-force = 9.81 m/s²):
```ts
{ x: number, y: number, z: number, timestamp: number } // g-force per axis, timestamp in seconds
```
Example toggles between fast (16 ms) and slow (1000 ms) update intervals via UI buttons.

### Gyroscope (Android, iOS, Web)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/gyroscope/
`GyroscopeMeasurement`:
```ts
{ x: number, y: number, z: number, timestamp: number } // rotation rad/s per axis, timestamp seconds
```

### Magnetometer / MagnetometerUncalibrated (Android, iOS)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/magnetometer/
`MagnetometerMeasurement`:
```ts
{ x: number, y: number, z: number, timestamp: number } // microteslas per axis, timestamp seconds
```
`MagnetometerUncalibrated` is importable but has no separately documented methods/types in SDK 56.

### LightSensor (Android only)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/light-sensor/ — `platforms: ['android', 'expo-go']`.
`LightSensorMeasurement`:
```ts
{ illuminance: number /*lux*/, timestamp: number /*seconds*/ }
```
The iOS module is a stub: `isAvailableAsync()` always resolves `false` and listeners never fire (`src/ExpoLightSensor.ios.ts`).

### Barometer (Android, iOS)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/barometer/
`isAvailableAsync()` requires at least Android 2.3. Web is unsupported (throws `UnavailabilityError`).
`BarometerMeasurement`:
```ts
{ pressure: number /*hPa*/, relativeAltitude?: number /*meters, iOS only*/, timestamp: number /*seconds*/ }
```
```js
const subscription = Barometer.addListener(({ pressure, relativeAltitude }) => {
  console.log({ pressure, relativeAltitude });
});
```

### Pedometer (Android, iOS)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/pedometer/
```js
import { Pedometer } from 'expo-sensors';
```
- `Pedometer.isAvailableAsync()` → `Promise<boolean>`
- `Pedometer.getPermissionsAsync()` / `Pedometer.requestPermissionsAsync()`
- `Pedometer.getStepCountAsync(start, end)` — **iOS only**; only the past seven days of data are stored/available.
- `Pedometer.watchStepCount(callback)` → `EventSubscription` (`.remove()`); updates are NOT delivered while the app is backgrounded.

`PedometerResult`: `{ steps: number }`
`PermissionResponse`: `{ granted: boolean; status: PermissionStatus; expires: 'never' | number; canAskAgain: boolean }`
`PermissionStatus`: `GRANTED | DENIED | UNDETERMINED`

### DeviceMotion (Android, iOS, Web)
Source: https://docs.expo.dev/versions/v56.0.0/sdk/devicemotion/
```js
import { DeviceMotion } from 'expo-sensors';
```
Methods: `addListener`, `isAvailableAsync`, `requestPermissionsAsync` → `Promise<PermissionResponse>`, `getPermissionsAsync` → `Promise<PermissionResponse>`, `setUpdateInterval`, `getListenerCount`, `hasListeners`, `removeAllListeners`, `removeSubscription` (deprecated — use `subscription.remove()`).

`DeviceMotionMeasurement`:
```js
{
  acceleration: null | { timestamp, x, y, z }                  // m/s²
  accelerationIncludingGravity: { timestamp, x, y, z }         // m/s²
  interval: number                                             // ms
  orientation: 0 | 90 | 180 | -90
  rotation: { alpha, beta, gamma, timestamp }                  // degrees
  rotationRate: null | { alpha, beta, gamma, timestamp }       // deg/s
}
```
Constant `DeviceMotion.Gravity` = standard Earth gravitational acceleration `9.80665` m/s².
iOS permission key: `NSMotionUsageDescription`.

### SDK 56 Notes
- `removeSubscription(subscription)` is deprecated across all sensors; prefer `subscription.remove()`. **`removeAllListeners()` is *not* deprecated** — it carries no `@deprecated` tag in the published `DeviceSensor.d.ts` of either `expo-sensors@56.0.6` or `@57.0.2`, and remains the supported way to drop every listener at once.
- Android 12+/API 31+ enforces a 200 Hz cap unless `HIGH_SAMPLING_RATE_SENSORS` is granted.
- **Require >= 56.0.6 — iOS motion permission always `denied` below that.** With iOS precompiled modules (the default), the prebuilt `ExpoSensors.xcframework` unconditionally defined `EXPO_DISABLE_MOTION_PERMISSION`, which is only meant to be set when opting out via `motionPermission: false`. Result: `DeviceMotion` / `Pedometer` permission requests always resolve `denied`. Fixed **on the SDK 56 line** in expo-sensors `56.0.6` (#46686) — `npx expo install expo-sensors`, no SDK upgrade needed. Verified by tarball: `spm.config.json` in `56.0.5` carries a `"compilerFlags": ["-DEXPO_DISABLE_MOTION_PERMISSION=1"]` block that is gone in `56.0.6` and in `57.0.2`. If you are stuck below 56.0.6, the only workaround is `expo-build-properties` `ios.usePrecompiledModules: false` (see `16-app-config-foundational.md`).
- Patch-level: 56.0.5 fixed an ES-module import error in the typed config plugin (#46089). SDK 56 pin is `~56.0.6`.

---

## 3. expo-screen-orientation

Source: https://docs.expo.dev/versions/v56.0.0/sdk/screen-orientation/

### Installation
```sh
npx expo install expo-screen-orientation
```
For existing RN apps also run `npx expo install expo`.

### Methods
- `lockAsync(orientationLock: OrientationLock)` → `Promise<void>`
  ```ts
  await ScreenOrientation.lockAsync(ScreenOrientation.OrientationLock.LANDSCAPE_LEFT);
  ```
- `unlockAsync()` → `Promise<void>` — resets to `OrientationLock.DEFAULT`.
- `getOrientationAsync()` → `Promise<Orientation>`
- `getOrientationLockAsync()` → `Promise<OrientationLock>`
- `getPlatformOrientationLockAsync()` → `Promise<PlatformOrientationInfo>`
- `lockPlatformAsync(options: PlatformOrientationInfo)` → `Promise<void>`
- `supportsOrientationLockAsync(orientationLock: OrientationLock)` → `Promise<boolean>`

### Event Subscriptions
Supported pattern — keep your own subscription and call `.remove()`:
```ts
const sub = ScreenOrientation.addOrientationChangeListener(cb);
sub.remove();
```
- `addOrientationChangeListener(listener)` → `EventSubscription` — fires on portrait↔landscape changes.
- `removeOrientationChangeListener(subscription)` — **deprecated**, use `subscription.remove()`.
- `removeOrientationChangeListeners()` — removes all listeners; **also deprecated** ("will be removed in future versions. Keep track of your own subscriptions.").

### Enums
```ts
Orientation: UNKNOWN(0), PORTRAIT_UP(1), PORTRAIT_DOWN(2), LANDSCAPE_LEFT(3), LANDSCAPE_RIGHT(4)

OrientationLock: DEFAULT(0), ALL(1), PORTRAIT(2), PORTRAIT_UP(3), PORTRAIT_DOWN(4),
                 LANDSCAPE(5), LANDSCAPE_LEFT(6), LANDSCAPE_RIGHT(7), OTHER(8), UNKNOWN(9)

SizeClassIOS: UNKNOWN(0), COMPACT(1), REGULAR(2)

WebOrientationLock: ANY, LANDSCAPE, LANDSCAPE_PRIMARY, LANDSCAPE_SECONDARY,
                    NATURAL, PORTRAIT, PORTRAIT_PRIMARY, PORTRAIT_SECONDARY, UNKNOWN

// string-valued; distinct from WebOrientationLock
WebOrientation: PORTRAIT_PRIMARY  = 'portrait-primary'
                PORTRAIT_SECONDARY = 'portrait-secondary'
                LANDSCAPE_PRIMARY  = 'landscape-primary'
                LANDSCAPE_SECONDARY = 'landscape-secondary'
```

### Types
```ts
OrientationChangeEvent = { orientationInfo: ScreenOrientationInfo; orientationLock: OrientationLock }

PlatformOrientationInfo = {
  screenOrientationArrayIOS?: Orientation[];        // iOS
  screenOrientationConstantAndroid?: number;        // Android
  screenOrientationLockWeb?: WebOrientationLock;    // Web
}

ScreenOrientationInfo = {
  orientation: Orientation;
  horizontalSizeClass?: SizeClassIOS;  // iOS
  verticalSizeClass?: SizeClassIOS;    // iOS
}
```

### SDK 56 Notes
- No `useOrientationLock` hook exists in SDK 56.
- Both `removeOrientationChangeListener` and `removeOrientationChangeListeners` are deprecated.
- **Require >= 56.0.4.** Earlier 56.0.x crash with `EXC_BAD_ACCESS` from recursive `supportedInterfaceOrientations` when `react-native-screens` sets per-screen orientation (#45733, fixed in `56.0.4`). Also in `origin/sdk-57`, so this is not something 57 gives you.
- Patch-level: 56.0.5 fixed an ES-module import error in the typed config plugin (#46089). SDK 56 pin is `~56.0.5`.

---

## 4. expo-brightness

Source: https://docs.expo.dev/versions/v56.0.0/sdk/brightness/

### Installation
```sh
npx expo install expo-brightness
```

### Configuration / Permissions
- **Android:** requires `WRITE_SETTINGS`. For non-CNG apps add to AndroidManifest.xml:
  ```xml
  <uses-permission android:name="android.permission.WRITE_SETTINGS" />
  ```
- **iOS:** no permission required.

### Methods
- `getBrightnessAsync()` → current app brightness 0–1 (Android, iOS)
- `setBrightnessAsync(brightnessValue)` — sets screen brightness; on iOS persists until device locks.
- `getSystemBrightnessAsync()` — global system brightness (Android only)
- `setSystemBrightnessAsync(brightnessValue)` — sets system brightness; requires `WRITE_SETTINGS` (Android)
- `restoreSystemBrightnessAsync()` — reset to system-wide brightness (Android)
- `isUsingSystemBrightnessAsync()` — whether using system value (Android)
- `getSystemBrightnessModeAsync()` → `BrightnessMode` (Android)
- `setSystemBrightnessModeAsync(brightnessMode)` — set MANUAL/AUTOMATIC (Android)
- `isAvailableAsync()` → `Promise<boolean>`
- `addBrightnessListener(listener: (event: BrightnessEvent) => void)` → `EventSubscription` (native event `Expo.brightnessDidChange`)
- `getPermissionsAsync()` / `requestPermissionsAsync()`

`BrightnessEvent`: `{ brightness: number }` — 0–1.

**No web support**: `platforms: ['android', 'ios', 'expo-go']`.

### Hook
```ts
const [permissionResponse, requestPermission] = Brightness.usePermissions();
```

### Enum
```ts
BrightnessMode.UNKNOWN = 0
BrightnessMode.AUTOMATIC = 1  // OS adjusts automatically
BrightnessMode.MANUAL = 2     // constant brightness
```

### Example
```jsx
import * as Brightness from 'expo-brightness';

useEffect(() => {
  (async () => {
    const { status } = await Brightness.requestPermissionsAsync();
    if (status === 'granted') {
      Brightness.setSystemBrightnessAsync(1);
    }
  })();
}, []);
```

### SDK 56 Notes
- Android/iOS only — there is no web implementation.
- Patch-level: 56.0.5 fixed an ES-module import error in the typed config plugin (#46089). SDK 56 pin is `~56.0.5`.

---

## 5. expo-battery

Source: https://docs.expo.dev/versions/v56.0.0/sdk/battery/

### Installation
```sh
npx expo install expo-battery
```

### Hooks
- `useBatteryLevel()` → `number`
- `useBatteryState()` → `BatteryState`
- `useLowPowerMode()` → `boolean`
- `usePowerState()` → `{ batteryLevel, batteryState, lowPowerMode }`

### Methods
- `getBatteryLevelAsync()` → battery level 0–1
  ```ts
  await Battery.getBatteryLevelAsync(); // 0.759999
  ```
- `getBatteryStateAsync()` → `Promise<BatteryState>`
  ```ts
  await Battery.getBatteryStateAsync(); // BatteryState.CHARGING
  ```
- `getPowerStateAsync()` → `Promise<PowerState>`
  ```ts
  await Battery.getPowerStateAsync();
  // { batteryLevel: 0.759999, batteryState: BatteryState.UNPLUGGED, lowPowerMode: true }
  ```
- `isLowPowerModeEnabledAsync()` → `Promise<boolean>` (power saver / low power mode)
- `isBatteryOptimizationEnabledAsync()` → `Promise<boolean>` (Android 6.0+)
- `isAvailableAsync()` → `Promise<boolean>`

### Event Listeners
There are exactly **three**. All return `EventSubscription` (re-exported as `Subscription`) with `.remove()`:
- `addBatteryLevelListener(cb: (e: BatteryLevelEvent) => void)` — fires on significant level change
- `addBatteryStateListener(cb: (e: BatteryStateEvent) => void)` — fires on charging-state change
- `addLowPowerModeListener(cb: (e: PowerModeEvent) => void)` — fires when power mode toggles

There is **no** `addPowerStateListener`. Compose `usePowerState()` or the three listeners above instead.

### Types
```ts
BatteryState: UNKNOWN, UNPLUGGED, CHARGING, FULL, NOT_CHARGING /*Android only*/

PowerState = {
  batteryLevel: number;
  batteryState: BatteryState;
  lowPowerMode: boolean;
}

BatteryLevelEvent = { batteryLevel: number }
BatteryStateEvent = { batteryState: BatteryState }
PowerModeEvent    = { lowPowerMode: boolean }
```

### SDK 56 Notes
- **Web:** expo-battery uses the [Battery Status API](https://developer.mozilla.org/en-US/docs/Web/API/Battery_Status_API) (`navigator.getBattery`), implemented only in Chromium browsers (Chrome, Edge, Opera). On unsupported browsers `getBatteryLevelAsync()` resolves to `-1`, `getBatteryStateAsync()` resolves to `BatteryState.UNKNOWN`, and `isAvailableAsync()` returns `false`. (The v57 docs cut dropped this note, but the web implementation is unchanged — verified in `src/ExpoBattery.web.ts` of published `expo-battery@57.0.1`: `if (!batteryManager) return -1;` / `return BatteryState.UNKNOWN;`.)

---

## 6. expo-cellular

Source: https://docs.expo.dev/versions/v56.0.0/sdk/cellular/

### Installation & Import
```sh
npx expo install expo-cellular
```
```js
import * as Cellular from 'expo-cellular';
```

### Permissions
- **Android:** `READ_PHONE_STATE` required (declare in app.json).
- **iOS:** none required.

### Methods
No synchronous constants exist. `Cellular.carrier`, `Cellular.isoCountryCode`, `Cellular.allowsVoip`, `Cellular.mobileCountryCode` and `Cellular.mobileNetworkCode` were removed in an earlier SDK and are **not** exported in 56 or 57 — the full surface is the list below plus `CellularGeneration`.

- `allowsVoipAsync()` — **deprecated**. `Promise<boolean>` (Android); `null` on iOS/web.
- `getCarrierNameAsync()` → `Promise<string>` (Android only; e.g. `"T-Mobile"`); `null` on iOS/web.
- `getCellularGenerationAsync()` → `Promise<CellularGeneration>` (Android, iOS, Web)
- `getIsoCountryCodeAsync()` → `Promise<string>` (Android only; e.g. `"us"`); `null` on iOS/web.
- `getMobileCountryCodeAsync()` → `Promise<string>` (Android only; e.g. `"310"`); `null` on iOS/web.
- `getMobileNetworkCodeAsync()` → `Promise<string>` (Android only); `null` on iOS/web.
- `getPermissionsAsync()` / `requestPermissionsAsync()` → `Promise<PermissionResponse>`

### Hook
```ts
const [status, requestPermission] = Cellular.usePermissions();
```

### Enum
```ts
CellularGeneration.UNKNOWN = 0     // not connected / indeterminate
CellularGeneration.CELLULAR_2G = 1 // CDMA, EDGE, GPRS, IDEN
CellularGeneration.CELLULAR_3G = 2 // EHRPD, EVDO, HSPA, UMTS
CellularGeneration.CELLULAR_4G = 3 // LTE
CellularGeneration.CELLULAR_5G = 4 // NR, NRNSA
```

### SDK 56 Notes
- `allowsVoipAsync()` is deprecated but **present in both 56 and 57** — settled by tarball, not by docs: `build/Cellular.d.ts` of `expo-cellular@56.0.5` and `@57.0.1` both declare `export declare function allowsVoipAsync(): Promise<boolean | null>;`. Its removal (#47148) is in **neither** release branch's changelog, i.e. post-57. Do not plan a 56→57 migration around it.
- Patch-level: 56.0.5 fixed an ES-module import error in the typed config plugin (#46089). SDK 56 pin is `~56.0.5`.

---

## 7. expo-network

Source: https://docs.expo.dev/versions/v56.0.0/sdk/network/

### Installation & Import
```sh
npx expo install expo-network
```
```js
import * as Network from 'expo-network';
```

### Configuration
Android adds `ACCESS_NETWORK_STATE` and `ACCESS_WIFI_STATE` automatically.

### Hook
```ts
// Android, iOS, tvOS, Web
const networkState = useNetworkState();
console.log(`Current network type: ${networkState.type}`);
```
Returns `NetworkState`; sets up a listener cleaned up on unmount.

### Methods
- `getIpAddressAsync()` → `Promise<string>` (Android, iOS, tvOS, Web) — IPv4, `0.0.0.0` if unavailable; web uses ipify for public IP.
  ```ts
  await Network.getIpAddressAsync(); // "92.168.32.44"
  ```
- `getNetworkStateAsync()` → `Promise<NetworkState>` (Android, iOS, tvOS, Web) — on web `type` is `UNKNOWN` if connected else `NONE`.
  ```ts
  await Network.getNetworkStateAsync();
  // { type: NetworkStateType.CELLULAR, isConnected: true, isInternetReachable: true }
  ```
- `isAirplaneModeEnabledAsync()` → `Promise<boolean>` (Android only)
  ```ts
  await Network.isAirplaneModeEnabledAsync(); // false
  ```

### Event Subscriptions
- `addNetworkStateListener(listener)` → `EventSubscription` (Android, iOS, tvOS, Web)
  ```ts
  const subscription = addNetworkStateListener(({ type, isConnected, isInternetReachable }) => {
    console.log(`Network type: ${type}, Connected: ${isConnected}, Internet Reachable: ${isInternetReachable}`);
  });
  ```

### Types
```ts
NetworkState = {
  isConnected: boolean;          // active connection (false for NONE/UNKNOWN)
  isInternetReachable: boolean;  // see note below; matches isConnected on iOS
  type: NetworkStateType;
}

NetworkStateEvent = NetworkState  // payload to addNetworkStateListener
```

### Enum: NetworkStateType
```ts
BLUETOOTH  // "BLUETOOTH" (Android)
CELLULAR   // "CELLULAR"  (Android, iOS)
ETHERNET   // "ETHERNET"  (Android, iOS)
NONE       // "NONE"
OTHER      // "OTHER"     (Android)
UNKNOWN    // "UNKNOWN"
VPN        // "VPN"       (Android)
WIFI       // "WIFI"      (Android, iOS)
WIMAX      // "WIMAX"     (Android)
```

### Error Codes
| Code | Description |
|------|-------------|
| `ERR_NETWORK_IP_ADDRESS` | Unknown Wi-Fi host or no network interfaces (iOS) |
| `ERR_NETWORK_UNDEFINED_INTERFACE` | Undefined `interfaceName` parameter |
| `ERR_NETWORK_SOCKET_EXCEPTION` | Socket creation/access error |
| `ERR_NETWORK_INVALID_PERMISSION_INTERNET` | Missing `ACCESS_WIFI_STATE` permission |
| `ERR_NETWORK_NO_ACCESS_NETWORKINFO` | Unable to access network information |

### SDK 56 Notes
- `useNetworkState()` hook and `addNetworkStateListener()` are available; tvOS is a supported platform alongside Android/iOS/Web.
- `getMacAddressAsync()` is no longer documented in SDK 56 (removed in earlier SDKs due to OS restrictions).
- **Android `isInternetReachable` changed inside the 56.0.x line** (#45101, landed in expo-network **56.0.4** — after the v56 docs freeze, so the v56 docs still describe the old semantics; note the release-branch changelog files this entry under the `56.0.0` heading, but the tarballs disagree: `NET_CAPABILITY_VALIDATED` has 0 hits in `expo-network@56.0.0/android` and 1 from `56.0.4` on). Current behaviour on 56.0.4+ and 57: `true` requires the active network to have internet capability (`NET_CAPABILITY_INTERNET`), confirmed internet access (`NET_CAPABILITY_VALIDATED`), and a usable connection state; VPN connections additionally require non-zero downstream bandwidth. The module now also emits network state changes when network **capabilities** change, not only on connect/disconnect. Do not rely on the old `NetInfo.isConnected()` / `getActiveNetwork()` semantics.
- **macOS support landed on the SDK 56 line too** — expo-network `56.0.5` (#46535). `ios/ExpoNetwork.podspec` gains `:osx => '13.4'` in `56.0.5`; it is absent in `56.0.4` and present in `57.0.1`. The docs frontmatter never caught up (`platforms: ['android', 'ios', 'web', 'tvos', 'expo-go']` in both v56 and v57 `network.mdx`), so macOS is shipped-but-undocumented on both SDKs. SDK 56 pin is `~56.0.5`.

---

## SDK 57 delta

**There is no delta.** Every one of the seven packages carries the same 57.0.0 entry on `origin/sdk-57`:

```
## 57.0.0 — 2026-06-25

_This version does not introduce any user-facing changes._
```

and every subsequent 57.0.x release on that branch (location up to 57.0.6, sensors to 57.0.2, the other five to 57.0.1) is an empty heading. Nothing in sections 1–7 was added, removed, or resignatured by the 56 → 57 move.

**Methodology.** Claims below come from `git show origin/sdk-56|origin/sdk-57:packages/<pkg>/CHANGELOG.md` plus the published npm tarballs (`npm pack <pkg>@<version>`, then read `build/*.d.ts`, `ios/`, `android/src/`, `plugin/build/`). The versioned docs pages and `docs/public/static/data/v5x.0.0/*.json` were **not** used and should not be: the v57 data files are copies of an unversioned snapshot taken before the 57 cut, so they lag reality in both directions.

### Breaking

**None.** Nothing in this domain was removed or resignatured in 57.

Two removals are frequently mis-attributed to 57. Both are **post-57** — present in neither release branch — so do not migrate away from them when upgrading:

- **expo-cellular `allowsVoipAsync` is NOT removed in 57.** `build/Cellular.d.ts` of `expo-cellular@57.0.1` still declares `export declare function allowsVoipAsync(): Promise<boolean | null>;`, identical to `@56.0.5`. #47148 removes it but appears in neither `origin/sdk-56` nor `origin/sdk-57`. Still deprecated on both; still safe to call on both.
- **expo-location keeps its permission-response shape.** The removal of the stale top-level `scope` / `accuracy` from the foreground permission response (#48009) is post-57. `LocationPermissionResponse` is `PermissionResponse & { ios?, android? }` in the published `56.0.22` and `57.0.6` alike, so the shape in section 1 stays valid for both.

### Not a 57 delta — SDK 56 patch drift

Everything a model is likely to file under "new in 57" here actually shipped on the SDK 56 line. Each is reachable with `npx expo install <pkg>` at the SDK 56 pin; none of them is a reason to upgrade.

| Feature / fix | Shipped in | Also in 57 | PR |
|---|---|---|---|
| expo-location Motion Activity API | **56.0.10** | yes | #44893 |
| expo-sensors iOS motion permission no longer always `denied` | **56.0.6** | yes | #46686 |
| expo-network macOS platform support | **56.0.5** | yes | #46535 |
| expo-network Android `isInternetReachable` hardening | **56.0.4** | yes | #45101 |
| expo-screen-orientation iOS `EXC_BAD_ACCESS` crash fix | **56.0.4** | yes | #45733 |
| ES-module import error in the typed config plugins | **56.0.5** (location: 56.0.12) | yes | #46089 |

The current SDK 56 pins (location `~56.0.22`, sensors `~56.0.6`, network / screen-orientation / cellular / brightness `~56.0.5`, battery `~56.0.4`) all sit at or above the minimums above, so a project already on the SDK 56 pin has every one of these.

#### expo-location — Motion Activity (API reference)

Documented from the v57 docs page but **runtime-available from expo-location `56.0.10`** — the v56 docs simply froze before it landed, which is why it reads as a 57 feature. Verified by tarball: 0 hits for `getMotionActivityAsync` in `56.0.9/build`, 6 in `56.0.10/build`, 6 in `57.0.6/build`.

```ts
getMotionActivityAsync(): Promise<MotionActivityObject>
watchMotionActivityAsync(
  callback: MotionActivityCallback,
  errorHandler?: LocationErrorCallback
): Promise<LocationSubscription>
getMotionActivityPermissionsAsync(): Promise<PermissionResponse>
requestMotionActivityPermissionsAsync(): Promise<PermissionResponse>
useMotionActivityPermissions()  // createPermissionHook pair
```

Types (`packages/expo-location/src/Location.types.ts`):
```ts
enum MotionActivityConfidence { Low = 0, Medium = 1, High = 2 }

enum MotionActivityType {
  Automotive = 'automotive',
  Cycling = 'cycling',
  Running = 'running',
  Walking = 'walking',
  Stationary = 'stationary',
  Unknown = 'unknown',
}

MotionActivityState = {
  detected: boolean;
  confidence: MotionActivityConfidence;   // Low whenever detected is false
}

MotionActivityObject = {
  activities: Record<MotionActivityType, MotionActivityState>;  // keyed record, NOT an array
  timestamp: number;                                            // ms since epoch
}

MotionActivityCallback = (activity: MotionActivityObject) => any
```

```ts
const { activities } = await Location.getMotionActivityAsync();
if (activities.automotive.detected) {
  console.log('driving, confidence:', activities.automotive.confidence);
}
```

Platform semantics (`src/Location.ts`, `ios/LocationModule.swift`, `android/.../LocationModule.kt` of the published tarball):
- Android + iOS only. On web `watchMotionActivityAsync` just `console.warn`s and the permission getters return `UNDETERMINED` / `granted: false`.
- **Does not require location permissions** — it has its own permission family.
- Android: uses Google Play Services activity recognition; on Android 10+ `android.permission.ACTIVITY_RECOGNITION` must be granted first (enable it via the plugin's `isAndroidMotionActivityEnabled`, see section 1).
- iOS: uses `CMMotionActivityManager`, needs `NSMotionUsageDescription`, and requires a physical device with a motion coprocessor (iPhone 5s and later) — it throws otherwise.
- Foreground only: updates pause when the app is backgrounded and resume on return.
- `getMotionActivityAsync` is a convenience wrapper — it subscribes, takes the first update, and unsubscribes.
- Confidence differs by platform: Android buckets each activity's raw 0–100 `DetectedActivity` probability independently; iOS reports one reading-wide `CMMotionActivityConfidence` shared by every detected activity.

### Bugs that survive the upgrade

Upgrading to 57 does not fix any of these. Both SDKs need the same workaround.

- **`pausesUpdatesAutomatically` still defaults to `true` on iOS.** `EXLocationTaskConsumer.m:73` reads `defaultValue:true` in published `56.0.22` and `57.0.6` alike. The correction (#47008) is in neither release branch. Pass `false` explicitly.
- **Android background `timeInterval` / `distanceInterval` are still ignored.** The whole `android/src` tree is byte-identical between `56.0.22` and `57.0.6`; the fix (#46790) is in neither release branch. Use `deferredUpdatesInterval` / `deferredUpdatesDistance`.

### Docs-only differences (no code change)

Cosmetic v56 → v57 documentation drift. Nothing to act on; listed so a docs diff does not get mistaken for an API change.

- **expo-battery's web / Battery Status API note was dropped from the v57 docs page.** Docs regression only — `src/ExpoBattery.web.ts` is unchanged, so section 5's web note holds for 57 too.
- **expo-location plugin wording:** the `isAndroidForegroundServiceEnabled` description dropped the parenthetical "(required on Android 14 and later when running a location foreground service)".
- **expo-cellular:** the doc comments on `allowsVoipAsync` and `getCellularGenerationAsync` were reworded. Signatures unchanged.
- **Motion Activity and expo-network macOS appear "new" in the v57 docs** purely because the v56 docs froze before they were published on the 56 line. See the patch-drift table above.

### Version pins (56 → 57)

From `packages/expo/bundledNativeModules.json` on `origin/sdk-56` and `origin/sdk-57` — the file that ships inside the `expo` package and that `npx expo install` resolves against:

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| expo-battery | `~56.0.4` | `~57.0.1` |
| expo-brightness | `~56.0.5` | `~57.0.1` |
| expo-cellular | `~56.0.5` | `~57.0.1` |
| expo-location | `~56.0.22` | `~57.0.6` |
| expo-network | `~56.0.5` | `~57.0.1` |
| expo-screen-orientation | `~56.0.5` | `~57.0.1` |
| expo-sensors | `~56.0.6` | `~57.0.2` |
| expo-task-manager (needed for background location) | `~56.0.23` | `~57.0.6` |

The 57 column is **not** a flat `~57.0.0` — do not assume it. And do not quote `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json`: those snapshots are frozen at the SDK cut and already wrong (they give expo-location `~56.0.16` / `~57.0.0`).

Minimum useful 56.x versions established above: expo-location `56.0.10` (Motion Activity), expo-sensors `56.0.6` (iOS motion permission), expo-network `56.0.5` (macOS) / `56.0.4` (`isInternetReachable`), expo-screen-orientation `56.0.4` (iOS crash fix). All are at or below the current SDK 56 pins, so `npx expo install` on SDK 56 gets you every one of them.
