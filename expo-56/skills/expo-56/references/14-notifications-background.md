# Notifications & Background Execution — Expo SDK 56 Reference

Knowledge base reference covering local/push notifications, background task execution, and device/application introspection for Expo SDK 56. Each section cites its source URL.

---

## 1. expo-notifications

Source: https://docs.expo.dev/versions/v56.0.0/sdk/notifications/

Library for scheduling local notifications, handling incoming notifications, managing Android channels, interactive categories, and obtaining push tokens.

### Installation

```sh
npx expo install expo-notifications
```

### App Config Plugin

Configure via `app.json` / `app.config.js`:

```json
{
  "expo": {
    "plugins": [
      [
        "expo-notifications",
        {
          "icon": "./local/assets/notification_icon.png",
          "color": "#ffffff",
          "defaultChannel": "default",
          "sounds": ["./local/assets/notification_sound.wav"],
          "enableBackgroundRemoteNotifications": false
        }
      ]
    ]
  }
}
```

Configurable properties:
- `icon` — Android only; 96x96 white PNG with transparency.
- `color` — Android only; tint color (default `#ffffff`).
- `defaultChannel` — Android only; default FCMv1 channel.
- `sounds` — Array of local sound file paths.
- `enableBackgroundRemoteNotifications` — iOS only; enables background remote notifications (default `false`).

### Push Token Management

**`getExpoPushTokenAsync(options)`** — Returns an Expo token for sending via Expo's push service. Requires `projectId`.

```ts
const expoPushToken = await Notifications.getExpoPushTokenAsync({
  projectId: 'your-project-id',
});
```

**`getDevicePushTokenAsync()`** — Returns native FCM (Android) / APNs (iOS) token for third-party services.

**`addPushTokenListener(listener)`** — Registers listener for push token changes (rare token-roll events).

### Scheduling

**`scheduleNotificationAsync(request)`** — Schedule for future delivery.

```ts
const id = await Notifications.scheduleNotificationAsync({
  content: {
    title: "Title",
    body: "Body text",
    data: { customData: "value" },
  },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
    seconds: 60,
    repeats: false,
  },
});
```

Trigger types (`Notifications.SchedulableTriggerInputTypes`):
- `TIME_INTERVAL` — delay in seconds with optional `repeats`. On iOS, repeating interval must be ≥ 60 seconds.
- `DATE` — specific JavaScript `Date`.
- `CALENDAR` — date-component matching (iOS).
- `DAILY` — hour/minute recurrence.
- `WEEKLY` — weekday + hour/minute.
- `MONTHLY` — day/hour/minute monthly.
- `YEARLY` — month/day/hour/minute annually.

Other scheduling methods:
- `cancelScheduledNotificationAsync(identifier)` — cancel one by ID.
- `cancelAllScheduledNotificationsAsync()` — cancel all.
- `getAllScheduledNotificationsAsync()` — fetch all scheduled requests.
- `getNextTriggerDateAsync(trigger)` — Unix timestamp (ms) of next trigger, or `null`.

To present immediately, pass `trigger: null`.

### Notification Handler (foreground presentation)

**`setNotificationHandler(handler)`** — Controls presentation while app is running. Handler must respond within **3 seconds** or the notification is discarded.

```ts
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowBanner: true,
    shouldShowList: true,
    shouldPlaySound: false,
    shouldSetBadge: false,
  }),
  handleSuccess: (notificationId) => {},
  handleError: (notificationId, error) => {},
});
```

Note (SDK 56): the handler return uses `shouldShowBanner` / `shouldShowList` (the older `shouldShowAlert` field was replaced in SDK 52+).

### Event Listeners

All listeners return an `EventSubscription` with a `.remove()` method. `removeNotificationSubscription(subscription)` also exists.

**`addNotificationReceivedListener(listener)`** — Fires when a notification arrives while app is running.

```ts
const subscription = Notifications.addNotificationReceivedListener(
  notification => {
    console.log(notification);
  }
);
```

**`addNotificationResponseReceivedListener(listener)`** — Fires on user interaction (tap, action button).

```ts
const subscription = Notifications.addNotificationResponseReceivedListener(
  response => {
    const { actionIdentifier } = response;
    const notification = response.notification;
  }
);
```

**`addNotificationsDroppedListener(listener)`** — Fires on Android when FCM drops notifications.

**`useLastNotificationResponse()`** — React hook returning the most recent response.

```ts
const lastResponse = Notifications.useLastNotificationResponse();
// undefined (loading) | null (none) | NotificationResponse
```

**`getLastNotificationResponseAsync()`** — Async equivalent of the hook.

**`clearLastNotificationResponseAsync()`** — Clears the last response.

### Background Notification Tasks

Requires `expo-task-manager`. Handles data-only (headless) notifications when app is backgrounded/terminated.

```ts
TaskManager.defineTask('BACKGROUND_NOTIFICATION', ({ data, error }) => {
  // Handle notification in background/terminated state
  return BackgroundNotificationResult.NoData;
});

Notifications.registerTaskAsync('BACKGROUND_NOTIFICATION');
```

- `registerTaskAsync(taskName)` / `unregisterTaskAsync(taskName)`.
- Requires headless payload (data only, no title/body). On iOS, set `_contentAvailable: true`.

### Permissions

**`getPermissionsAsync()`** — Current permission status. On iOS, inspect `ios.status`: `NOT_DETERMINED`, `DENIED`, `AUTHORIZED`, `PROVISIONAL`, `EPHEMERAL`.

**`requestPermissionsAsync(permissions)`**:

```ts
const status = await Notifications.requestPermissionsAsync({
  ios: {
    allowAlert: true,
    allowBadge: true,
    allowSound: true,
    allowCriticalAlerts: false,
    allowProvisional: false,
  },
});
```

### Badge Management

- `getBadgeCountAsync()` — current badge number (0 = none).
- `setBadgeCountAsync(badgeCount, options)` — returns boolean success.

### Presenting / Dismissing

- `dismissNotificationAsync(identifier)` — remove one from tray.
- `dismissAllNotificationsAsync()` — remove all.
- `getPresentedNotificationsAsync()` — notifications currently in tray (Android 6.0+, iOS).

### Android Notification Channels

**`setNotificationChannelAsync(channelId, channel)`**:

```ts
await Notifications.setNotificationChannelAsync('emails', {
  name: 'Email notifications',
  importance: Notifications.AndroidImportance.HIGH,
  sound: 'email_sound.wav',
  vibrationPattern: [0, 250, 250, 250],
  lightColor: '#FF231F7C',
  enableLights: true,
  enableVibrate: true,
  bypassDnd: false,
});
```

Other channel methods:
- `getNotificationChannelAsync(channelId)`
- `getNotificationChannelsAsync()`
- `deleteNotificationChannelAsync(channelId)`
- `setNotificationChannelGroupAsync(groupId, group)`
- `getNotificationChannelGroupAsync(groupId)` / `getNotificationChannelGroupsAsync()` / `deleteNotificationChannelGroupAsync(groupId)`

On Android 8.0+, channel config governs sound/vibration; duplicate settings on both the notification and channel for cross-version compatibility.

### Interactive Notifications (Categories)

**`setNotificationCategoryAsync(identifier, actions, options)`**:

```ts
await Notifications.setNotificationCategoryAsync('comments', [
  {
    identifier: 'reply',
    buttonTitle: 'Reply',
    options: { opensAppToForeground: true },
    textInput: {
      placeholder: 'Type your response...',
      submitButtonTitle: 'Send',
    },
  },
  {
    identifier: 'dismiss',
    buttonTitle: 'Dismiss',
    options: { isDestructive: true },
  },
], {
  previewPlaceholder: 'New comment',
  intentIdentifiers: [],
});
```

Reference the category from a notification:

```ts
await Notifications.scheduleNotificationAsync({
  content: {
    categoryIdentifier: 'comments',
    title: 'Comment reply',
    body: 'Someone commented',
  },
  trigger: null,
});
```

- `getNotificationCategoriesAsync()`
- `deleteNotificationCategoryAsync(identifier)`

### Custom Sounds

Configure local paths in the plugin, then reference by filename only:

```ts
await Notifications.scheduleNotificationAsync({
  content: { sound: 'custom_sound.wav' },
  trigger: { seconds: 2 },
});
```

On Android 8.0+, also set the channel sound:

```ts
await Notifications.setNotificationChannelAsync('emails', {
  sound: 'custom_sound.wav',
});
```

### Topic Subscription (Android)

- `subscribeToTopicAsync(topic)` / `unsubscribeFromTopicAsync(topic)` — FCM topic broadcast.

### Constants

- `Notifications.DEFAULT_ACTION_IDENTIFIER` = `'expo.modules.notifications.actions.DEFAULT'` — indicates the user tapped the notification body (not an action button).

### SDK 56 notes / caveats

- iOS push notifications are not available in Expo Go (dev build required) — applies since SDK 53. Local notifications still work in Expo Go.
- Android: launching from a notification in a debug build can fail to display the splash screen (~70% of the time); test in release mode (`npx expo run:android --variant release`).
- APNs entitlement is set to "development" by default; Xcode auto-switches to "production" in release archives.
- Foreground handler return shape uses `shouldShowBanner` / `shouldShowList` (replaced `shouldShowAlert`).

---

## 2. Push Notifications — Overview

Source: https://docs.expo.dev/push-notifications/overview/

Expo abstracts FCM (Android) and APNs (iOS) behind a single push service so Android and iOS notifications are handled uniformly. The guide flow:

1. **Setup foundation** — notification kinds and behaviors to understand first.
2. **Initial configuration** — obtaining push tokens and establishing credentials.
3. **Implementation** — calling the Expo Push Service API from your backend and responding to notification events in-app.
4. **Support** — FAQ for common questions.

A tutorial video covers configuring Firebase for FCMv1 on Android, establishing credentials through EAS, building, and testing with Expo's notification tools. A full doc index is available at `/llms.txt`.

---

## 3. Push Notifications — Setup

Source: https://docs.expo.dev/push-notifications/push-notifications-setup/

### Install

```sh
npx expo install expo-notifications expo-constants
```

`expo-notifications` handles permission requests and obtaining the `ExpoPushToken`; `expo-constants` retrieves `projectId` from app config.

### Configure

```json
{
  "expo": {
    "plugins": ["expo-notifications"]
  }
}
```

### `registerForPushNotificationsAsync` (full example, verbatim)

```tsx
async function registerForPushNotificationsAsync() {
  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('default', {
      name: 'default',
      importance: Notifications.AndroidImportance.MAX,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: '#FF231F7C',
    });
  }

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;
  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }
  if (finalStatus !== 'granted') {
    handleRegistrationError('Permission not granted to get push token for push notification!');
    return;
  }
  const projectId = Constants?.expoConfig?.extra?.eas?.projectId ?? Constants?.easConfig?.projectId;
  if (!projectId) {
    handleRegistrationError('Project ID not found');
  }
  try {
    const pushTokenString = (
      await Notifications.getExpoPushTokenAsync({
        projectId,
      })
    ).data;
    console.log(pushTokenString);
    return pushTokenString;
  } catch (e: unknown) {
    handleRegistrationError(`${e}`);
  }
}
```

The `projectId` attributes tokens to a specific project and remains stable across account changes.

### Testing workflow

1. Start the dev server.
2. Open the build on a physical device / supported emulator.
3. Copy the generated `ExpoPushToken`.
4. Send test messages via Expo's push notifications tool.

**Requirement:** push notifications work only on physical devices, Android Emulators with Google Play services, or iOS Simulators on Xcode 14+ (macOS 13+, iOS 16+).

---

## 4. Push Notifications — Sending

Source: https://docs.expo.dev/push-notifications/sending-notifications/

### Endpoints

- Send: `POST https://exp.host/--/api/v2/push/send`
- Receipts: `POST https://exp.host/--/api/v2/push/getReceipts`

### Headers

```text
host: exp.host
accept: application/json
accept-encoding: gzip, deflate
content-type: application/json
```

### Basic send

```sh
curl -H "Content-Type: application/json" -X POST "https://exp.host/--/api/v2/push/send" -d '{
  "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "title":"hello",
  "body": "world"
}'
```

### Message payload fields

Submit a single message object or an array of up to 100 messages for the same project.

Core (both platforms): `to` (string or array), `title`, `body`, `data` (object, ~4KB max).

iOS only: `subtitle`, `sound` ("default" or custom file), `badge`, `_contentAvailable` (boolean — background task execution), `interruptionLevel` ("active" | "critical" | "passive" | "time-sensitive"), `mutableContent` (boolean).

Android only: `channelId`, `icon` (drawable resource name), `tag` (replace displayed notifications).

Both: `ttl` (seconds for redelivery), `expiration` (Unix timestamp), `priority` ("default" | "normal" | "high"), `collapseId`, `categoryId`, `richContent` (`{ image }` URL).

### Batch send

```json
[
  { "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]", "sound": "default", "body": "Hello world!" },
  { "to": "ExponentPushToken[yyyyyyyyyyyyyyyyyyyyyy]", "badge": 1, "body": "You've got mail" },
  { "to": ["ExponentPushToken[zzzzzzzzzzzzzzzzzzzzzz]", "ExponentPushToken[aaaaaaaaaaaaaaaaaaaaaa]"], "body": "Breaking news!" }
]
```

### Push ticket response

```json
{
  "data": [
    { "status": "ok", "id": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX" },
    { "status": "ok", "id": "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY" }
  ]
}
```

Error ticket:

```json
{
  "data": [
    {
      "status": "error",
      "message": "\"ExponentPushToken[...]\" is not a registered recipient",
      "details": { "error": "DeviceNotRegistered" }
    }
  ]
}
```

### Checking receipts

```sh
curl -H "Content-Type: application/json" -X POST "https://exp.host/--/api/v2/push/getReceipts" -d '{
  "ids": [
    "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
  ]
}'
```

```json
{
  "data": {
    "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX": { "status": "ok" },
    "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY": { "status": "error", "message": "...", "details": {} }
  }
}
```

### Errors

Ticket: `DeviceNotRegistered` (stop sending to token).

Receipt: `DeviceNotRegistered`, `MessageTooBig` (>4096 bytes), `MessageRateExceeded` (use exponential backoff), `MismatchSenderId`, `InvalidCredentials`.

Request: `TOO_MANY_REQUESTS` (>600/sec per project), `PUSH_TOO_MANY_EXPERIENCE_IDS` (mixed project tokens), `PUSH_TOO_MANY_NOTIFICATIONS` (>100 messages), `PUSH_TOO_MANY_RECEIPTS` (>1000 IDs).

### Reliability best practices

- Max six simultaneous connections.
- Exponential backoff for HTTP 429/5xx.
- Check receipts ~15 min after sending (receipts cleared after 24h).
- Stop sending on `DeviceNotRegistered`.
- No guaranteed availability SLA.

### Enhanced security

Enable push security in the EAS Dashboard to require an access token:

```text
Authorization: Bearer ${accessToken}
```

Missing/invalid tokens return `UNAUTHORIZED`.

---

## 5. Push Credentials — FCM (Android) & APNs (iOS)

Source (FCM): https://docs.expo.dev/push-notifications/fcm-credentials/
Source (custom FCM/APNs): https://docs.expo.dev/push-notifications/sending-notifications-custom/

### FCM V1 (Android)

1. Create / use a Firebase project ([Firebase Console](https://console.firebase.google.com)).
2. **Service account key:** Project settings → Service accounts → "Generate New Private Key". Store the JSON securely; add to `.gitignore`.
3. **EAS configuration** via EAS CLI credentials workflow: `Android` → `production` → `Google Service Account` → choose the FCM V1 push management option → upload the service account JSON.
4. **google-services.json:** download from Firebase, place at project root (public; can be committed). Reference in `app.json`:

```json
{
  "expo": {
    "android": {
      "googleServicesFile": "./path/to/google-services.json"
    }
  }
}
```

5. **Existing service account:** grant the "Firebase Messaging API Admin" role via [IAM Admin](https://console.cloud.google.com/iam-admin/iam), then upload via EAS as above.

### Sending directly via FCM/APNs (bypassing Expo)

- Use `getDevicePushTokenAsync()` (not `getExpoPushTokenAsync()`) to obtain native tokens.
- **FCMv1:** obtain an OAuth 2.0 access token via Google Auth Library, then POST to Google's FCM endpoint with the Firebase project name and device token.
- **APNs:** requires the APNs entitlement in iOS config. Authorization uses a JWT generated from your `.p8` key, Key ID, and Apple Team ID, sent over HTTP/2 to Apple's sandbox or production servers.
- Expo notes this is not a comprehensive resource; defer to official Firebase/Apple docs.

---

## 6. expo-task-manager

Source: https://docs.expo.dev/versions/v56.0.0/sdk/task-manager/

Enables background task execution across Android, iOS, and tvOS. Included in Expo Go but with platform limitations.

### Install

```sh
npx expo install expo-task-manager
```

### Methods

- **`defineTask(taskName, taskExecutor)`** — Registers a task handler at global scope. Must NOT be called inside React lifecycle methods (background execution launches the JS engine independently).
- **`getRegisteredTasksAsync()`** — Promise resolving to an array of registered tasks with configs/options.
- **`getTaskOptionsAsync(taskName)`** — Returns options passed at registration, or `null` if task doesn't exist.
- **`isAvailableAsync()`** — API availability. Returns `false` on Android+Expo Go and on web; iOS background execution needs a dev build.
- **`isTaskDefined(taskName)`** — Synchronous check for a defined handler.
- **`isTaskRegisteredAsync(taskName)`** — Async check whether a task persists across sessions.
- **`unregisterTaskAsync(taskName)`** — Remove a single task (prefer module-specific methods like `Location.stopLocationUpdatesAsync`).
- **`unregisterAllTasksAsync()`** — Remove all tasks (useful at sign-out).

### Executor pattern

```js
TaskManager.defineTask(TASK_NAME, ({ data, error, executionInfo }) => {
  if (error) { /* handle error */ }
  if (data) { /* process data */ }
});
```

### Interfaces

- **TaskManagerTaskBody:** `data` (task payload), `error` (null | TaskManagerError), `executionInfo` (`eventId`, `taskName`, optional `appState` on iOS).
- **TaskManagerTask:** `taskName`, `taskType`, `options`.
- **TaskManagerError:** `code`, `message`.

---

## 7. expo-background-task

Source: https://docs.expo.dev/versions/v56.0.0/sdk/background-task/

Runs deferrable background tasks using platform-native schedulers — `WorkManager` (Android) and `BGTaskScheduler` (iOS) — optimizing battery. Integrates with `expo-task-manager` for JS execution. This is the SDK 56 replacement for the deprecated `expo-background-fetch` (see deprecation note below).

### Install

```sh
npx expo install expo-background-task
```

### iOS configuration (non-CNG / manual)

For Continuous Native Generation (CNG) projects this is applied automatically. Otherwise add to **Info.plist**:

```xml
<key>UIBackgroundModes</key>
<array>
  <string>processing</string>
</array>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
  <string>com.expo.modules.backgroundtask.processing</string>
</array>
```

### Methods

- **`getStatusAsync()`** → `Promise<BackgroundTaskStatus>` — API availability.
- **`registerTaskAsync(taskName, options?)`** — Register a task (defined via `TaskManager.defineTask()`). Tasks persist and restore on app init. `options`: `{ minimumInterval?: number }`.
- **`unregisterTaskAsync(taskName)`** — Cancel future executions.
- **`triggerTaskWorkerForTestingAsync()`** → `Promise<boolean>` — Simulate a system trigger (debug only).
- **`addExpirationListener(listener)`** (iOS only) — Notifies when the system interrupts background execution. Returns `{ remove: () => void }`.

### Types & enums

- **BackgroundTaskOptions:** `minimumInterval` (minutes between repeats; minimum 15, default 12 hours).
- **BackgroundTaskStatus:** `Restricted` (1), `Available` (2).
- **BackgroundTaskResult:** `Success` (1), `Failed` (2).

### vs. expo-background-fetch (deprecation note)

`expo-background-fetch` is deprecated and superseded by `expo-background-task` in SDK 56. `expo-background-task` uses native deferrable schedulers (`WorkManager` / `BGTaskScheduler`) for battery efficiency and integrates with `expo-task-manager`, whereas background-fetch focused on simple periodic network operations. Migrate to `expo-background-task`.

---

## 8. expo-device

Source: https://docs.expo.dev/versions/v56.0.0/sdk/device/

### Install

```sh
npx expo install expo-device
```

### Usage

```jsx
import * as Device from 'expo-device';

export default function App() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text>{Device.manufacturer}: {Device.modelName}</Text>
    </View>
  );
}
```

### Constants

| Constant | Type | Platforms | Notes |
|---|---|---|---|
| `brand` | `string \| null` | Android, iOS | Consumer brand; `null` on web |
| `designName` | `string \| null` | Android | Maps to `Build.DEVICE`; `null` on iOS/web |
| `deviceName` | `string \| null` | Android, iOS, tvOS, Web | User-set device name; iOS 16+ returns generic "iPhone" without entitlements |
| `deviceType` | `DeviceType \| null` | Android, iOS, tvOS, Web | Android by screen diagonal (3"–6.9" = PHONE, 7"–18" = TABLET) |
| `deviceYearClass` | `number \| null` | Android, iOS, tvOS, Web | device-year-class; `null` on web |
| `isDevice` | `boolean` | All | True for real devices; false for sim/emulator; always true on web |
| `manufacturer` | `string \| null` | Android, iOS, tvOS, Web | Actual hardware manufacturer |
| `modelId` | `any` | iOS | Internal id (e.g. "iPhone7,2"); `null` elsewhere |
| `modelName` | `string \| null` | Android, iOS, tvOS, Web | Human-friendly model name |
| `osBuildFingerprint` | `string \| null` | Android | Unique build identifier |
| `osBuildId` | `string \| null` | Android, iOS, tvOS, Web | `Build.DISPLAY` (Android) / `kern.osversion` (iOS) |
| `osInternalBuildId` | `string \| null` | Android, iOS, tvOS, Web | `Build.ID` (Android) / equals `osBuildId` (iOS) |
| `osName` | `string \| null` | All | "Android", "iOS", "iPadOS", or web browser OS |
| `osVersion` | `string \| null` | All | Human-readable OS version |
| `platformApiLevel` | `number \| null` | Android | Android SDK version; `null` elsewhere |
| `productName` | `string \| null` | Android | `Build.PRODUCT`; `null` elsewhere |
| `supportedCpuArchitectures` | `string[] \| null` | All | Supported processor architectures |
| `totalMemory` | `number \| null` | Android, iOS, tvOS, Web | Total RAM in bytes; `null` on web |

### Methods

- `getDeviceTypeAsync()` → `Promise<DeviceType>` — async version of `deviceType`.
- `getMaxMemoryAsync()` (Android) → `Promise<number>` — max Java VM memory (bytes).
- `getPlatformFeaturesAsync()` (Android) → `Promise<string[]>` — system feature names; `[]` on iOS/web.
- `getUptimeAsync()` (Android, iOS) → `Promise<number>` — ms since last reboot (excludes Android deep sleep).
- `hasPlatformFeatureAsync(feature)` (Android) → `Promise<boolean>`; always false on iOS/web.
- `isRootedExperimentalAsync()` (All) → `Promise<boolean>` — **unreliable** rooting/jailbreak detection; always false on web.
- `isSideLoadingEnabledAsync()` (Android) → `Promise<boolean>` — requires `REQUEST_INSTALL_PACKAGES` permission.

### DeviceType enum

`UNKNOWN` (0), `PHONE` (1), `TABLET` (2), `DESKTOP` (3), `TV` (4).

### Error codes

`ERR_DEVICE_ROOT_DETECTION` — thrown by `isRootedExperimentalAsync` on permission issues.

---

## 9. expo-application

Source: https://docs.expo.dev/versions/v56.0.0/sdk/application/

### Install

```sh
npx expo install expo-application
```

```js
import * as Application from 'expo-application';
```

### Constants

| Constant | Type | Platforms | Notes / Example |
|---|---|---|---|
| `applicationId` | `string \| null` | Android, iOS, tvOS, Web | Bundle ID (iOS) / application ID (Android); `null` on web. e.g. `"com.cocoacasts.scribbles"` |
| `applicationName` | `string \| null` | Android, iOS, tvOS, Web | Human-readable app name. e.g. `"Expo"`, `"Instagram"` |
| `nativeApplicationVersion` | `string \| null` | Android, iOS, tvOS, Web | User-facing store version. e.g. `"2.11.0"` |
| `nativeBuildVersion` | `string \| null` | Android, iOS, tvOS, Web | Internal build number. e.g. `"114"` |

### Methods

- `getAndroidId()` (Android) → `string` — hex device id, e.g. `"dd96dec43fb81c97"`.
- `getInstallationTimeAsync()` (Android, iOS, tvOS, Web) → `Promise<Date>` — initial install timestamp.
- `getInstallReferrerAsync()` (Android) → `Promise<string>` — Play Store referrer URL.
- `getIosApplicationReleaseTypeAsync()` (iOS) → `Promise<ApplicationReleaseType>`.
- `getIosIdForVendorAsync()` (iOS) → `Promise<string>` — IDFV, e.g. `"68753A44-4D6F-1226-9C60-0050E4C00067"`.
- `getIosPushNotificationServiceEnvironmentAsync()` (iOS) → `Promise<'development' | 'production' | null>`.
- `getLastUpdateTimeAsync()` (Android) → `Promise<Date>` — most recent Play Store update timestamp.

### ApplicationReleaseType enum (iOS)

`UNKNOWN`, `SIMULATOR`, `ENTERPRISE`, `DEVELOPMENT`, `AD_HOC`, `APP_STORE`.

---
