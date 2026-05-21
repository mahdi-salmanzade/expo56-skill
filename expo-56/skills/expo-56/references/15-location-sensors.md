# Location, Sensors & Device-State — Expo SDK 56

Knowledge-base reference for Expo SDK 56 location, sensors, and device-state modules. All APIs target SDK 56 (`https://docs.expo.dev/versions/v56.0.0/sdk/<pkg>/`).

---

## Table of Contents
1. [expo-location](#1-expo-location)
2. [expo-sensors](#2-expo-sensors)
3. [expo-screen-orientation](#3-expo-screen-orientation)
4. [expo-brightness](#4-expo-brightness)
5. [expo-battery](#5-expo-battery)
6. [expo-cellular](#6-expo-cellular)
7. [expo-network](#7-expo-network)

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
| `androidForegroundServiceIcon` | — | Android | Path to 96x96 white PNG for notification icon |

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
Requirements: location permissions granted; task defined via `TaskManager.defineTask`; iOS requires a development build (not in Expo Go).

- `stopLocationUpdatesAsync(taskName)` → `Promise<void>`
- `hasStartedLocationUpdatesAsync(taskName)` → `Promise<boolean>`

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
  city, country, district, formattedAddress /*Android*/, isoCountryCode,
  name, postalCode, region, street, streetNumber, subregion,
  timezone /*iOS*/ : string | null;
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
  pausesUpdatesAutomatically?: boolean;   // iOS, default false
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

### Full Example
```tsx
import { useState, useEffect } from 'react';
import { Text, View, StyleSheet } from 'react-native';
import * as Location from 'expo-location';

export default function App() {
  const [location, setLocation] = useState<Location.LocationObject | null>(null);
  const [errorMsg, setErrorMsg] = useState<string | null>(null);

  useEffect(() => {
    async function getCurrentLocation() {
      let { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') {
        setErrorMsg('Permission to access location was denied');
        return;
      }

      let location = await Location.getCurrentPositionAsync({});
      setLocation(location);
    }

    getCurrentLocation();
  }, []);

  let text = 'Waiting...';
  if (errorMsg) text = errorMsg;
  else if (location) text = JSON.stringify(location);

  return (
    <View style={styles.container}>
      <Text style={styles.paragraph}>{text}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 20 },
  paragraph: { fontSize: 18, textAlign: 'center' },
});
```

### SDK 56 Notes
- `NSLocationAlwaysUsageDescription` is deprecated; use `NSLocationAlwaysAndWhenInUseUsageDescription`.
- Background location is not supported in Expo Go on iOS; use a development build.
- Deferred updates batch location changes when backgrounded to reduce battery use.
- Geofencing limits: iOS 20, Android 100 per app.

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

### Shared sensor API
Each of Accelerometer / Gyroscope / Magnetometer / MagnetometerUncalibrated / Barometer / DeviceMotion / LightSensor exposes:
- `addListener(listener)` → `EventSubscription` (call `.remove()`)
- `setUpdateInterval(intervalMs)` — Android 12+ capped at 200 Hz
- `isAvailableAsync()` → `Promise<boolean>`
- `getListenerCount()` → `number`
- `hasListeners()` → `boolean`
- `removeSubscription(subscription)`
- `getPermissionsAsync()` / `requestPermissionsAsync()`
- `removeAllListeners()` — **deprecated**, use `subscription.remove()`

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
Methods: `addListener`, `isAvailableAsync`, `requestPermissionsAsync` → `Promise<PermissionResponse>`, `getPermissionsAsync` → `Promise<PermissionResponse>`, `setUpdateInterval`, `getListenerCount`, `hasListeners`, `removeAllListeners` (deprecated), `removeSubscription`.

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
- `removeAllListeners()` is deprecated across all sensors; prefer `subscription.remove()`.
- Android 12+/API 31+ enforces a 200 Hz cap unless `HIGH_SAMPLING_RATE_SENSORS` is granted.

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
- `addOrientationChangeListener(listener)` → `EventSubscription` — fires on portrait↔landscape changes.
- `removeOrientationChangeListener(subscription)` — **deprecated**.
- `removeOrientationChangeListeners()` — removes all listeners.

### Enums
```ts
Orientation: UNKNOWN(0), PORTRAIT_UP(1), PORTRAIT_DOWN(2), LANDSCAPE_LEFT(3), LANDSCAPE_RIGHT(4)

OrientationLock: DEFAULT(0), ALL(1), PORTRAIT(2), PORTRAIT_UP(3), PORTRAIT_DOWN(4),
                 LANDSCAPE(5), LANDSCAPE_LEFT(6), LANDSCAPE_RIGHT(7), OTHER(8), UNKNOWN(9)

SizeClassIOS: UNKNOWN(0), COMPACT(1), REGULAR(2)

WebOrientationLock: ANY, LANDSCAPE, LANDSCAPE_PRIMARY, LANDSCAPE_SECONDARY,
                    NATURAL, PORTRAIT, PORTRAIT_PRIMARY, PORTRAIT_SECONDARY, UNKNOWN
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
- `removeOrientationChangeListener` is deprecated.

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
- `getPermissionsAsync()` / `requestPermissionsAsync()`

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
All return a `Subscription` with `.remove()`:
- `addBatteryLevelListener(callback)` — fires on significant level change
- `addBatteryStateListener(callback)` — fires on charging-state change
- `addLowPowerModeListener(callback)` — fires when power mode toggles
- `addPowerStateListener(callback)`

### Types
```ts
BatteryState: UNKNOWN, UNPLUGGED, CHARGING, FULL, NOT_CHARGING /*Android only*/

PowerState = {
  batteryLevel: number;
  batteryState: BatteryState;
  lowPowerMode: boolean;
}
```

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

### Constants
`allowsVoip`, `carrier`, `isoCountryCode`, `mobileCountryCode`, `mobileNetworkCode` — synchronous values (largely Android; null on iOS/web).

### Methods
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
  isInternetReachable: boolean;  // matches isConnected on iOS
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
