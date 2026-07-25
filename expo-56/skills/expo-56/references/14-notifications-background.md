# Notifications & Background Execution — Expo SDK 56 Reference

Knowledge base reference covering local/push notifications, background task execution, and device/application introspection for Expo SDK 56. Each section cites its source URL.

### Dependency pins (SDK 56)

From `git show origin/sdk-56:packages/expo/bundledNativeModules.json` — this file ships inside the `expo` package and is what `expo install` actually resolves against:

| Package | SDK 56 pin |
|---|---|
| `expo-notifications` | `~56.0.22` |
| `expo-task-manager` | `~56.0.23` |
| `expo-background-task` | `~56.0.23` |
| `expo-background-fetch` | `~56.0.23` (deprecated, still shipped) |
| `expo-device` | `~56.0.4` |
| `expo-application` | `~56.0.3` |
| `expo-constants` | `~56.0.22` |

> Do **not** read pins out of `docs/public/static/schemas/v56.0.0/native-modules.json` — that schema is frozen and stale (it still says `expo-notifications ~56.0.16`, `expo-task-manager ~56.0.17`). `bundledNativeModules.json` on the release branch is the only accurate source.

**SDK 56 patch drift.** Every patch above 56.0.13 in these packages is *"This version does not introduce any user-facing changes"* with exactly one exception: `expo-background-task@56.0.15` (2026-05-26) carries "[iOS] Fix precompiled XCFramework builds resolving the task service helper" ([#46188](https://github.com/expo/expo/pull/46188)) — a native build fix, no JS surface. Nothing else in this document has drifted within SDK 56.

### Corrections to prior memory (read first)

Four things a model reliably gets wrong in this domain:

1. The foreground handler returns **`shouldShowBanner` / `shouldShowList`**, not `shouldShowAlert` (replaced in SDK 53+ / `expo-notifications@0.31.0`, [#36361](https://github.com/expo/expo/pull/36361); SDK 52 shipped 0.29.14, which still only had `shouldShowAlert`).
2. Every non-`null` trigger object must carry a **`type`** (or `channelId`) discriminator. The pre-SDK-52 bare `{ seconds: n }` shape (valid through SDK 51) **throws** a `TypeError` at runtime.
3. **`removeNotificationSubscription` no longer exists** (nor `removePushTokenSubscription`). Call `subscription.remove()`.
4. On Android 13+ you must **create a notification channel before requesting a push token**, or the OS permission prompt never appears.

Two more that only bite at runtime:

5. `getLastNotificationResponseAsync` / `clearLastNotificationResponseAsync` are deprecated; the current APIs are **synchronous** (`getLastNotificationResponse()` / `clearLastNotificationResponse()`).
6. The background-notification result enum is **`BackgroundNotificationTaskResult`**, not `BackgroundNotificationResult` (upstream's own JSDoc example has this typo — do not copy it).

### Choosing a background API

All three sit on top of `expo-task-manager`; pick by trigger source:

| Need | Use |
|---|---|
| React to a data-only (headless) push | `Notifications.registerTaskAsync()` (section 1) |
| Periodic deferrable work the OS schedules | `expo-background-task` (section 7) |
| Legacy periodic fetch | `expo-background-fetch` — deprecated since the SDK 53 cycle, do not start new work here |
| Location / geofencing / other module-driven events | The owning module's own `start*Async`, with `TaskManager.defineTask` for the executor |

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
          "mode": "development",
          "enableBackgroundRemoteNotifications": false
        }
      ]
    ]
  }
}
```

Configurable properties (`packages/expo-notifications/plugin/src/withNotifications.ts`):
- `icon` — Android only; 96x96 white PNG with transparency.
- `color` — Android only; tint color (default `#ffffff`).
- `defaultChannel` — Android only; default FCMv1 channel.
- `sounds` — Array of local sound file paths.
- `mode` — iOS only; `'development' | 'production'` (default `'development'`). Writes the `aps-environment` entitlement — this is the knob behind the "APNs entitlement defaults to development" caveat below. The plugin only sets it when the entitlement is not already present.
- `enableBackgroundRemoteNotifications` — iOS only; adds `remote-notification` to `UIBackgroundModes` (default `false`). Required for headless/background notifications — see "Background Notification Tasks".

### Push Token Management

**`getExpoPushTokenAsync(options?)`** → `Promise<ExpoPushToken>` — Returns `{ type: 'expo', data: string }` for sending via Expo's push service.

```ts
const expoPushToken = (await Notifications.getExpoPushTokenAsync({
  projectId: 'your-project-id',
})).data;
```

`ExpoPushTokenOptions` (all optional): `projectId`, `applicationId`, `devicePushToken`, `deviceId`, `development` (iOS sandbox vs production APNs), `baseUrl`, `url`, `type`. `projectId` falls back to `Constants.easConfig?.projectId ?? Constants.expoConfig?.extra?.eas?.projectId`; if neither resolves the call throws `ERR_NOTIFICATIONS_NO_EXPERIENCE_ID`. `applicationId` falls back to `Application.applicationId`.

**`getDevicePushTokenAsync()`** → `Promise<DevicePushToken>` — native FCM (Android) / APNs (iOS) token for third-party services. **Returns an object, not a string:**

```ts
const { type, data } = await Notifications.getDevicePushTokenAsync();
// type: 'ios' | 'android' | 'web'; data: string on native (an object on web)
```

**`unregisterForNotificationsAsync()`** → `Promise<void>` — Unregisters the device from the push notification service.

**`setAutoServerRegistrationEnabledAsync(enabled)`** — Toggles automatic device-push-token registration with Expo's servers.

**`addPushTokenListener(listener)`** — Registers listener for push token changes (rare token-roll events). Do not call `getDevicePushTokenAsync` inside the listener — it retriggers the listener.

### Android permissions & ordering

Source: `docs/pages/versions/v56.0.0/sdk/notifications.mdx:431-445`.

- `RECEIVE_BOOT_COMPLETED` is added automatically by the library's **AndroidManifest.xml** (used to restore scheduled notifications after a reboot).
- Android 12+ (API 31): scheduling a notification that fires at an *exact* time requires `<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"/>` in **AndroidManifest.xml**.
- **Android 13+: the OS permission prompt will not appear until at least one notification channel exists.** `setNotificationChannelAsync` must be called *before* `getDevicePushTokenAsync` / `getExpoPushTokenAsync`. This is the usual cause of "no permission prompt / token request hangs on Android 13+".

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

Note (SDK 56): the handler return uses `shouldShowBanner` / `shouldShowList` (the older `shouldShowAlert` field was deprecated in SDK 53 / `expo-notifications@0.31.0`).

### Event Listeners

All listeners return an `EventSubscription`; unsubscribe with `subscription.remove()`. There is **no** `removeNotificationSubscription` / `removePushTokenSubscription` export — they were deprecated and have since been deleted (`packages/expo-notifications/src/index.ts`).

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

**`getLastNotificationResponse()`** → `NotificationResponse | null` — **Synchronous** non-hook equivalent.

**`clearLastNotificationResponse()`** → `void` — **Synchronous**; clears the last response (also clears the value returned by `useLastNotificationResponse`). Use after routing off a notification so the route is not re-selected.

> `getLastNotificationResponseAsync()` and `clearLastNotificationResponseAsync()` still exist but are `@deprecated` in SDK 56 — they are thin wrappers over the synchronous versions above.

### Background Notification Tasks

Requires `expo-task-manager`. Handles data-only (headless) notifications when app is backgrounded/terminated.

```ts
TaskManager.defineTask('BACKGROUND_NOTIFICATION', ({ data, error }) => {
  // Handle notification in background/terminated state
  return Notifications.BackgroundNotificationTaskResult.NoData;
});

Notifications.registerTaskAsync('BACKGROUND_NOTIFICATION');
```

- `registerTaskAsync(taskName)` / `unregisterTaskAsync(taskName)`.
- `BackgroundNotificationTaskResult` (iOS; mirrors `UIBackgroundFetchResult`): `NewData` (0), `NoData` (1), `Failed` (2). **The name is `BackgroundNotificationTaskResult`** — the upstream JSDoc example says `BackgroundNotificationResult`, which does not exist.
- Requires a headless payload: `data` only, **no `title` / `body`**. On iOS, set `_contentAvailable: true`.

**iOS build requirement** (`docs/pages/versions/v56.0.0/sdk/notifications.mdx:469-502`): `remote-notification` must be in `UIBackgroundModes`.
- CNG projects: set `enableBackgroundRemoteNotifications: true` on the config plugin; prebuild applies it.
- Non-CNG / bare iOS projects: add it to **`ios/project-name/Supporting/Expo.plist`** (not Info.plist):

```xml
<key>UIBackgroundModes</key>
<array>
  <string>remote-notification</string>
</array>
```

### Permissions

Both permission calls resolve to a `NotificationPermissionsStatus` **object** — `{ status, granted, canAskAgain, expires, ios?: { status, allowsAlert, ... } }` — not a bare status string.

**`getPermissionsAsync()`** — Current permission status. On iOS, rely on `ios.status` rather than the root `status`: `NOT_DETERMINED`, `DENIED`, `AUTHORIZED`, `PROVISIONAL`, `EPHEMERAL` (accessible via `Notifications.IosAuthorizationStatus`).

Canonical "may I notify at all?" check (from the source JSDoc):

```ts
export async function allowsNotificationsAsync() {
  const settings = await Notifications.getPermissionsAsync();
  return (
    settings.granted || settings.ios?.status === Notifications.IosAuthorizationStatus.PROVISIONAL
  );
}
```

**`requestPermissionsAsync(permissions?)`** — Defaults to requesting alert + badge + sound when called with no argument:

```ts
const settings = await Notifications.requestPermissionsAsync({
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

Configure local paths in the plugin, then reference by **base filename only** (no path):

```ts
await Notifications.setNotificationChannelAsync('new_emails', {
  name: 'E-mail notifications',
  importance: Notifications.AndroidImportance.HIGH,
  sound: 'custom_sound.wav', // base filename only
});

await Notifications.scheduleNotificationAsync({
  content: {
    title: "You've got mail!",
    sound: 'custom_sound.wav', // base filename only
  },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
    seconds: 2,
    channelId: 'new_emails',
  },
});
```

On Android 8.0+ the **channel** governs the sound, so the `channelId` on the trigger is what actually routes the notification to the custom-sound channel — setting `content.sound` alone is not enough.

**Gotcha:** every non-`null` `trigger` object must contain a `type` or a `channelId` entry. The legacy bare `{ seconds: 2 }` shape throws:

```text
TypeError: The `trigger` object you provided is invalid. It needs to contain a `type` or `channelId` entry.
```

(`packages/expo-notifications/src/hasValidTriggerObject.ts`, thrown from `parseTrigger()`.) Passing `undefined` throws a different `TypeError` — use an explicit `null` to present immediately.

### Topic Subscription (Android)

- `subscribeToTopicAsync(topic)` / `unsubscribeFromTopicAsync(topic)` — FCM topic broadcast.

### Constants

- `Notifications.DEFAULT_ACTION_IDENTIFIER` = `'expo.modules.notifications.actions.DEFAULT'` — indicates the user tapped the notification body (not an action button).

### SDK 56 notes / caveats

- Push (remote) notifications provided by `expo-notifications` are **unavailable in Expo Go from SDK 53 onward** — a development build is required. The SDK reference states this for Android explicitly (`docs/pages/versions/v56.0.0/sdk/notifications.mdx:32`); the push FAQ states it generally ("In SDK 53 and later, Expo Go does not support push notifications functionality", `docs/pages/push-notifications/faq.mdx`). Local / in-app notifications still work in Expo Go. Importing `expo-notifications` in Expo Go logs a warning.
- Android: launching from a notification in a debug build can fail to display the splash screen (~70% of the time); test in release mode (`npx expo run:android --variant release`).
- APNs entitlement (`aps-environment`) is set to `development` by default; override with the plugin's `mode` prop. Xcode auto-switches to "production" in release archives.
- Foreground handler return shape uses `shouldShowBanner` / `shouldShowList` (replaced `shouldShowAlert` back in SDK 53).

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

**`richContent` caveat:** Android renders the image out of the box; **on iOS you must add a Notification Service Extension target** to the app for it to show.

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

**Gotcha — silent auto-unregistration.** If the native side requests execution of a task whose executor is not defined in the JS bundle, `TaskManager` logs `Execution of "<name>" was requested but looks like it is not defined ... Make sure that "TaskManager.defineTask" is called during initialization phase` **and then calls `unregisterTaskAsync(taskName)` itself** — the task is silently gone until you register it again. This is the failure mode when `defineTask` is not reached at module scope early enough. In Expo Router projects, put `defineTask` calls in a separate file (for example **tasks.ts**) and import that file at the top of **app/\_layout.tsx** so it runs before any navigation.

### Interfaces

- **TaskManagerTaskBody:** `data` (task payload), `error` (null | TaskManagerError), `executionInfo` (`eventId`, `taskName`, optional `appState` on iOS).
- **TaskManagerTask:** `taskName`, `taskType`, `options`.
- **TaskManagerError:** `code`, `message`.

---

## 7. expo-background-task

Source: https://docs.expo.dev/versions/v56.0.0/sdk/background-task/

Runs deferrable background tasks using platform-native schedulers — `WorkManager` (Android) and `BGTaskScheduler` (iOS) — optimizing battery. Integrates with `expo-task-manager` for JS execution. This is the replacement for the deprecated `expo-background-fetch` (see deprecation note below).

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

### Gotchas

From `packages/expo-background-task/src/BackgroundTask.ts`:

- **`registerTaskAsync` throws if the task was not defined first:** `Error: Task '<name>' is not defined. You must define a task using TaskManager.defineTask before registering.` Call `TaskManager.defineTask` at module scope before registering.
- **Registration silently no-ops when the status is `Restricted`.** It logs one `console.warn` — the message branches on `Platform.OS`: on iOS `Background tasks are not supported on iOS simulators. Skipped registering task: <name>.`, everywhere else `Background tasks are not available in the current environment. Skipped registering task: <name>.` — then returns. Registration *appears* to succeed but nothing ever runs. This is the number-one reason background tasks "do nothing" during development: use a physical device. (The warning fires at most once per process.)
- **`getStatusAsync()` is hardcoded to return `Restricted` in Expo Go.** Background tasks require a development build.
- `registerTaskAsync` returns early (no-op) if the task is already registered; `unregisterTaskAsync` returns early if it is not.

### vs. expo-background-fetch (deprecation note)

`expo-background-fetch` was marked deprecated in favour of `expo-background-task` back in the **SDK 53 cycle** (`expo-background-fetch@13.1.0`, 2025-04-04, [#35817](https://github.com/expo/expo/pull/35817)) — not in SDK 56. It is **still shipped** in both SDK 56 and SDK 57 with `isDeprecated: true` and the warning "is not receiving patches and will be removed in an upcoming release"; it has not been removed. `expo-background-task` uses native deferrable schedulers (`WorkManager` / `BGTaskScheduler`) for battery efficiency and integrates with `expo-task-manager`, whereas background-fetch focused on simple periodic network operations. Migrate to `expo-background-task`.

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
- `getIosIdForVendorAsync()` (iOS) → `Promise<string | null>` — IDFV, e.g. `"68753A44-4D6F-1226-9C60-0050E4C00067"`. Can be `null`.
- `getIosPushNotificationServiceEnvironmentAsync()` (iOS) → `Promise<'development' | 'production' | null>` — reads the `aps-environment` entitlement. This is the value `getExpoPushTokenAsync`'s `development` option defaults to, and it pairs with the `expo-notifications` plugin's `mode` prop.
- `getLastUpdateTimeAsync()` (Android) → `Promise<Date>` — most recent Play Store update timestamp.

### ApplicationReleaseType enum (iOS)

`UNKNOWN`, `SIMULATOR`, `ENTERPRISE`, `DEVELOPMENT`, `AD_HOC`, `APP_STORE`.

---

## SDK 57 delta

**This domain has zero API changes in SDK 57.** Everything above applies verbatim to SDK 57; only the version pins move.

Verified 2026-07-25 against the **release branches** `origin/sdk-56` and `origin/sdk-57` (not `main`, which is SDK 58 in progress):

- `git diff --stat origin/sdk-56 origin/sdk-57 -- packages/{expo-notifications,expo-task-manager,expo-background-task,expo-background-fetch,expo-device,expo-application}/src` and `.../plugin/src` returns **empty output** — not one TypeScript or config-plugin byte differs between the two release branches.
- The generated API data is **byte-identical** for all six packages: `origin/sdk-56:docs/public/static/data/v56.0.0/<pkg>.json` vs `origin/sdk-57:docs/public/static/data/unversioned/<pkg>.json` hash the same. Same exported members, kinds, enum values and signatures.
- The `## 57.0.0 — 2026-07-08` section of `origin/sdk-57:CHANGELOG.md` contains **no** `expo-notifications`, `expo-task-manager`, `expo-background-task`, `expo-background-fetch`, `expo-device` or `expo-application` heading.
- Docs: on `origin/sdk-57` the SDK 57 pages live under `docs/pages/versions/**unversioned**/sdk/` — `versions/v57.0.0/sdk/` holds only the `ui` subtree. `diff` of `origin/sdk-56:.../v56.0.0/sdk/<pkg>.mdx` against `origin/sdk-57:.../unversioned/sdk/<pkg>.mdx` yields **only** the `sourceCodeUrl` frontmatter line (`tree/sdk-56` → `tree/main`) for all six packages. (The `versions/v57.0.0/sdk/*.mdx` files you see on `main` are a later snapshot and carry post-57 docs edits — do not diff against those.)
- The unversioned `/push-notifications/*` guides (sections 2-5 above) have no 56/57 split at all — they are a single set of pages. No 57 treatment needed.

### New in 57

Two native-only changes, both genuinely 57-only (checked against `origin/sdk-56`'s changelogs — neither was backported):

- `expo-task-manager@57.0.3` (2026-07-15) — "[iOS] Fix a crash when a task execution request is evaluated re-entrantly, by making its completion callback fire exactly once" ([#47594](https://github.com/expo/expo/pull/47594)). The single changed file is `packages/expo-task-manager/ios/EXTaskManager/EXTaskExecutionRequest.m`. **No 56.x backport** — `origin/sdk-56:packages/expo-task-manager/CHANGELOG.md` has no `#47594`, so SDK 56 iOS apps still carry this crash. This is the one real reason in this domain to move to 57.
- `expo-application@57.0.2` (2026-07-17) — "[iOS] Expose the embedded provisioning profile's `expirationDate`" ([#47190](https://github.com/expo/expo/pull/47190)). Native-internal only: it exists solely in `ios/EXApplication/EXProvisioningProfile.{h,m}`. There is **no TypeScript export** and no entry in the generated `expo-application.json` API data. Do not try to call it from JS.

Those two files plus `android/build.gradle` and `package.json` version bumps are the *entire* 56 → 57 native diff for these six packages.

Explicitly *not* deltas, despite plausible-sounding leads:

- **The iOS UIKit scene-lifecycle rework (`SceneDelegate.swift`, `ExpoAppSceneDelegate`, `UIApplicationSceneManifest`) is not in SDK 57 at all** — it landed after the 57 cut. On `origin/sdk-57`, `git grep -l SceneDelegate -- packages/ templates/` matches only `expo-updates` e2e fixtures and a `Platform.h` comment; no template and no app delegate uses it. Either way it would not touch `expo-notifications`, whose only app-delegate hook is `NotificationsAppDelegateSubscriber.swift` (`didRegisterForRemoteNotificationsWithDeviceToken` / `didFailToRegisterForRemoteNotificationsWithError` / `didReceiveRemoteNotification:fetchCompletionHandler:`), unchanged between `sdk-56` and `sdk-57`.
- **`expo-background-fetch` is not removed in SDK 57.** The SDK 57 page still carries `isDeprecated: true` with the same "will be removed in an upcoming release" warning, and `bundledNativeModules.json` on `origin/sdk-57` still pins it. There is no forced 56 → 57 migration for any package in this file.
- The Android release-build switch to `proguard-android-optimize.txt` (R8-on-by-default) is **not** in SDK 57 — `origin/sdk-57:templates/expo-template-bare-minimum/android/app/build.gradle` still uses `getDefaultProguardFile("proguard-android.txt")`. It landed after the 57 cut. (Both `expo-notifications` and `expo-task-manager` ship `consumerProguardFiles` keep rules regardless, so neither needs action whenever it does land.)

### Version pins (56 → 57)

From `packages/expo/bundledNativeModules.json` on each release branch. These are **not** flat `~57.0.0` — every package has already taken patches:

| Package | SDK 56 | SDK 57 |
|---|---|---|
| `expo-notifications` | `~56.0.22` | `~57.0.7` |
| `expo-task-manager` | `~56.0.23` | `~57.0.6` |
| `expo-background-task` | `~56.0.23` | `~57.0.6` |
| `expo-background-fetch` | `~56.0.23` | `~57.0.6` |
| `expo-device` | `~56.0.4` | `~57.0.1` |
| `expo-application` | `~56.0.3` | `~57.0.2` |
| `expo-constants` | `~56.0.22` | `~57.0.7` |

All fourteen versions confirmed published on npm as of 2026-07-25. Ignore `docs/public/static/schemas/v57.0.0/native-modules.json`, which is frozen and reports `~57.0.0` across the board.

---
