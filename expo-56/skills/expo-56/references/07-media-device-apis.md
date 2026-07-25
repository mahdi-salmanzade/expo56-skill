# Expo SDK 56 — Redesigned Data/Device APIs

Knowledge base reference for SDK 56's redesigned data/device APIs (expo-calendar, expo-contacts, expo-media-library) plus expo-audio, expo-haptics, and expo-asset.

Sources (verified 2026-07-25 against `origin/sdk-56` @ `c2f04619` and `origin/sdk-57` @ `47851a94`):
- https://expo.dev/changelog/sdk-56
- https://docs.expo.dev/versions/v56.0.0/sdk/calendar/ — `packages/expo-calendar/src/Calendar.ts`, `src/ExpoCalendar.types.ts`, `plugin/src/withCalendar.ts`
- https://docs.expo.dev/versions/v56.0.0/sdk/contacts/ — `packages/expo-contacts/src/types/{Contact,Contact.props,Group,Container}.ts`, `src/ContactsModule.ts`, `src/ContactAccessButton.tsx`
- https://docs.expo.dev/versions/v56.0.0/sdk/media-library/ — `packages/expo-media-library/src/index.ts`, `src/types/{Asset,Album,Query,AssetField,AssetMetadata,AssetInfo}.ts`
- https://docs.expo.dev/versions/v56.0.0/sdk/audio/ — `packages/expo-audio/src/AudioStream.ts`, `src/AudioStream.types.ts`, `src/Audio.types.ts`
- https://docs.expo.dev/versions/v56.0.0/sdk/haptics/ — `packages/expo-haptics/src/Haptics.types.ts`, `src/ExpoHaptics.web.ts`
- https://docs.expo.dev/versions/v56.0.0/sdk/asset/ — `packages/expo-asset/src/Asset.ts`, `src/AssetHooks.ts`, `plugin/src/withAssets.ts`

Version pins for this file, from `packages/expo/bundledNativeModules.json` on `origin/sdk-56` — that file ships inside the `expo` package and is what `expo install` actually resolves against: `expo-calendar ~56.0.9`, `expo-contacts ~56.0.11`, `expo-media-library ~56.0.10`, `expo-audio ~56.0.13`, `expo-haptics ~56.0.3`, `expo-asset ~56.0.21`. (Each matches the highest `56.x` on npm.) Do **not** use `docs/public/static/schemas/v56.0.0/native-modules.json` — it is frozen at an older commit and under-reports every one of these except `expo-haptics`. See the [SDK 57 delta](#sdk-57-delta) at the end for the 57 pins.

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

### Legacy → new API cheat sheet (read this first)

The legacy names still *exist* on the root import but **throw immediately**. The runtime message is (from `packages/expo-calendar/src/legacyWarnings.ts`):

```
Method createEventInCalendarAsync imported from "expo-calendar" is deprecated.
Import the legacy API from "expo-calendar/legacy" or migrate to the new object-oriented API from "expo-calendar".
API reference and migration examples are available in the calendar docs: https://docs.expo.dev/versions/latest/sdk/calendar/
```

`expo-media-library` emits the same message with "class-based API" instead of "object-oriented API". If you see it, you wrote SDK ≤55 code.

| Legacy call (throws from root import) | SDK 56 replacement |
| --- | --- |
| `Calendar.getEventsAsync()` | `Calendar.listEvents(calendars, start, end)` / `calendar.listEvents(start, end)` |
| `Calendar.createEventAsync()` | `calendar.createEvent(details)` |
| `Calendar.createEventInCalendarAsync()` | `calendar.addEventWithForm(options?)` |
| `Calendar.editEventInCalendarAsync()` | `event.editInCalendar(params?)` |
| `Calendar.openEventInCalendarAsync()` | `event.openInCalendar(params?)` |
| `Contacts.getContactsAsync()` | `Contact.getAll(options?)` / `Contact.getAllDetails(fields, options?)` |
| `Contacts.addContactAsync()` | `Contact.create(contact)` |
| `Contacts.updateContactAsync()` | `contact.update(...)` / `contact.patch(...)` |
| `Contacts.presentFormAsync()` | `Contact.presentCreateForm(...)` / `contact.editWithForm(...)` |
| `MediaLibrary.getAssetsAsync()` | `new Query()....exe()` |
| `MediaLibrary.getAssetInfoAsync()` | `asset.getInfo()` |
| `MediaLibrary.saveToLibraryAsync()` | `Asset.create(filePath, album?)` |
| `MediaLibrary.setAssetFavoriteAsync()` | `asset.setFavorite(bool)` |

Escape hatch for all three: `import * as Calendar from 'expo-calendar/legacy'` (same for `expo-contacts/legacy`, `expo-media-library/legacy`).

---

## expo-calendar (SDK 56, redesigned)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/calendar/

Platforms: `['ios*', 'android*']` — **not supported in Expo Go or Snack**. You must create a development build (`docs/pages/versions/v56.0.0/sdk/calendar.mdx:6,18`). Do not generate Expo Go calendar demos.

The redesigned API centers on object-oriented entity classes. Each entity is returned as a class instance with its own methods, and individual items can be re-fetched via static `get(...)` methods.

### Class: `ExpoCalendar`

Represents a calendar. Properties include `id`, `title`, `color`, `allowsModifications`, and platform-specific attributes (`isPrimary`, `isSynced` on Android; `entityType`, `type` on iOS).

Methods:
- `createEvent(details)` → `Promise<ExpoCalendarEvent>`
- `createReminder(details)` → `Promise<ExpoCalendarReminder>` — **iOS only**, throws `UnavailabilityError` elsewhere
- `listEvents(startDate, endDate)` → `Promise<ExpoCalendarEvent[]>`
- `listReminders(startDate, endDate, status?)` → `Promise<ExpoCalendarReminder[]>` — **iOS only**, throws `UnavailabilityError` elsewhere
- `addEventWithForm(options?: AddEventWithFormOptions)` → `Promise<DialogEventResult>` — launches the OS "add event" form. This is the replacement for the deprecated `createEventInCalendarAsync()`. iOS landed in 56.0.0; **Android landed in 56.0.7** (#46004), so pin `>=56.0.7` if you need it cross-platform.
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

### Class: `ExpoCalendarReminder` (iOS only)

Task/reminder objects. All three methods below throw `UnavailabilityError` on non-iOS platforms (56.0.9, #46416). Properties: `title`, `dueDate`, `startDate`, `completed`, `completionDate`, `notes`, `location`, `alarms`, `recurrenceRule`.

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

- `useCalendarPermissions(options?: PermissionHookOptions<{ writeOnly?: boolean }>)`
- `useRemindersPermissions(options?: PermissionHookOptions<object>)` — **iOS only**

Only `useCalendarPermissions` takes `writeOnly`; `useRemindersPermissions` has **no** `writeOnly` option (`Calendar.ts:407-441`). On Android/web `useRemindersPermissions()` deliberately returns a denied response — `{ granted: false, status: PermissionStatus.DENIED, canAskAgain: false, expires: 'never' }` — instead of throwing, so it needs no try/catch (56.0.9, #46416).

At module level the flag is **positional**, not an options object: `requestCalendarPermissions(writeOnly?: boolean)` and `getCalendarPermissions(writeOnly?: boolean)`.

### Module-level methods

- `Calendar.createCalendar(details?)` → `Promise<ExpoCalendar>`
- `Calendar.getCalendars(entityType?)` → `Promise<ExpoCalendar[]>`
- `Calendar.getDefaultCalendarSync()` → `ExpoCalendar` — **iOS only**; throws `UnavailabilityError` on Android. Android substitute: `getCalendars()` and pick a writable calendar (`isPrimary` helps identify per-account primaries).
- `Calendar.listEvents(calendars: (string | ExpoCalendar)[], startDate, endDate)` — accepts calendar IDs or instances, mixed
- `Calendar.presentPicker()` → `Promise<ExpoCalendar | null>` — **iOS only**; resolves `null` when the picker is cancelled
- `Calendar.getSourcesSync()` → `Source[]` — **iOS only**; throws `UnavailabilityError` on Android. Android substitute: read each calendar's `source` field from `getCalendars()`.
- Permissions: `getCalendarPermissions(writeOnly?)`, `requestCalendarPermissions(writeOnly?)`, `getRemindersPermissions()` (iOS only, throws elsewhere), `requestRemindersPermissions()` (iOS only, throws elsewhere)

Example — permission → calendar → event:

```ts
import * as Calendar from 'expo-calendar';

async function addStandup() {
  const { granted } = await Calendar.requestCalendarPermissions(); // pass true for write-only (iOS)
  if (!granted) return;
  const calendars = await Calendar.getCalendars(Calendar.EntityTypes.EVENT);
  const writable = calendars.find((c) => c.allowsModifications);
  if (!writable) return;
  await writable.createEvent({
    title: 'Standup',
    startDate: new Date(),
    endDate: new Date(Date.now() + 30 * 60 * 1000),
  });
}
```

### Deprecated (legacy) APIs

All legacy async functions are deprecated, including:
`createCalendarAsync()`, `createEventAsync()`, `createReminderAsync()`, `getEventsAsync()`, `getRemindersAsync()`, `deleteEventAsync()`, `updateEventAsync()`, `createEventInCalendarAsync()` (→ `calendar.addEventWithForm()`), `editEventInCalendarAsync()` (→ `event.editInCalendar()`), `openEventInCalendarAsync()` (→ `event.openInCalendar()`), `getAttendeesForEventAsync()`, `createAttendeeAsync()`.

Legacy methods can be imported from `"expo-calendar/legacy"` but will throw at runtime if used directly (from the main entry).

### Config plugin

Props (all iOS; `plugin/src/withCalendar.ts:9-36`):
- `calendarPermission?: string | false` — `NSCalendarsUsageDescription`
- `writeOnlyCalendarPermission?: string | false` — `NSCalendarsWriteOnlyAccessUsageDescription`; only used when `writeOnlyAccess` is `true`
- `remindersPermission?: string | false` — `NSRemindersUsageDescription` / `NSRemindersFullAccessUsageDescription`
- `writeOnlyAccess?: boolean` (default `false`) — iOS 17+ write-only mode. When `true` the plugin **swaps** `NSCalendarsFullAccessUsageDescription` for `NSCalendarsWriteOnlyAccessUsageDescription` (`withCalendar.ts:43-52`).

**Write-only flow (iOS):** to add events without reading calendar data, set `writeOnlyAccess: true` in the config plugin *and* pass `true` to `requestCalendarPermissions(true)`. That is enough for `ExpoCalendar.createEvent`. Anything that **reads** — `getCalendars()`, `listEvents()`, `presentPicker()` — still requires full calendar access.

---

## expo-contacts (SDK 56, redesigned)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/contacts/

Platforms: `['android', 'ios', 'expo-go']` — unlike `expo-calendar`, this **does** work in Expo Go.

### Class: `Contact`

Primary class for contact management, replacing legacy callback-based methods.

Static methods:
- `Contact.create(contact: CreateContactRecord)` → `Promise<Contact>`
- `Contact.getAll(options?: ContactQueryOptions)` → `Promise<Contact[]>`
- `Contact.getAllDetails<T extends readonly ContactField[]>(fields: T, options?: ContactQueryOptions)` → `Promise<PartialContactDetails<T>[]>` — **granular bulk fetch**: returns only requested fields, avoiding full instances
- `Contact.getCount()` → `Promise<number>`
- `Contact.hasAny()` → `Promise<boolean>`
- `Contact.presentPicker()` → `Promise<Contact | null>` — native contact selection UI
- `Contact.presentCreateForm(contact?: CreateContactRecord, options?: CreateFormOptions)` → `Promise<boolean>` — **both params optional**; resolves `true` if a contact was actually created
- `Contact.presentAccessPicker()` → `Promise<Contact[]>` — iOS 18+ system access dialog

Instance — granular fetch:
- `contact.getDetails<T extends readonly ContactField[]>(fields?: T)` → `Promise<PartialContactDetails<T>>` — the per-instance counterpart of the static bulk fetch, and what the docs use in their main examples. Omit `fields` for everything.
  ```ts
  const d = await contact.getDetails([ContactField.GIVEN_NAME, ContactField.PHONES]);
  ```

Instance — per-field accessors. Rather than memorising ~60 signatures, apply the **naming rule** (`src/types/Contact.ts:119-520`):
- Scalar fields: `getX(): Promise<string | null>` / `setX(value: string | null): Promise<boolean>` — e.g. `getGivenName`/`setGivenName`, `getFamilyName`, `getMiddleName`, `getMaidenName`, `getNickname`, `getPrefix`, `getSuffix`, `getCompany`, `getDepartment`, `getJobTitle`, `getNote`, `getImage`, `getPhoneticGivenName`/`MiddleName`/`FamilyName`, `getPhoneticCompanyName`.
- Read-only scalars: `getFullName(): Promise<string>`, `getThumbnail(): Promise<string | null>`.
- Booleans/objects follow the same shape: `getIsFavourite()/setIsFavourite(bool)` (British spelling), `getBirthday()/setBirthday(ContactDate | null)`, `getNonGregorianBirthday()/setNonGregorianBirthday(...)`.
- List fields get a four-method set: `addX(new): Promise<string>` (returns the new id), `getXs(): Promise<Existing[]>`, `deleteX(existing): Promise<void>`, `updateX(updated): Promise<void>`. The list fields are **Email, Phone, Date, ExtraName, Address, Relation, UrlAddress, SocialProfile, ImAddress** — so `addEmail`/`getEmails`/`deleteEmail`/`updateEmail`, `addPhone`/`getPhones`/…, and so on.

Instance methods (modification / UI):
- `patch(contact: ContactPatch)` → `Promise<void>` — partial update (undefined fields ignored; lists entirely replaced)
- `update(contact: CreateContactRecord)` → `Promise<void>` — full replacement
- `delete()` → `Promise<void>`
- `editWithForm(options?: FormOptions)` → `Promise<boolean>` — iOS; opens the system edit form for this contact

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

Full `ContactQueryOptions` (`src/types/Contact.props.ts:53-75`):
```ts
{
  limit?: number;
  offset?: number;
  sortOrder?: ContactsSortOrder;
  name?: string;      // substring match on name
  rawContacts?: boolean; // iOS only, default false — include raw contact data
}
```

### Class: `Container` (iOS only)

Represents contact storage sources (iCloud, Google, Exchange).

Static: `Container.getAll(): Promise<Container[]>`, `Container.getDefault(): Promise<Container | null>`.
Instance: `getContacts(): Promise<Contact[]>` (takes **no** options, unlike `Group.getContacts`), `getGroups(): Promise<Group[]>`, `getName(): Promise<string | null>`, `getType(): Promise<ContainerType | null>`.

### Class: `Group` (iOS only)

Contact grouping with membership management.

Static: `Group.create(name: string, containerId?: string): Promise<Group>`, `Group.getAll(containerId?: string): Promise<Group[]>` — `containerId` is **optional** on both.
Instance: `getContacts(options?: ContactQueryOptions): Promise<Contact[]>`, `addContact(contact: Contact): Promise<void>`, `removeContact(contact: Contact): Promise<void>`, `getName(): Promise<string | null>`, `setName(name: string): Promise<void>`, `delete(): Promise<void>`.

### Component: `ContactAccessButton` (iOS 18.0+)

React component enabling limited-access contact selection. **Guard it**: `ContactAccessButton.isAvailable()` is a *static method* returning `true` only on iOS 18.0+; `render()` returns `null` on non-iOS, so an unguarded usage silently renders nothing on iOS 17 and below.

```tsx
{ContactAccessButton.isAvailable() ? <ContactAccessButton query={q} caption="phone" /> : <MyFallback />}
```

Props (extends `ViewProps`): `query?: string`, `caption?: 'default' | 'email' | 'phone'`, `ignoredEmails?: string[]`, `ignoredPhoneNumbers?: string[]`, `tintColor?: ColorValue`, `backgroundColor?: ColorValue`, `textColor?: ColorValue`.

### Form options (iOS)

Two separate types, with different histories — do not over-pin:

- `CreateFormOptions` (for `Contact.presentCreateForm`) was introduced **whole** in **56.0.10** (#46960), with `cancelButtonTitle?: string` (default `"Cancel"`), `showsCancelButton?: boolean` (default `true`), and `preventAnimation?: boolean`. This type needs `>=56.0.10`.
- `FormOptions` (for `contact.editWithForm`) already had `cancelButtonTitle?` and `preventAnimation?` from the 56.0.6 root promotion — they work on any SDK 56 patch. 56.0.10 only **added `showsCancelButton?`** to it, deprecated `isNew?` as a no-op, and fixed `cancelButtonTitle` (it previously only applied when editing an existing contact).

`FormOptions` additionally has `allowsActions?`, `shouldShowLinkedContacts?`, `groupId?`, and the deprecated no-op `isNew?`.

### Data types / enums

`ContactField` — 30 members. The mapping rule is exact: **SCREAMING_SNAKE key → camelCase string value** (`GIVEN_NAME = 'givenName'`). The full set (`src/types/Contact.props.ts:7-38`):

`IS_FAVOURITE`, `FULL_NAME`, `GIVEN_NAME`, `MIDDLE_NAME`, `FAMILY_NAME`, `MAIDEN_NAME`, `NICKNAME`, `PREFIX`, `SUFFIX`, `PHONETIC_GIVEN_NAME`, `PHONETIC_MIDDLE_NAME`, `PHONETIC_FAMILY_NAME`, `COMPANY`, `PHONETIC_COMPANY_NAME`, `DEPARTMENT`, `JOB_TITLE`, `NOTE`, `IMAGE`, `THUMBNAIL`, `BIRTHDAY`, `NON_GREGORIAN_BIRTHDAY`, `EMAILS`, `PHONES`, `ADDRESSES`, `EXTRA_NAMES`, `DATES`, `RELATIONS`, `URL_ADDRESSES`, `SOCIAL_PROFILES`, `IM_ADDRESSES`.

> Gotcha: it is `IS_FAVOURITE = 'isFavourite'` — **British spelling**. (`expo-media-library`'s unrelated `AssetField.IS_FAVORITE` uses the American spelling. They do not match.) `BIRTHDAY` carries no platform restriction.

- `ContactsSortOrder` — `GivenName`, `FamilyName`, `UserDefault`, `None`.
- `ContactPatch` — partial update type (undefined fields ignored; lists entirely replaced).
- `ContactDetails` — complete contact representation with all fields.

### Module-level functions

- `Contacts.addContactsChangeListener(listener: () => void): EventSubscription` — observe contact modifications. Platform timing: Android ~5–7 second delay via `ContentObserver`; iOS immediate via `CNContactStoreDidChangeNotification`. (The JSDoc example inside the source says `addContactChangeListener` — that is a source typo; the real export is plural.)
- `Contacts.removeAllContactsChangeListeners(): void`
- `Contacts.getPermissionsAsync()` / `Contacts.requestPermissionsAsync()` → `ContactsPermissionResponse`

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
- `Asset.create(filePath: string, album?: Album)` → `Promise<Asset>` — create asset from a file path (replaces `saveToLibraryAsync`)
- `Asset.delete(assets: Asset[])` → `Promise<void>` — batch deletion

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
- `asset.getShape()` → `Promise<Shape | null>`

iOS-only getters — these throw `UnavailabilityError` on Android (the guard lives in the JS wrapper at `src/index.ts:19-51`, not native):
- `asset.getMediaSubtypes()` → `Promise<MediaSubtype[]>`
- `asset.getLivePhotoVideoUri()` → `Promise<string | null>`
- `asset.getIsInCloud()` → `Promise<boolean>`
- `asset.getOrientation()` → `Promise<number | null>`

> **iCloud-offloaded assets (56.0.8, #47790).** On iOS, `getUri()`, `getInfo()`, `getExif()` and `getOrientation()` now allow network access and will download the original from iCloud when "Optimize iPhone Storage" has offloaded it — matching the legacy API's `shouldDownloadFromNetwork` default. Expect real latency on those assets. Before 56.0.8 they simply failed to resolve.

> **Android saves >~2 GB — needs `>=56.0.10` (#47811).** Saving a file larger than about 2 GB (e.g. a long video via `Asset.create` / `createAssetAsync`) failed with "Unable to copy file into external storage" because the native copy called `FileChannel.transferTo` once instead of looping. Fixed on the SDK 56 line in **56.0.10** and on the 57 line in 57.0.3 — this is a patch bump, **not** a reason to upgrade to SDK 57.

### Class: `Album`

Static:
- `Album.create(name: string, assetsRefs: string[] | Asset[], moveAssets?: boolean)` → `Promise<Album>` — `assetsRefs` accepts file-path strings **or** `Asset` instances
- `Album.get(title: string)` → `Promise<Album | null>` — **can resolve `null`**; always null-check
- `Album.getAll()` → `Promise<Album[]>`
- `Album.delete(albums: Album[], deleteAssets?: boolean)` → `Promise<void>`

Instance:
- `album.getTitle()` → `Promise<string>`
- `album.getAssets()` → `Promise<Asset[]>`
- `album.add(assets: Asset | Asset[])` → `Promise<void>`
- `album.removeAssets(assets: Asset[])` — iOS only; removes without deleting
- `album.delete()` → `Promise<void>`

### Class: `Query` (Builder pattern)

Chainable filtered asset retrieval (`src/types/Query.ts:21-115`):
- `eq<T extends AssetField>(field: T, value: AssetFieldValueMap[T])` — equality, value type keyed off the field
- `within<T extends AssetField>(field: T, value: AssetFieldValueMap[T][])` — array membership
- `gt(field: AssetField, value: number)`, `gte(...)` — greater-than comparisons (**numbers only**)
- `lt(field: AssetField, value: number)`, `lte(...)` — less-than comparisons (**numbers only**)
- `album(album: Album)` — filter by album. Takes an **`Album` instance**, not an id or title string: `.album((await Album.get('Camera'))!)`
- `orderBy(sortDescriptors: SortDescriptor | AssetField)` — where `SortDescriptor = { key: AssetField; ascending?: boolean }`. Passing a bare `AssetField` sorts ascending.
- `limit(n)`, `offset(n)` — pagination
- `exe()` — executes, returns `Promise<Asset[]>`
- `exeForMetadata()` → `Promise<AssetMetadata[]>` — **added in 56.0.8** (#46485). Cheap bulk fetch straight from the media store: no file-path resolution, no file decoding. Use it for list/grid screens; use `Asset` getters only for heavy fields (URI, EXIF).

`AssetMetadata` (note the nullability, which differs from `AssetInfo`):
```ts
{
  id: string;
  filename: string | null;
  mediaType: MediaType;
  width: number | null;   // may be null on Android when the media store lacks it
  height: number | null;
  duration: number | null;
  creationTime: number | null;
  modificationTime: number | null;
  isFavorite: boolean;
}
```

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

- `AssetField` — **7 members**: `CREATION_TIME`, `MODIFICATION_TIME`, `MEDIA_TYPE`, `DURATION`, `WIDTH`, `HEIGHT`, `IS_FAVORITE` (American spelling; added in 56.0.8, #45769).
- `AssetFieldValueMap` — what makes `eq`/`within` type-safe: `CREATION_TIME`/`MODIFICATION_TIME`/`WIDTH`/`HEIGHT`/`DURATION` → `number`, `MEDIA_TYPE` → `MediaType`, `IS_FAVORITE` → `boolean`.
- `MediaType`: `IMAGE`, `VIDEO`, `AUDIO`, `UNKNOWN`.

### Hook: `usePermissions(options)`

```tsx
const [permissionResponse, requestPermission] = MediaLibrary.usePermissions({
  writeOnly: true,
  granularPermissions: ['photo']
});

// full read flow
useEffect(() => {
  (async () => {
    if (!permissionResponse?.granted) {
      const res = await requestPermission();
      if (!res.granted) return;
    }
    const recent = await new Query()
      .eq(AssetField.MEDIA_TYPE, MediaType.IMAGE)
      .orderBy({ key: AssetField.CREATION_TIME, ascending: false })
      .limit(50)
      .exeForMetadata();
    setItems(recent);
  })();
}, [permissionResponse?.granted]);
```

### Module-level functions

- `requestPermissionsAsync(writeOnly = false, granularPermissions?: GranularPermission[])` → `Promise<PermissionResponse>`
- `getPermissionsAsync(writeOnly = false, granularPermissions?: GranularPermission[])` → `Promise<PermissionResponse>`
- `presentPermissionsPicker(mediaTypes?: MediaTypeFilter[])` → `Promise<void>` — `@platform ios`, `@platform android 14+`. **No-op unless the user granted `limited` access.** It does not tell you what changed; subscribe via `addListener()` (iOS reports `hasIncrementalChanges: false` when permissions changed).
- `addListener(callback)` — subscribe to library changes
- `removeAllListeners()`

> Two different string unions, easily conflated:
> - `GranularPermission = 'audio' | 'photo' | 'video'` — for `requestPermissionsAsync`/`getPermissionsAsync`/`usePermissions`. **Android 13+ only**; silently dropped on iOS.
> - `MediaTypeFilter = 'photo' | 'video'` — for `presentPermissionsPicker`. No `'audio'`.

`PermissionResponse` extends the standard Expo one with `accessPrivileges?: 'all' | 'limited' | 'none'` (typed in 56.0.8, #47177). 56.0.8 also fixed the iOS permission guards for limited and write-only photo access (#47216).

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

Platforms: `['android', 'ios', 'web', 'tvos', 'expo-go']`. Playback/recording surface lives in `references/12-video-audio-playback.md`; this section owns only `useAudioStream`.

> **Android recording under Microsoft Intune — needs `>=56.0.13` (#47005).** Recording crashed in apps wrapped by the Intune app-protection SDK. Fixed on the SDK 56 line in **56.0.13** and on the 57 line in 57.0.3, so this is a patch bump, **not** a reason to upgrade to SDK 57.

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

> **The hook does not start capture.** You must call `stream.start()` yourself, and you must hold microphone permission first (`requestRecordingPermissionsAsync()`). A `useAudioStream` call with only `onBuffer` fires nothing. (`src/AudioStream.ts:13-17`.)

`AudioStream` (`declare class AudioStream extends SharedObject<AudioStreamEvents>`):
- `id: string`
- `readonly sampleRate: number` / `readonly channels: number` — the **actual** values the hardware delivers. Only meaningful after `start()`, and they may differ from what you requested.
- `readonly isStreaming: boolean`
- `start(): Promise<void>` — begins capture
- `stop(): void` — stops capture and releases native resources

`AudioStreamBuffer` (what `onBuffer` receives):
```ts
{
  data: ArrayBuffer;   // raw PCM. 'float32' = 4 bytes/sample in [-1.0, 1.0];
                       // 'int16' = 2 bytes/sample, little-endian signed.
                       // Multi-channel samples are INTERLEAVED: [L, R, L, R, ...]
  sampleRate: number;  // actual, may differ from requested
  channels: number;    // actual
  timestamp: number;   // seconds since start()
}
```
`AudioStreamEncoding = 'float32' | 'int16'`.

Example:
```tsx
import { useAudioStream, requestRecordingPermissionsAsync } from 'expo-audio';

function Recorder() {
  const { stream, isStreaming } = useAudioStream({
    channels: 1,
    encoding: 'float32',
    sampleRate: 48000,
    onBuffer: (buffer) => processPcm(buffer.data),
  });

  const begin = async () => {
    const { granted } = await requestRecordingPermissionsAsync();
    if (!granted) return;
    await stream.start();
  };

  return <Button title={isStreaming ? 'Stop' : 'Start'} onPress={() => (isStreaming ? stream.stop() : begin())} />;
}
```

Gotchas:
- The options `[sampleRate, channels, encoding]` are the `useReleasingSharedObject` dependency list, so the **native stream is torn down and recreated** whenever any of them changes. Keep them stable (module constants, not inline-computed values).
- **Web is a stub.** `AudioStream.web.ts` returns `{ stream: null as any, isStreaming: false }` — nothing is captured and `stream.start()` throws a null-reference error. Guard with `Platform.OS !== 'web'`.

### Live-stream additions (changelog + AudioStatus)

Summarised here for completeness; the playback-side detail lives in `references/12-video-audio-playback.md`.

New `AudioStatus` fields:
- `isLive` — whether the current audio source is a live stream with indefinite duration
- `currentOffsetFromLive` — seconds behind the live edge, or `null` if not a live stream
- `error` — playback error message, or `null` if no error

New mode/options:
- iOS: `isLiveStream` lock-screen option
- `AudioMode.playsInSilentMode: boolean` (default `true`) — **cross-platform, not Android-only**. It has always existed on iOS; SDK 56 added the Android implementation (#43117), where `false` suppresses playback when the ringer mode is silent or vibrate. (`src/Audio.types.ts:553-560`.)
- `AudioMode.shouldPlayInBackground` — whether the audio session stays active in background (default `false`)

---

## expo-haptics (SDK 56)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/haptics/

### Web haptics on Safari

Web haptics work on **Safari**. Attribution correction: this shipped in **expo-haptics 55.0.12** (2026-04-06, #44261) and was merely re-advertised in the SDK 56 changelog — it is not an SDK 56 feature, and SDK 55 apps already have it.

Implementation matters here: `navigator.vibrate` does not exist on iOS Safari, so the module falls back to creating a hidden `<input type="checkbox" switch>` and programmatically clicking it, borrowing the OS haptic that iOS fires for switch toggles (`src/ExpoHaptics.web.ts`). Everywhere else it uses the **Web Vibration API** with per-style patterns (e.g. `Light` → `[40]`, `Error` → `[60, 100, 60, 100, 60]`). Browser support varies and the device must have vibration hardware. Order matters: if `navigator.vibrate` exists the module calls it and returns immediately; the `(pointer: coarse)` check gates **only** the iOS-Safari switch fallback, so the hidden-checkbox trick never fires on desktop browsers that lack `navigator.vibrate`.

Supported platforms: `['android', 'ios', 'web']`.

### Methods

- `Haptics.impactAsync(style)` → `Promise<void>` — `style` is optional `ImpactFeedbackStyle` (default Medium). Styles: `Light`, `Medium`, `Heavy`, `Rigid`, `Soft`.
- `Haptics.notificationAsync(type)` → `Promise<void>` — `type` is optional `NotificationFeedbackType` (default Success). Types: `Success`, `Warning`, `Error`.
- `Haptics.selectionAsync()` → `Promise<void>` — signals a selection change was registered.
- `Haptics.performAndroidHapticsAsync(type: AndroidHaptics)` — Android only.

### `AndroidHaptics` enum

19 members. **The casing is unusual and gets hallucinated**: it is `Gesture_Start` — capitalised words joined by underscores — not `GESTURE_START` and not `gestureStart`. Values are kebab-case (`'gesture-start'`). Full list (`src/Haptics.types.ts:52-131`):

`Confirm`, `Reject`, `Gesture_Start`, `Gesture_End`, `Toggle_On`, `Toggle_Off`, `Clock_Tick`, `Context_Click`, `Drag_Start`, `Keyboard_Tap`, `Keyboard_Press`, `Keyboard_Release`, `Long_Press`, `Virtual_Key`, `Virtual_Key_Release`, `No_Haptics`, `Segment_Tick`, `Segment_Frequent_Tick`, `Text_Handle_Move`.

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

`expo-asset` now supports 3D models in **GLB** format (56.0.0, #42495). `.glb` is recognised because the plugin's `MODEL_TYPES = ['.glb']` feeds `ACCEPTED_TYPES` (`plugin/src/utils.ts:8`).

The config plugin takes a **single prop**, `assets?: string[]` — files or directories relative to the project root — and returns the config untouched if it is absent or empty (`plugin/src/withAssets.ts:8-27`):

```json
{ "plugins": [["expo-asset", { "assets": ["./assets/models", "./assets/models/robot.glb"] }]] }
```

Accepted extensions: `.json`, `.db`, images `.png/.jpg/.gif`, media `.mp4/.mp3/.lottie/.riv`, models `.glb`. Font extensions (`.otf`/`.ttf`) pass the extension check but are then dropped with a warning — use expo-font instead (`plugin/src/utils.ts:57-64`).

Supported platforms: `['android', 'ios', 'tvos', 'web', 'expo-go']`.

### Class: `Asset`

Properties:
- `localUri` — `file://` URI pointing to the local file on the device (after download)
- `uri` — remote asset location (Expo servers or dev CLI)
- `name`, `type` — asset filename components (`name` excludes the extension and anything from `@` onward)
- `width`, `height` — image dimensions (nullable for non-image assets)
- `downloaded` — boolean, download completion flag
- `hash` — `string | null`, the MD5 hash of the asset's data

Methods:
- `downloadAsync()` — downloads asset data to device cache; resolves to the `Asset` instance
- `static loadAsync(moduleId: number | number[] | string | string[]): Promise<Asset[]>` — `fromModule()` + `downloadAsync()`. **It always resolves to an array, even for a single module.** This is the #1 gotcha here; the source's own example destructures:
  ```ts
  const [{ localUri }] = await Asset.loadAsync(require('./assets/snack-icon.png'));
  ```
- `static fromModule(virtualAssetModule)` — creates an `Asset` from a `require()` statement or external URL
- `static fromURI(uri: string): Asset` — creates an `Asset` from a URL (including base64 data URIs)
- `static fromMetadata(meta: AssetMetadata): Asset`

### Hook: `useAssets`

```ts
const [assets, error] = useAssets([require('path/to/asset.jpg'), require('path/to/other.png')]);
```
Signature: `useAssets(moduleIds: number | number[]): [Asset[] | undefined, Error | undefined]`.

Two things narrower than `loadAsync`:
- It accepts **only `require()` ids**, not URL strings. `Asset.loadAsync` accepts URLs; `useAssets` does not.
- Assets are not reloaded when the list changes — the internal `useEffect` has an **empty dependency array** (`src/AssetHooks.ts:32`), so it runs exactly once.

---

## SDK 57 delta

**The JS/TS API surface of this domain is byte-identical between SDK 56 and SDK 57.** Every statement above is equally true on SDK 57 — do not go hunting for a migration. The only 57-side movement is two native iOS `expo-audio` fixes, listed below.

Evidence: `git diff origin/sdk-56 origin/sdk-57 -- packages/<pkg>/src packages/<pkg>/plugin/src` returns **empty output** for all six packages this file owns — `expo-calendar`, `expo-contacts`, `expo-media-library`, `expo-audio`, `expo-haptics`, `expo-asset`. Not one line of the TS/JS API surface differs between the two release branches. The whole-package diff (excluding `build/` and `CHANGELOG.md`) is the version string in `package.json` and `android/build.gradle`, plus a whitespace fix in `expo-media-library`'s `MediaStoreAudio.kt` and iOS internals in `expo-audio`'s `AudioModule.swift` / `AudioPlaylist.swift`. The versioned docs are identical too, apart from `sourceCodeUrl` and two calendar prose edits — `calendar.mdx:172` (an anchor link dropped from "system-provided calendar UI") and `calendar.mdx:180` (the write-only-access rewrite, already folded into the [expo-calendar config plugin](#config-plugin) section above, since it is equally true for SDK 56).

The reason 57 looks empty is that SDK 56 is still actively patched and almost everything landed on both branches at once — see [Not 57 deltas](#not-57-deltas--backported-to-the-sdk-56-line) below.

> **Trap for future maintainers:** do **not** diff `docs/public/static/data/v56.0.0/*.json` against `v57.0.0/*.json`. The frozen v56 data on `main` was generated at an older commit than the `sdk-56` branch tip, so it reports false 57-only additions (e.g. `RecordingDirectory` / `RecordingOptions.directory`, which actually shipped in expo-audio **56.0.12**, #46189). Diff `packages/<pkg>/src` between the branches instead.

### Genuinely 57-only

Two bug fixes, both native iOS, no API change. Verified present in `packages/expo-audio/CHANGELOG.md` on `origin/sdk-57` and **absent** from the same file on `origin/sdk-56`:

- `expo-audio` 57.0.0 — [iOS] the audio session is deactivated off the main thread, avoiding app hangs (#47066).
- `expo-audio` 57.0.1 — [iOS] fixes playlist `currentIndex` freezing after the first auto-advance (#47257).

These are the **only** functional reason in this domain to prefer 57 over a fully-patched 56.

### Not 57 deltas — backported to the SDK 56 line

Both of these shipped on both branches. Upgrading to 57 to get them is wasted work; bump the patch instead. Details are in the main body:

- [Android] recording crash under Microsoft Intune (#47005) — `expo-audio` **56.0.13** and 57.0.3. See [expo-audio](#expo-audio-sdk-56).
- [Android] saving files larger than ~2 GB (#47811) — `expo-media-library` **56.0.10** and 57.0.3. See [Class: `Asset`](#class-asset).

Likewise, everything the redesign added late is on both lines: `Query.exeForMetadata()` / `AssetField.IS_FAVORITE` / `accessPrivileges` are `57.0.0` **and** `56.0.8`; the iOS contact-form options are `57.0.0` **and** `56.0.10`; the calendar `UnavailabilityError` guards are `57.0.0` **and** `56.0.9`. Staying on SDK 56 costs you nothing here — just keep patches current.

`expo-haptics` and `expo-asset` 57.x releases are all literally "_This version does not introduce any user-facing changes._". `expo-calendar` 57.0.0 (the #46416 reminder guards / `@platform ios` annotations) and `expo-contacts` 57.0.0 (the #46960 form options) do carry entries, but both are byte-identical to what shipped in 56.0.9 and 56.0.10 respectively; their 57.0.1+ patches are no-ops.

### Version pins (56 → 57)

From `packages/expo/bundledNativeModules.json` on `origin/sdk-56` and `origin/sdk-57` — the pins `expo install` resolves against. Each value is also the highest published patch on npm for that line.

| Package | SDK 56 | SDK 57 |
| --- | --- | --- |
| `expo-calendar` | `~56.0.9` | `~57.0.1` |
| `expo-contacts` | `~56.0.11` | `~57.0.2` |
| `expo-media-library` | `~56.0.10` | `~57.0.3` |
| `expo-audio` | `~56.0.13` | `~57.0.3` |
| `expo-haptics` | `~56.0.3` | `~57.0.1` |
| `expo-asset` | `~56.0.21` | `~57.0.7` |

> Do **not** read these pins out of `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json`. Those files are frozen at an older commit: v56 reports `expo-asset ~56.0.16` / `expo-media-library ~56.0.6` / `expo-calendar ~56.0.8`, and v57 reports a flat `~57.0.0` for all six. Both are wrong.

Every feature this file gates on a patch version (`addEventWithForm` on Android 56.0.7, `exeForMetadata` 56.0.8, `CreateFormOptions` 56.0.10, the reminders `UnavailabilityError` guards 56.0.9) is already satisfied by the SDK 56 pins in the table above.
