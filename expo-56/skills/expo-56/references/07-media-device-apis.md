# Expo SDK 56 — Redesigned Data/Device APIs

Knowledge base reference for SDK 56's redesigned data/device APIs (expo-calendar, expo-contacts, expo-media-library) plus expo-audio, expo-haptics, and expo-asset.

Sources:
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/sdk/calendar/
- https://docs.expo.dev/versions/v56.0.0/sdk/contacts/
- https://docs.expo.dev/versions/v56.0.0/sdk/media-library/
- https://docs.expo.dev/versions/v56.0.0/sdk/audio/
- https://docs.expo.dev/versions/v56.0.0/sdk/haptics/
- https://docs.expo.dev/versions/v56.0.0/sdk/asset/

---

## Overview: The Redesign (Changelog)

Source: https://expo.dev/changelog/sdk-56

In SDK 56, **expo-calendar**, **expo-contacts**, and **expo-media-library** received major redesigns that promote them from experimental to **stable** status. The redesign shares a common philosophy:

- **Object-oriented approach** — items like media assets or individual contacts are now represented as **classes** (instances with methods), instead of plain data objects passed to module-level functions.
- **Granular data fetching** — instead of loading entire objects at once, you can fetch only the **specific properties you need** (per-field getters / field selection), reducing overhead.
- **Builder pattern querying** — cleaner querying and filtering via a chainable Builder pattern.
- **Original APIs are now deprecated** in favor of the redesigned versions. Legacy functions remain importable from a `/legacy` subpath but throw at runtime if used directly from the main entry point.

The changelog points to a dedicated blog post for usage examples on the new MediaLibrary and Contacts APIs.

Other libraries in this domain:
- **expo-audio** — new `useAudioStream` hook for real-time microphone buffer access; new live-stream options/fields.
- **expo-haptics** — web haptics now work on Safari.
- **expo-asset** — supports GLB model assets for 3D / AR work.

---

## expo-calendar (SDK 56, redesigned)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/calendar/

The redesigned API centers on object-oriented entity classes. Each entity is returned as a class instance with its own methods, and individual items can be re-fetched via static `get(...)` methods.

### Class: `ExpoCalendar`

Represents a calendar. Properties include `id`, `title`, `color`, `allowsModifications`, and platform-specific attributes (`isPrimary`, `isSynced` on Android; `entityType`, `type` on iOS).

Methods:
- `createEvent(details)` → `Promise<ExpoCalendarEvent>`
- `createReminder(details)` → `Promise<ExpoCalendarReminder>`
- `listEvents(startDate, endDate)` → `Promise<ExpoCalendarEvent[]>`
- `listReminders(startDate, endDate, status?)` → `Promise<ExpoCalendarReminder[]>`
- `update(details)` → `Promise<void>`
- `delete()` → `Promise<void>`
- `get(calendarId)` — static, returns a calendar instance

### Class: `ExpoCalendarEvent`

Properties: `id`, `title`, `startDate`, `endDate`, `location`, `notes`, `alarms`, `recurrenceRule`, `allDay`, `availability`, `status`, and more.

Methods:
- `createAttendee(attendee)` → `Promise<ExpoCalendarAttendee>`
- `getAttendees()` → `Promise<ExpoCalendarAttendee[]>`
- `update(details)` → `Promise<void>`
- `delete()` → `Promise<void>`
- `editInCalendar(params?)` — launches OS UI
- `openInCalendar(params?)` — launches OS preview
- `getOccurrenceSync(recurringEventOptions?)` — returns event instance
- `get(eventId)` — static

### Class: `ExpoCalendarReminder`

Task/reminder objects. Properties: `title`, `dueDate`, `startDate`, `completed`, `completionDate`, `notes`, `location`, `alarms`, `recurrenceRule`.

Methods:
- `update(details)` → `Promise<void>`
- `delete()` → `Promise<void>`
- `get(reminderId)` — static

### Class: `ExpoCalendarAttendee`

Properties: `name`, `email`, `role`, `status`, `type`.

Methods:
- `update(details)` (Android only)
- `delete()` (Android only)

### Hooks

- `useCalendarPermissions(options?)`
- `useRemindersPermissions(options?)`

Both accept an optional `writeOnly` parameter for iOS write-only access.

### Module-level methods

- `Calendar.createCalendar(details?)` → `Promise<ExpoCalendar>`
- `Calendar.getCalendars(entityType?)` → `Promise<ExpoCalendar[]>`
- `Calendar.getDefaultCalendarSync()` → `ExpoCalendar`
- `Calendar.listEvents(calendars, startDate, endDate)` — supports multiple calendars
- `Calendar.presentPicker()` — iOS calendar selection
- `Calendar.getSourcesSync()` → `Source[]`
- Permissions: `getCalendarPermissions()`, `requestCalendarPermissions()`, `getRemindersPermissions()`, `requestRemindersPermissions()`

### Deprecated (legacy) APIs

All legacy async functions are deprecated, including:
`createCalendarAsync()`, `createEventAsync()`, `createReminderAsync()`, `getEventsAsync()`, `getRemindersAsync()`, `deleteEventAsync()`, `updateEventAsync()`, `editEventInCalendarAsync()`, `getAttendeesForEventAsync()`, `createAttendeeAsync()`.

Legacy methods can be imported from `"expo-calendar/legacy"` but will throw at runtime if used directly (from the main entry).

### Config plugin

Supports: `calendarPermission`, `remindersPermission`, `writeOnlyCalendarPermission`, and `writeOnlyAccess` (iOS 17+ write-only mode).

---

## expo-contacts (SDK 56, redesigned)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/contacts/

### Class: `Contact`

Primary class for contact management, replacing legacy callback-based methods.

Static methods:
- `Contact.create(contact)` — creates a new contact
- `Contact.getAll(options)` — retrieves all contacts with filtering/sorting
- `Contact.getAllDetails(fields, options)` — **granular bulk fetch**: returns only requested fields, avoiding full instances
- `Contact.getCount()` — total contact count
- `Contact.hasAny()` — whether any contacts exist
- `Contact.presentPicker()` — native contact selection UI
- `Contact.presentCreateForm(contact)` — native creation form
- `Contact.presentAccessPicker()` — iOS 18+ system access dialog

Instance methods (granular data retrieval — fetch only the field you need):
`getGivenName()`, `getFamilyName()`, `getEmails()`, `getPhones()`, `getAddresses()`, `getCompany()`, `getDepartment()`, `getImage()`, etc.

Instance methods (modification):
- `patch(contact)` — partial update (undefined fields ignored; lists entirely replaced)
- `update(contact)` — full replacement
- Field-specific modifiers: `addPhone()`, `addEmail()`, `deletePhone()`, `updateEmail()`, and setters like `setGivenName(...)`.

Example (verbatim):
```ts
const contact = await Contact.create({
  givenName: 'John',
  familyName: 'Doe',
  phones: [{ label: 'mobile', number: '+12123456789' }]
});
await contact.setGivenName('Andrew');
const phones = await contact.getPhones();
```

### Builder/Query pattern — `ContactQueryOptions`

`Contact.getAllDetails(fields, options)` uses field selection plus query options for flexible filtering/pagination/sorting:

```ts
const details = await Contact.getAllDetails(
  [ContactField.FULL_NAME, ContactField.PHONES],
  {
    limit: 20,
    offset: 10,
    sortOrder: ContactsSortOrder.GivenName,
    name: 'John'
  }
);
```

### Class: `Container` (iOS only)

Represents contact storage sources (iCloud, Google, Exchange).

Static: `Container.getAll()`, `Container.getDefault()`.
Instance: `getContacts()`, `getGroups()`, `getName()`, `getType()`.

### Class: `Group` (iOS only)

Contact grouping with membership management.

Static: `Group.create(name, containerId)`, `Group.getAll(containerId)`.
Instance: `getContacts(options)`, `addContact(contact)`, `removeContact(contact)`, `getName()`, `setName(name)`, `delete()`.

### Component: `ContactAccessButton` (iOS 18.0+)

React component enabling limited-access contact selection.
Props: `query`, `caption`, `ignoredEmails`, `ignoredPhoneNumbers`, `backgroundColor`, `textColor`, `tintColor`.

### Data types / enums

- `ContactField` — enum of retrievable properties: `GIVEN_NAME`, `FAMILY_NAME`, `PHONES`, `EMAILS`, `ADDRESSES`, `IMAGE`, `BIRTHDAY` (iOS), `NOTE`, `FULL_NAME`, etc.
- `ContactsSortOrder` — `GivenName`, `FamilyName`, `UserDefault`, `None`.
- `ContactPatch` — partial update type (undefined fields ignored; lists entirely replaced).
- `ContactDetails` — complete contact representation with all fields.

### Event subscriptions

`Contacts.addContactsChangeListener(listener)` — observe contact modifications. Platform timing: Android ~5–7 second delay via `ContentObserver`; iOS immediate via `CNContactStoreDidChangeNotification`.

### Deprecated (legacy) APIs

Marked deprecated (callback/async based): `Contacts.addContactAsync()`, `Contacts.getContactsAsync()`, `Contacts.updateContactAsync()`, `Contacts.removeContactAsync()`, group/container legacy methods, `Contacts.presentFormAsync()`, `Contacts.shareContactAsync()`.

Note: deprecated APIs must be imported from `expo-contacts/legacy`; they throw at runtime otherwise.

---

## expo-media-library (SDK 56, redesigned)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/media-library/

The redesigned API is built around three classes: **`Asset`**, **`Album`**, and **`Query`**. Available on Android, iOS, tvOS, and Expo Go.

### Class: `Asset`

Represents a single media file (image, video, or audio). Uses granular getters so you fetch only the metadata you need.

Static:
- `Asset.create(filePath, album?)` — create asset from a file path (replaces `saveToLibraryAsync`)
- `Asset.delete(assets[])` — batch deletion

Instance:
- `asset.delete()` — remove asset from device
- `asset.getFilename()` — filename with extension
- `asset.getMediaType()` — `MediaType` enum value
- `asset.getUri()` — system file URI
- `asset.getWidth()`, `asset.getHeight()` — pixel dimensions
- `asset.getCreationTime()`, `asset.getModificationTime()` — Unix timestamps (ms)
- `asset.getDuration()` — duration for audio/video
- `asset.getExif()` — EXIF metadata (images)
- `asset.getFavorite()`, `asset.setFavorite(boolean)` — favorite status
- `asset.getLocation()` — geographic coordinates
- `asset.getAlbums()` — containing albums array
- `asset.getInfo()` — complete `AssetInfo` object (replaces `getAssetInfoAsync`)

### Class: `Album`

Static:
- `Album.create(name, assetsRefs[], moveAssets?)` — create album
- `Album.get(title)` — retrieve by name
- `Album.getAll()` — all device albums
- `Album.delete([albums], deleteAssets?)` — batch deletion

Instance:
- `album.getTitle()`
- `album.getAssets()`
- `album.add(assets)` — add single or multiple assets
- `album.removeAssets(assets[])` — iOS only; removes without deleting
- `album.delete()`

### Class: `Query` (Builder pattern)

Chainable filtered asset retrieval:
- `eq(field, value)` — equality
- `gt(field, value)`, `gte(field, value)` — greater-than comparisons
- `lt(field, value)`, `lte(field, value)` — less-than comparisons
- `within(field, values[])` — array membership
- `album(albumRef)` — filter by album
- `orderBy(sortDescriptor | assetField)` — ordering
- `limit(n)`, `offset(n)` — pagination
- `exe()` — executes, returns `Promise<Asset[]>`

Example (verbatim):
```tsx
const assets = await new Query()
  .eq(AssetField.MEDIA_TYPE, MediaType.IMAGE)
  .lte(AssetField.HEIGHT, 1080)
  .orderBy(AssetField.CREATION_TIME)
  .limit(20)
  .exe();
```

### Enums / fields

- `AssetField`: `CREATION_TIME`, `MODIFICATION_TIME`, `MEDIA_TYPE`, `DURATION`, `WIDTH`, `HEIGHT`.
- `MediaType`: `IMAGE`, `VIDEO`, `AUDIO`, `UNKNOWN`.

### Hook: `usePermissions(options)`

```tsx
const [permissionResponse, requestPermission] = MediaLibrary.usePermissions({
  writeOnly: true,
  granularPermissions: ['photo']
});
```
Granular permissions: `'audio' | 'photo' | 'video'`.

### Module-level functions

- `requestPermissionsAsync(writeOnly?, granularPermissions?)`
- `getPermissionsAsync(writeOnly?, granularPermissions?)`
- `presentPermissionsPicker(mediaTypes?)` — iOS / Android 14+; update scoped access
- `addListener(callback)` — subscribe to library changes
- `removeAllListeners()`

### Key type: `AssetInfo`

```tsx
{
  id: string;
  filename: string;
  mediaType: MediaType;
  uri: string;
  width: number;
  height: number;
  duration: number | null;
  creationTime: number | null;
  modificationTime: number | null;
  isFavorite: boolean;
}
```

### Deprecated (legacy) APIs

Throw at runtime unless imported from `'expo-media-library/legacy'`:
`addAssetsToAlbumAsync`, `removeAssetsFromAlbumAsync`, `createAlbumAsync`, `deleteAlbumsAsync`, `createAssetAsync`, `deleteAssetsAsync`, `getAlbumAsync`, `getAlbumsAsync`, `getMomentsAsync`, `getAssetsAsync` (→ `Query` class), `getAssetInfoAsync` (→ `asset.getInfo()`), `setAssetFavoriteAsync` (→ `asset.setFavorite()`), `saveToLibraryAsync` (→ `Asset.create()`).

---

## expo-audio (SDK 56)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/audio/

### New hook: `useAudioStream` — real-time microphone buffer access

Signature:
```typescript
useAudioStream(options?: AudioStreamOptions): AudioStreamResult
```

`AudioStreamOptions`:
- `channels` — number of audio channels (1 = mono, 2 = stereo; default `1`)
- `encoding` — PCM encoding format: `'float32'` or `'int16'` (default `'float32'`)
- `sampleRate` — desired sample rate in Hz (default `48000`)
- `onBuffer` — optional callback receiving `AudioStreamBuffer` objects

`AudioStreamResult`:
- `stream` — the `AudioStream` instance
- `isStreaming` — boolean, whether capture is active

Example (verbatim):
```tsx
const result = useAudioStream({
  channels: 1,
  encoding: 'float32',
  sampleRate: 48000,
  onBuffer: (buffer) => console.log(buffer)
});
```

### Live-stream additions (changelog + AudioStatus)

New `AudioStatus` fields:
- `isLive` — whether the current audio source is a live stream with indefinite duration
- `currentOffsetFromLive` — seconds behind the live edge, or `null` if not a live stream
- `error` — playback error message, or `null` if no error

New mode/options:
- iOS: `isLiveStream` lock-screen option
- Android: `playsInSilentMode` option — whether audio playback is allowed when the device is in silent mode (default `true`)
- Related `AudioMode` option: `shouldPlayInBackground` — whether the audio session stays active in background (default `false`)

---

## expo-haptics (SDK 56)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/haptics/

### SDK 56 highlight: web haptics on Safari

Per the SDK 56 changelog, web haptics now work on **Safari**. On Web the library uses the **Web Vibration API** (browser support varies; the device must have vibration hardware, and some browsers may ignore vibration in certain contexts such as background tabs).

Supported platforms: Android, iOS, Web.

### Methods

- `Haptics.impactAsync(style)` → `Promise<void>` — `style` is optional `ImpactFeedbackStyle` (default Medium). Styles: `Light`, `Medium`, `Heavy`, `Rigid`, `Soft`.
- `Haptics.notificationAsync(type)` → `Promise<void>` — `type` is optional `NotificationFeedbackType` (default Success). Types: `Success`, `Warning`, `Error`.
- `Haptics.selectionAsync()` → `Promise<void>` — signals a selection change was registered.
- `Haptics.performAndroidHapticsAsync(type)` — Android only; `type` is an `AndroidHaptics` enum value.

### Android note

Docs recommend `performAndroidHapticsAsync` over `impactAsync`/`notificationAsync` on Android — it is similar to iOS haptic feedback and does **not** require the `VIBRATE` permission.

### Install

```sh
npx expo install expo-haptics
```

---

## expo-asset (SDK 56)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/asset/

### SDK 56 highlight: GLB model support for 3D / AR

`expo-asset` now supports 3D models in **GLB** format. The config plugin lists `.glb` among configurable asset file types ("3D Models: `.glb`"), enabling bundling/loading of GLB models for 3D / AR work.

Supported platforms: Android, iOS, tvOS, Web, Expo Go.

### Class: `Asset`

Properties:
- `localUri` — `file://` URI pointing to the local file on the device (after download)
- `uri` — remote asset location (Expo servers or dev CLI)
- `name`, `type` — asset filename components
- `width`, `height` — image dimensions (nullable for non-image assets)
- `downloaded` — boolean, download completion flag

Methods:
- `downloadAsync()` — downloads asset data to device cache; resolves to the `Asset` instance
- `loadAsync(moduleId)` — convenience wrapper combining `fromModule()` + `downloadAsync()`; accepts a single or array of `require()` calls or URLs
- `fromModule(virtualAssetModule)` — creates an `Asset` from a `require()` statement or external URL

### Hook: `useAssets`

```ts
const [assets, error] = useAssets([require('path/to/asset.jpg'), require('path/to/other.png')]);
```
Returns a tuple `[Asset[] | undefined, Error | undefined]`. Note: assets are not reloaded when the asset list changes dynamically.
