# Expo SDK 56 — expo-sqlite, expo-updates / EAS Update, bundle diffing, Convex

Official documentation reference for the Expo SDK 56 knowledge base. Captures APIs, config
keys, and code examples for `expo-sqlite`, `expo-updates` / EAS Update (including **bundle
diffing** — Expo's term for the bsdiff/Hermes-bytecode patching feature), and the Convex
integration. Grouping rationale: these are the SDK 56 changelog areas not covered by the
other reference files.

> `expo-sqlite` and `expo-updates` have **identical public JS APIs — and identical `src/` —
> in SDK 56 and SDK 57** (zero-line diff between the `sdk-56` and `sdk-57` release branches);
> only version pins and an iOS updates-logging refactor differ.
> See [SDK 57 delta](#sdk-57-delta).

### Get this right (the four things models most often get wrong)

1. `enableFTS` / `useSQLCipher` / `useLibSQL` are **config-plugin props**, not
   `SQLiteOpenOptions`. Passing them to `openDatabaseAsync()` does nothing.
2. `sql` is an instance property on `SQLiteDatabase` — use ``db.sql`...` `` or
   `const { sql } = db;`. There is no `import { sql } from 'expo-sqlite'`.
3. There are **four** runtime version policies: `nativeVersion | sdkVersion | appVersion |
   fingerprint`. `ios.runtimeVersion` / `android.runtimeVersion` override the top-level value.
4. `updates.enableBsdiffPatchSupport` only became default-`true` in **expo-updates 56.0.13**
   — not at 56.0.0.

---

## 1. SDK 56 Changelog Highlights

Source: https://expo.dev/changelog/sdk-56

### expo-sqlite

All three SDK 56 items are **native representation changes**, not new features. Verbatim from
`packages/expo-sqlite/CHANGELOG.md` under `## 56.0.0`:

- **[Android] / [iOS] Returned blob columns now use native `ArrayBuffer`s** (PR #42640 /
  PR #42642) — replaces the legacy `JavaScriptArrayBuffer`.
- **Statement bind params now use native `ArrayBuffer`s for blob columns** (PR #42639).
- **Session changesets now use native `ArrayBuffer`s** (PR #42638).
- **[iOS] Updated sync function signatures** (`runSync`, `applyChangesetSync`,
  `invertChangesetSync`) to accept `any AnyArrayBuffer` in place of the removed
  `JavaScriptArrayBuffer` (PR #44337).
- New feature in 56.0.0: **a typed config plugin function** is now exported (PR #44098).

Do not attribute the underlying features to SDK 56: the **SQLite Session Extension** and
`backupDatabaseAsync`/`Sync` shipped in `expo-sqlite` **15.2.0 (2025-04-04, SDK 53)**, and
**tagged template literals** (`db.sql`) shipped in **55.0.0 (2026-01-21, PR #40972)**.

### Bundle diffing (expo-updates / EAS Update)

Expo's own term is **bundle diffing** / **bundle patch** — the docs page never says "Hermes
bytecode diffing". Search the docs for "bundle diffing".

- Requires **SDK 55 or later**. Uses the [bsdiff](https://en.wikipedia.org/wiki/Bsdiff)
  algorithm.
- Status: **beta** (`docs/pages/eas-update/bundle-diffing.mdx:11`).
- Default: the `updates.enableBsdiffPatchSupport` default flipped `false → true` in
  **expo-updates 56.0.13 (2026-05-19, PR #45928)**. On SDK 55 and on expo-updates
  56.0.0–56.0.12 you must set it to `true` explicitly. Verified in
  `packages/@expo/config-plugins/src/utils/Updates.ts` →
  `getUpdatesBsdiffPatchSupportEnabled()` returns `true` when unset.
- Opt out: set `"enableBsdiffPatchSupport": false` in the `updates` block of **app.json**.
- **Patching against the embedded bundle shipped** (experimental opt-in) — see
  [section 4, Bundle diffing](#bundle-diffing). It is no longer "planned".
- The "**on average 58% smaller**" figure comes from the SDK 56 changelog blog post only; it
  is not corroborated anywhere in `expo/expo`.

### Convex Integration

EAS automates Convex backend provisioning and linking via the `eas integrations:convex:*`
command family. See [section 5](#5-convex-integration) — the integration is an EAS-CLI/server
feature with **no SDK version coupling** (it is not SDK-56-specific).

---

## 2. expo-sqlite API Reference

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sqlite/

### Opening a database

```ts
SQLite.openDatabaseAsync(databaseName, options?, directory?)  // Promise<SQLiteDatabase>
SQLite.openDatabaseSync(databaseName, options?, directory?)   // SQLiteDatabase (blocks thread)
```

**Runtime open options — `SQLiteOpenOptions` has exactly these members**
(`packages/expo-sqlite/src/NativeDatabase.ts`):

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `enableChangeListener` | `boolean` | `false` | Calls `sqlite3_update_hook()` so `addDatabaseChangeListener()` receives `onDatabaseChange` events |
| `useNewConnection` | `boolean` | `false` | Create a new connection even if one for the same database name is cached |
| `libSQLOptions` | `{ url: string; authToken: string; remoteOnly?: boolean }` | — | libSQL server integration (`remoteOnly` defaults `false`) |
| `finalizeUnusedStatementsBeforeClosing` | `boolean` | `true` | `@hidden` — finalize unclosed statements on close |

> **`enableFTS`, `useSQLCipher`, `useLibSQL` are NOT open options.** They are **config-plugin
> props** and require a native rebuild. Passing them to `openDatabaseAsync()` silently does
> nothing.

**Config-plugin props** (`app.json` → `plugins: [["expo-sqlite", { ... }]]`;
`packages/expo-sqlite/plugin/src/withSQLite.ts`):

| Prop | Default | Purpose |
|------|---------|---------|
| `customBuildFlags` | — | **String** of build flags, for example `"-DSQLITE_ENABLE_DBSTAT_VTAB=1 -DSQLITE_ENABLE_SNAPSHOT=1"` |
| `enableFTS` | `true` | Enable the FTS3/FTS4/FTS5 extensions |
| `useSQLCipher` | `false` | Use SQLCipher instead of stock SQLite |
| `useLibSQL` | `false` | Use libSQL instead of stock SQLite |
| `withSQLiteVecExtension` | `false` | Add the `sqlite-vec` extension to `bundledExtensions` |
| `android` / `ios` | — | Per-platform override blocks accepting the same keys (Android also accepts `useSQLiteVec`) |

```json app.json
{
  "expo": {
    "plugins": [
      ["expo-sqlite", {
        "enableFTS": true,
        "useSQLCipher": true,
        "android": { "enableFTS": false, "useSQLCipher": false },
        "ios": { "customBuildFlags": "-DSQLITE_ENABLE_DBSTAT_VTAB=1" }
      }]
    ]
  }
}
```

> **Docs-snapshot warning.** The `v57.0.0/sdk/sqlite` page shows `customBuildFlags` as an
> **array** — that page is stale; the plugin type is `customBuildFlags?: string`. Use the
> string form on both SDKs.

### Query execution methods

| Method | Use case |
|--------|----------|
| `runAsync(source, params)` | Write operations (INSERT, UPDATE, DELETE) → returns `SQLiteRunResult` |
| `getFirstAsync(source, params)` | Fetch a single row |
| `getAllAsync(source, params)` | Fetch all results at once |
| `getEachAsync(source, params)` | Async iterator for large result sets |
| `execAsync(source)` | Bulk queries (unescaped — use cautiously) |

`SQLiteRunResult` exposes `lastInsertRowId` and `changes`.

**Parameter binding** supports arrays, variadic arguments, or named objects with `$key` syntax.

> `getFirstAsync()` returns the row object directly — `result['COUNT(*)']`, **not**
> `result.rows[0][...]`. (The `v57.0.0/sdk/sqlite` docs snapshot regressed this; fixed in the
> unversioned and v56 pages by PR #46439.)

### Prepared statements (statement bind params)

```ts
const stmt = await db.prepareAsync('SELECT * FROM users WHERE id = ?');
try {
  const result = await stmt.executeAsync(userId);
  const row = await result.getFirstAsync();
} finally {
  await stmt.finalizeAsync();
}
```

Prepared statements separate query logic from input parameters; SQLite automatically escapes
inputs, making them the recommended defense against SQL injection for user input.

### Tagged template literals (shipped in SDK 55, PR #40972)

`sql` is an **instance property** of `SQLiteDatabase`, declared as an arrow-function class
property, so destructuring is safe. There is no module-level `sql` export.

```ts
const db = await SQLite.openDatabaseAsync('test.db');
const { sql } = db;                    // or call db.sql`...` directly

const users = await sql<User>`SELECT * FROM users WHERE age > ${21}`;  // T[]
const user  = await sql<User>`SELECT * FROM users WHERE id = ${id}`.first();
```

The query object is **thenable**: plain `await` runs `getAllAsync` for reads and `runAsync`
for INSERT/UPDATE/DELETE (resolving to `SQLiteRunResult`). Full `SQLiteTaggedQuery` surface:

| Member | Returns |
|--------|---------|
| `then()` (implicit `await`) | `T[]` for reads, `SQLiteRunResult` for writes |
| `first()` | `Promise<T \| null>` |
| `values()` | `Promise<any[][]>` — rows as arrays, for example `[["Alice", 30]]` |
| `each()` | `AsyncIterableIterator<T>` |
| `allSync()` | `T[]` (blocks the thread) |
| `firstSync()` | `T \| null` |
| `valuesSync()` | `any[][]` |
| `eachSync()` | `IterableIterator<T>` |

### React integration

```tsx
<SQLiteProvider databaseName="test.db" onInit={migrateDbIfNeeded}>
  <App />
</SQLiteProvider>
```

```tsx
// Inside a descendant component:
const db = useSQLiteContext();
```

`SQLiteProviderProps` (`packages/expo-sqlite/src/hooks.tsx`):

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `databaseName` | `string` | — | Required |
| `directory` | `string` | `defaultDatabaseDirectory` | |
| `options` | `SQLiteOpenOptions` | — | Runtime open options (see above) |
| `assetSource` | `{ assetId: number; forceOverwrite?: boolean }` | `forceOverwrite: false` | Import a bundled **.db** from `require()` |
| `children` | `React.ReactNode` | — | Required |
| `onInit` | `(db: SQLiteDatabase) => Promise<void>` | — | Migrations/setup before children render |
| `onError` | `(error: Error) => void` | rethrow | |
| `useSuspense` | `boolean` | `false` | `React.Suspense` integration |

> Combining `onError` with `useSuspense` **throws** at render:
> `'Cannot use `onError` with `useSuspense`, use error boundaries instead.'` Use an error
> boundary in the Suspense case.

### Transactions

```ts
db.withTransactionAsync(async () => { /* ... */ });
db.withExclusiveTransactionAsync(async (txn) => { /* ... */ });
```

### Native ArrayBuffer / blob handling (SDK 56)

Use `Uint8Array` for passing and receiving binary data; the database automatically handles
serialization/deserialization.

```ts
// Store binary data
await db.execAsync(`
DROP TABLE IF EXISTS blobs;
CREATE TABLE IF NOT EXISTS blobs (id INTEGER PRIMARY KEY NOT NULL, data BLOB);
`);

const blob = new Uint8Array([0x00, 0x01, 0x02, 0x03, 0x04, 0x05]);
await db.runAsync('INSERT INTO blobs (data) VALUES (?)', blob);
```

```ts
// Retrieve binary data
const row = await db.getFirstAsync<{ data: Uint8Array }>('SELECT * FROM blobs');
expect(row.data).toEqual(blob);
```

Relevant types (`packages/expo-sqlite/src/NativeStatement.ts`, `NativeSession.ts`):
- `SQLiteBindBlobValue = Uint8Array | ArrayBuffer` — both are accepted for blob binding.
- `SQLiteBindValue = string | number | null | boolean | SQLiteBindBlobValue`.
- `Changeset` — type alias for `Uint8Array`.

### Session extension / changesets (shipped SDK 53; native repr. changed in SDK 56)

Create a session from the database:

```ts
db.createSessionAsync(dbName: string = 'main')  // → Promise<SQLiteSession>
db.createSessionSync(dbName: string = 'main')   // → SQLiteSession (blocks thread)
```

`SQLiteSession` methods (each has an `...Async` and a synchronous `...Sync` variant):

| Method | Params | Returns | Description |
|--------|--------|---------|-------------|
| `attachAsync(table)` | `table: string \| null` (null = all tables) | `Promise<void>` | Attach table(s) for tracking |
| `enableAsync(enabled)` | `enabled: boolean` | `Promise<void>` | Enable/disable session tracking |
| `closeAsync()` | — | `Promise<void>` | Close session (`sqlite3_session_delete()`) |
| `createChangesetAsync()` | — | `Promise<Changeset>` | Generate changeset from tracked changes |
| `createInvertedChangesetAsync()` | — | `Promise<Changeset>` | Create inverted (undo) changeset |
| `applyChangesetAsync(changeset)` | `changeset: Changeset` | `Promise<void>` | Apply a changeset |
| `invertChangesetAsync(changeset)` | `changeset: Changeset` | `Promise<Changeset>` | Invert a changeset (undo) |

### Additional storage / advanced APIs

- `expo-sqlite/kv-store` — drop-in AsyncStorage replacement with synchronous methods
  (`import Storage from 'expo-sqlite/kv-store'`).
- `expo-sqlite/localStorage/install` — web-compatible `globalThis.localStorage`
  (`import 'expo-sqlite/localStorage/install'`; a no-op on web, stripped from the production
  bundle there).
- `loadExtensionAsync()` / `loadExtensionSync()` — load extensions (for example sqlite-vec).
- `db.serializeAsync()` / `serializeSync()` plus module-level `deserializeDatabaseAsync()` /
  `deserializeDatabaseSync()` — serialize/restore a database.
- `deleteDatabaseAsync(databaseName, directory?)` / `deleteDatabaseSync(...)`.
- `addDatabaseChangeListener(listener)` — requires `enableChangeListener: true` at open time.
- **Backup** (module-level, shipped 15.2.0 / SDK 53; both database names default to `'main'`):

  ```ts
  import { backupDatabaseAsync, backupDatabaseSync } from 'expo-sqlite';

  await backupDatabaseAsync({
    sourceDatabase,            // SQLiteDatabase
    sourceDatabaseName,        // string?, default 'main'
    destDatabase,              // SQLiteDatabase
    destDatabaseName,          // string?, default 'main'
  }); // Promise<void>
  ```

---

## 3. expo-updates API Reference

Source: https://docs.expo.dev/versions/v56.0.0/sdk/updates/

### `useUpdates()` hook

Returns `UseUpdatesReturnType` — all **14** members
(`packages/expo-updates/src/UseUpdates.types.ts`):

- `currentlyRunning: CurrentlyRunningInfo`
- `isStartupProcedureRunning: boolean` — startup check still running (happens when
  `fallbackToCacheTimeout` is shorter than the launch-time fetch)
- `isUpdateAvailable: boolean`
- `isUpdatePending: boolean`
- `isChecking: boolean`
- `isDownloading: boolean`
- `isRestarting: boolean`
- `restartCount: number` — JS restarts since cold start
- `downloadProgress?: number` — 0…1; smoother when the server sends `Content-Length`
- `availableUpdate?: UpdateInfo`
- `downloadedUpdate?: UpdateInfo`
- `checkError?: Error`
- `downloadError?: Error`
- `lastCheckForUpdateTimeSinceRestart?: Date` — does not persist across reloads/restarts

```tsx
const { currentlyRunning, isUpdateAvailable } = Updates.useUpdates();
if (isUpdateAvailable) {
  await Updates.fetchUpdateAsync();
}
```

### Exported functions (all nine)

| Method | Signature | Notes |
|--------|-----------|-------|
| `checkForUpdateAsync()` | `Promise<UpdateCheckResult>` | Queries server availability without downloading. Rejects in dev mode, Expo Go, or on network errors. |
| `fetchUpdateAsync()` | `Promise<UpdateFetchResult>` | Downloads latest update to device storage. Call `reloadAsync()` to apply immediately. Rejects in dev mode/Expo Go/network errors. |
| `reloadAsync(options?)` | `options?: { reloadScreenOptions?: ReloadScreenOptions }` → `Promise<void>` | Reloads app using newest downloaded update. Rejects in dev mode, Expo Go, or when updates disabled. |
| `readLogEntriesAsync(maxAge?)` | `maxAge: number = 3600000` → `Promise<UpdatesLogEntry[]>` | Native update logs. **This is how you verify a bundle patch was applied** — look for a "patch successfully applied" entry. See the Android purge caveat below. |
| `clearLogEntriesAsync()` | `Promise<void>` | |
| `getExtraParamsAsync()` | `Promise<Record<string, string>>` | Extra params sent with the manifest request |
| `setExtraParamAsync(key, value)` | `key: string, value: string \| null \| undefined` → `Promise<void>` | `null`/`undefined` removes the param |
| `setUpdateURLAndRequestHeadersOverride(configOverride)` | `void` | Requires `updates.disableAntiBrickingMeasures: true` |
| `setUpdateRequestHeadersOverride(requestHeaders)` | `void` | Requires `updates.disableAntiBrickingMeasures: true` |

**Module constants:** `Updates.channel`, `.checkAutomatically`, `.createdAt`,
`.emergencyLaunchReason`, `.isEmbeddedLaunch`, `.isEmergencyLaunch`, `.isEnabled`,
`.latestContext`, `.launchDuration`, `.manifest`, `.runtimeVersion`, `.updateId`.

**Rollback result variants:** `checkForUpdateAsync()` can resolve to
`UpdateCheckResultRollBack` and `fetchUpdateAsync()` to `UpdateFetchResultRollBackToEmbedded`
— these pair with `eas update:roll-back-to-embedded`. Handle them if you branch on the result.

> **Android log-purge caveat (SDK 56 *and* 57).**
> `UpdatesLogReader.ONE_DAY_MILLISECONDS` is `86400` — seconds, not milliseconds — on both
> `origin/sdk-56` and `origin/sdk-57`
> (`android/src/main/java/expo/modules/updates/logging/UpdatesLogReader.kt:57`), so the
> "older than one day" purge actually discards entries after **~86 seconds**. Call
> `readLogEntriesAsync()` promptly after a reload when verifying a bundle patch. Fixed to
> `86_400_000L` on `main` (#46182) — expect it in SDK 58, not 57.

`ReloadScreenOptions` (`packages/expo-updates/src/ReloadScreen.types.ts`):

| Key | Type | Default |
|-----|------|---------|
| `backgroundColor` | `string` | `'#ffffff'` |
| `image` | `string \| number \| ReloadScreenImageSource` | — |
| `imageResizeMode` | `'contain' \| 'cover' \| 'center' \| 'stretch'` | `'contain'` |
| `imageFullScreen` | `boolean` | `false` |
| `fade` | `boolean` | `false` |
| `spinner` | `{ enabled?: boolean; color?: string; size?: 'small' \| 'medium' \| 'large' }` | `size: 'medium'`; `enabled` defaults `true` with no image, `false` with an image |

`ReloadScreenImageSource = { url?, width?, height?, scale? }`.

### App config keys (`expo.updates` in app.json)

**Required:**
- `updates.url` (string) — remote update service endpoint.
- `runtimeVersion` (string | object) — build compatibility identifier.

**Optional — exactly 12 `updates.*` keys**
(`docs/public/static/schemas/v56.0.0/app-config-schema.json`; corroborated by
`packages/@expo/config-types/src/ExpoConfig.ts`, which is byte-identical on `origin/sdk-56`
and `origin/sdk-57`):
- `updates.enabled` (boolean, default `true`)
- `updates.checkAutomatically` (`"ON_LOAD" | "ON_ERROR_RECOVERY" | "NEVER" | "WIFI_ONLY"`, default `"ON_LOAD"`)
- `updates.fallbackToCacheTimeout` (number, default `0`) — ms before using cached update
- `updates.requestHeaders` (object) — custom HTTP headers
- `updates.useEmbeddedUpdate` (boolean, default `true`)
- `updates.codeSigningCertificate` (string)
- `updates.codeSigningMetadata` (object)
- `updates.disableAntiBrickingMeasures` (boolean, default `false`)
- `updates.assetPatternsToBeBundled` (string[], no default) — globs relative to project root
  selecting which assets ship in updates; when omitted, all asset files are included
- `updates.useNativeDebug` (boolean, default `false`) — sets `EX_UPDATES_NATIVE_DEBUG` in
  **Podfile.properties.json** / **gradle.properties** so Xcode/Android Studio debug builds run
  with expo-updates enabled and JS debugging disabled. Not for production.
- **`updates.enableBsdiffPatchSupport` (boolean, default `true` from expo-updates 56.0.13)** —
  controls bundle-diff / bsdiff patching; set to `false` to opt out.

**Runtime version policies — four values**, `'nativeVersion' | 'sdkVersion' | 'appVersion' |
'fingerprint'`:
- `"appVersion"` — uses the app `version` field.
- `"nativeVersion"` — combines `version` with `buildNumber`/`versionCode`.
- `"sdkVersion"` — derived from `sdkVersion`; throws if `sdkVersion` is unset.
- `"fingerprint"` — auto-calculated from project state at build time (not resolvable in a
  config plugin).

`runtimeVersion` also accepts two plain-string forms: an arbitrary version string, or the
literal `"exposdk:<x.y.z>"`.

**Per-platform overrides:** `ios.runtimeVersion` / `android.runtimeVersion` take precedence
over the top-level value — the resolver is
`config[platform]?.runtimeVersion ?? config.runtimeVersion`
(`packages/@expo/config-plugins/src/utils/Updates.ts`).

> **`appVersion` honours `ios.version` / `android.version` — in both SDKs.** PR #47416 makes
> `Updates.getAppVersion`, `Updates.getNativeVersion` and the `appVersion` policy prefer the
> platform-specific override; previously they were ignored and the app-config `version` field
> (`config.version ?? '1.0.0'`) was used. `getAppVersion` gained an optional `platform`
> argument — calling it without one preserves the old behaviour. Android `getNativeVersion`
> previously used the **iOS** version for the `${version}` component; that is fixed.
> **This is not a 56 → 57 delta:** it shipped in `@expo/config-plugins` **56.0.12** and
> **57.0.3**, both released 2026-07-07. If you pin an older `@expo/config-plugins`,
> `ios.version` / `android.version` are ignored by the `appVersion` policy. Because it can
> silently change the runtime version string a build computes, re-check your runtime version
> and republish after upgrading `@expo/config-plugins` across that boundary.

### App config → native key mapping (bare / non-CNG projects)

From `docs/pages/versions/v56.0.0/sdk/updates.mdx`. Android meta-data names are prefixed
`expo.modules.updates.`; the Map key column is for programmatic `UpdatesConfiguration`.

| App config property | Default | Req? | iOS plist key | Android meta-data name | Android Map key |
|---|---|---|---|---|---|
| `updates.enabled` | `true` | no | `EXUpdatesEnabled` | `ENABLED` | `enabled` |
| `updates.url` | (none) | **yes** | `EXUpdatesURL` | `EXPO_UPDATE_URL` | `updateUrl` |
| `updates.requestHeaders` | (none) | no | `EXUpdatesRequestHeaders` | `UPDATES_CONFIGURATION_REQUEST_HEADERS_KEY` | `requestHeaders` |
| `runtimeVersion` | (none) | **yes** | `EXUpdatesRuntimeVersion` | `EXPO_RUNTIME_VERSION` | `runtimeVersion` |
| `updates.checkAutomatically` | `ON_LOAD` | no | `EXUpdatesCheckOnLaunch` | `EXPO_UPDATES_CHECK_ON_LAUNCH` | `checkOnLaunch` |
| `updates.fallbackToCacheTimeout` | `0` | no | `EXUpdatesLaunchWaitMs` | `EXPO_UPDATES_LAUNCH_WAIT_MS` | `launchWaitMs` |
| `updates.useEmbeddedUpdate` | `true` | no | `EXUpdatesHasEmbeddedUpdate` | `HAS_EMBEDDED_UPDATE` | `hasEmbeddedUpdate` |
| `updates.codeSigningCertificate` | (none) | no | `EXUpdatesCodeSigningCertificate` | `CODE_SIGNING_CERTIFICATE` | `codeSigningCertificate` |
| `updates.codeSigningMetadata` | (none) | no | `EXUpdatesCodeSigningMetadata` | `CODE_SIGNING_METADATA` | `codeSigningMetadata` |
| `updates.assetPatternsToBeBundled` | (none) | no | N/A | N/A | N/A |
| `updates.disableAntiBrickingMeasures` | `false` | no | `EXUpdatesDisableAntiBrickingMeasures` | `DISABLE_ANTI_BRICKING_MEASURES` | `disableAntiBrickingMeasures` |
| `updates.enableBsdiffPatchSupport` | `true` | no | `EXUpdatesEnableBsdiffPatchSupport` | `ENABLE_BSDIFF_PATCH_SUPPORT` | `enableBsdiffPatchSupport` |

---

## 4. EAS Update

Sources:
- https://docs.expo.dev/eas-update/introduction/
- https://docs.expo.dev/eas-update/getting-started/

### What it is

EAS Update is a cloud service for over-the-air (OTA) updates for apps using the
`expo-updates` library. It pushes JavaScript, styling, and image changes without app store
resubmission. Users receive updates on their next app launch. Native code changes still
require a new build.

### Key concepts

- **Channels** — deployment targets that organize updates (e.g. `production`, `preview`).
- **Branches** — series of updates; channels point at branches.
- **Runtime Versions** — policy system ensuring updates are sent only to builds with
  compatible native code.
- **Deployments Dashboard** — shows which updates reach which builds; integrates with insights
  for adoption tracking.
- **Republishing** — revert problematic updates by republishing previous stable versions
  (works like version-control commits).

### Getting started

```sh
npm install --global eas-cli
eas login
eas update:configure
```

`npx install-expo-modules@latest` appears in the guide only for **bare React Native projects
being adopted into Expo** — skip it in an existing Expo project.

`eas update:configure` updates **app.json** with `runtimeVersion` and `updates.url`, and adds
`extra.eas.projectId`. It also adds native config:

**Android (AndroidManifest.xml):**
```xml
<meta-data android:name="expo.modules.updates.EXPO_UPDATE_URL"
  android:value="https://u.expo.dev/your-project-id"/>
<meta-data android:name="expo.modules.updates.EXPO_RUNTIME_VERSION"
  android:value="@string/expo_runtime_version"/>
```

**iOS (Expo.plist):**
```xml
<key>EXUpdatesRuntimeVersion</key>
<string>1.0.0</string>
<key>EXUpdatesURL</key>
<string>https://u.expo.dev/your-project-id</string>
```

### Channel setup

**EAS Build users (eas.json):**
```json
{
  "build": {
    "preview": { "channel": "preview" },
    "production": { "channel": "production" }
  }
}
```

**CNG (Continuous Native Generation) users (app.json):**
```json
{
  "expo": {
    "updates": {
      "requestHeaders": {
        "expo-channel-name": "your-channel-name"
      }
    }
  }
}
```

**Bare Android project (AndroidManifest.xml):**
```xml
<meta-data android:name="expo.modules.updates.UPDATES_CONFIGURATION_REQUEST_HEADERS_KEY"
  android:value="{&quot;expo-channel-name&quot;:&quot;your-channel-name&quot;}"/>
```

**Bare iOS project (Expo.plist):**
```xml
<key>EXUpdatesRequestHeaders</key>
<dict>
  <key>expo-channel-name</key>
  <string>your-channel-name</string>
</dict>
```

### Publishing updates

```sh
eas update --channel [channel-name] --message "[message]" --environment [environment-name]
```

`--environment` is **required for projects on Expo SDK 55 or greater** (per the EAS CLI
reference) — it selects which server-side EAS environment variables are loaded during the
command. Other notable flags: `--auto` (use the current git branch and commit message),
`--branch`, `--rollout-percentage <1-100>`, `--emit-metadata`, `--platform android|ios|all`
(default `all`), `--private-key-path` (code signing), `--skip-bundler`, `--input-dir`
(default `dist`).

### Bundle diffing

Source: https://docs.expo.dev/eas-update/bundle-diffing/ (unversioned — applies to SDK 56 and
57 alike). **Beta.** Requires SDK 55+.

Two modes:

1. **Patches between updates** — enabled by default on SDK 56+ (see section 1 for the exact
   56.0.13 cutover). Devices already running a published update receive a bsdiff patch.
2. **Patches from the embedded bundle** — experimental opt-in, so a *fresh install* can get a
   patch on its very first update check. Set the env var per build profile:

   ```json eas.json
   {
     "build": {
       "production": {
         "env": { "EAS_UPDATE_EXPERIMENTAL_UPLOAD_EMBEDDED_BUNDLE": "1" }
       }
     }
   }
   ```

   EAS Build then uploads the embedded bundle automatically. Without EAS Build, upload it
   yourself:

   ```sh
   eas update:embedded:upload --platform <ios|android> \
     --bundle <path-to-jsbundle> --manifest <path-to-app.manifest> \
     --channel <name> [--build-id <id>]
   ```

   `--platform`, `--bundle`, `--manifest`, `--channel` are all required; `--build-id` is
   required when invoked from EAS Build.

Manage uploaded embedded bundles (ID = the manifest UUID from **app.manifest**):

| Command | Flags |
|---------|-------|
| `eas update:embedded:list` | `-p ios\|android`, `--runtime-version`, `--channel` (pass `all` to skip the prompt), `--limit` (default 25, max 50), `--after-cursor`, `--json` |
| `eas update:embedded:view <ID>` | `--json` |
| `eas update:embedded:delete <ID>` | `--json`, `--non-interactive` (safe to retry) |

**When a patch is actually served:** only if it is meaningfully smaller than the full bundle
and cheap enough to compute; otherwise EAS serves the full bundle. A patch is precomputed only
against the *second-newest* update on the channel — other base updates get a patch generated
on demand — and generation takes a few minutes after publishing.

**Verify:** the Update Details page on expo.dev, or `Updates.readLogEntriesAsync()` (look for
"patch successfully applied").

---

## 5. Convex Integration

Source for the command surface: `docs/ui/components/EASCLIReference/data/eas-cli-commands.json`
(eas-cli 21.2.0). Nothing in `expo/expo` ties this integration to SDK 56 or 57.

| Command | Purpose |
|---------|---------|
| `eas integrations:convex:connect` | Connect Convex to your Expo project |
| `eas integrations:convex:dashboard` | Open the Convex dashboard for the linked project |
| `eas integrations:convex:project` | Show the Convex project linked to the current app |
| `eas integrations:convex:project:delete` | Remove the project link from EAS servers |
| `eas integrations:convex:team` | Show Convex teams linked to the app owner account |
| `eas integrations:convex:team:delete [CONVEX_TEAM]` | Remove a Convex team link |
| `eas integrations:convex:team:invite [CONVEX_TEAM]` | Send a team invite to your verified email |

```sh
eas integrations:convex:connect [--non-interactive] \
  [--region aws-us-east-1|aws-eu-west-1] \
  [--team-name <value>]      # defaults to the EAS account name
  [--project-name <value>]   # defaults to the app slug
```

`:project:delete` and `:team:delete` also take `-y, --yes` and `--non-interactive`.

Behavior (**from the SDK 56 changelog blog post; not verifiable from `expo/expo` source**):
- Installs the `convex` package.
- Creates or reuses a Convex **team linked to your EAS account**.
- Sets up a Convex **project with a dev deployment**.
- Writes `CONVEX_DEPLOY_KEY` and `EXPO_PUBLIC_CONVEX_URL` to **.env.local**.
- Creates `EXPO_PUBLIC_CONVEX_URL` as an **EAS environment variable** across Production,
  Preview, and Development environments, so EAS Build picks it up automatically.

---

## Sources

- SDK 56 changelog (blog, unverifiable from source): https://expo.dev/changelog/sdk-56
- expo-sqlite SDK 56: https://docs.expo.dev/versions/v56.0.0/sdk/sqlite/
- expo-updates SDK 56: https://docs.expo.dev/versions/v56.0.0/sdk/updates/
- EAS Update introduction: https://docs.expo.dev/eas-update/introduction/
- EAS Update getting started: https://docs.expo.dev/eas-update/getting-started/
- Bundle diffing for EAS Update: https://docs.expo.dev/eas-update/bundle-diffing/
- Monorepo (**release branches `origin/sdk-56` / `origin/sdk-57`**, never `main`):
  `packages/expo-sqlite/src/`, `packages/expo-updates/src/`,
  `packages/@expo/config-plugins/src/utils/Updates.ts`,
  `packages/@expo/config-types/src/ExpoConfig.ts`, per-package `CHANGELOG.md`
- Version pins: `packages/expo/bundledNativeModules.json` on the release branch
- `docs/public/static/schemas/v56.0.0/app-config-schema.json` (app-config keys only;
  `native-modules.json` in the same directory is stale — do not use it for pins)

---

## SDK 57 delta

**The public JS/TS API of `expo-sqlite` and `expo-updates` is unchanged in SDK 57.**
`git diff origin/sdk-56 origin/sdk-57 -- packages/expo-updates/src packages/expo-sqlite/src`
returns **zero lines** — the shipped TypeScript is identical. `@expo/config-types`
(`packages/@expo/config-types/src/ExpoConfig.ts`) is byte-identical across the two release
branches as well, so the app-config surface is the same: the same 12 `updates.*` keys and the
same four runtime version policies. Everything in sections 2–5 above applies verbatim to
SDK 57. (The only difference in the generated docs data is a TypeDoc rendering artifact —
`SQLiteProvider` shown as `React.MemoExoticComponent<(props) => JSX.Element>` instead of
`React.NamedExoticComponent<SQLiteProviderProps>` — not an API change.)

Bundle diffing and the Convex integration are documented in unversioned pages / eas-cli and
are not SDK-57-specific — nothing there changed for 57.

### Already on the SDK 56 line (patch drift, not a 56 → 57 delta)

SDK 56 is still actively patched. Every fix below was backported, so **upgrading the patch,
not the SDK, is what you actually need**. Minimum SDK 56 patch versions:

| Fix | Package | Min. SDK 56 version | SDK 57 version |
|-----|---------|--------------------|----------------|
| `appVersion` policy honours `ios.version` / `android.version` (#47416) | `@expo/config-plugins` | **56.0.12** (2026-07-07) | 57.0.3 |
| Fatal JNI crash with `useLibSQL: true` (#46651) | `expo-sqlite` | **56.0.5** (2026-06-10) | inherited by 57.0.0 |
| `isUpdatePending` false positive after a no-op check/fetch (#47830) | `expo-updates` | **56.0.23** (2026-07-23) | 57.0.9 |
| More default `getConfig` exclusion packages (#47503) | `@expo/fingerprint` | **0.19.7** (2026-07-07) | 0.20.3 |
| Unstable fingerprint with iOS precompiled modules (#46466) | `@expo/fingerprint` | **0.19.4** (2026-06-05) | inherited by 0.20.0 |

Details and evidence for each are in the subsections below.

### Breaking

**Nothing.** Two **runtime-version** items are commonly mis-filed as 56 → 57 breaks; neither
is a delta:

- **The `appVersion` policy honouring `ios.version` / `android.version` is not a 57 delta.**
  PR #47416 was backported to every supported line: it shipped in `@expo/config-plugins`
  **56.0.12 — 2026-07-07** and **57.0.3 — 2026-07-07** with an identical changelog entry
  (cherry-picks cd33f5f38d4 on `sdk-56`, c014fff0d55 on `sdk-57`, plus `sdk-55`/`sdk-54`).
  Details in [section 3](#3-expo-updates-api-reference); the boundary that matters is your
  installed `@expo/config-plugins` version, not your SDK.
- **The `@expo/fingerprint` major bump changes nothing for the `fingerprint` policy.**
  `origin/sdk-57`'s `CHANGELOG.md` records `## 0.20.0 — 2026-06-25 _This version does not
  introduce any user-facing changes._`, and
  `packages/@expo/fingerprint/src/sourcer/Expo.ts` is byte-identical between `origin/sdk-56`
  and `origin/sdk-57`: `SourceSkips.ExpoConfigVersions` strips exactly `version`,
  `android.versionCode`, `ios.buildNumber` (lines 135–139) on **both** SDKs. The extension
  that also strips `ios.version` / `android.version` (#47219) exists only on `main` and is
  still `## Unpublished` — post-57, do not plan around it. Two other fingerprint fixes are
  likewise not deltas: "fixed unstable fingerprint for iOS precompiled modules" (#46466)
  shipped in **0.19.4 — 2026-06-05**, already inside the SDK 56 `~0.19.9` range, and "more
  default `getConfig` exclusion packages" (#47503) shipped in both **0.19.7** (SDK 56 line)
  and **0.20.3** (SDK 57 line) on 2026-07-07.
  *Do not attribute fingerprint **presets** (`strict`/`balanced`/`relaxed`, `--preset`, the
  `package` source type) to SDK 57* — those landed 2026-07-17, nine days after 57.0.0.

### New in 57

**Nothing user-facing.** `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-updates/src
packages/expo-sqlite/src` is empty; the only substantive native difference is an iOS
logging refactor (`ios/EXUpdates/Logging/UpdatesLogReader.swift` + `UpdatesLogger.swift` — a
test-injectable `init(category:)` and formatting, no behaviour change).

> The 56 → 57 diff also adds `packages/expo-updates/e2e/fixtures/custom_init/SceneDelegate.swift`,
> which subclasses `ExpoAppSceneDelegate`. **Ignore it.** It is an e2e test fixture, and
> `ExpoAppSceneDelegate` appears nowhere else in the `origin/sdk-57` tree — nor anywhere in
> the **published `expo@57.0.8` tarball**, which contains no scene files at all. The UIKit
> scene-based lifecycle (`SceneDelegate`, `UIApplicationSceneManifest`, `ExpoAppSceneDelegate`
> in the shipped `expo` package) is **not in SDK 57**; it landed after the 57 cut. Do not
> restructure your AppDelegate for it as part of a 56 → 57 upgrade.

**Do not attribute the following to SDK 57.** All three are `main`-only, landed after the 57
branch cut, and are still `## Unpublished`; expect them in SDK 58:

- **[iOS] expo-updates no longer copying embedded assets into the updates cache on first
  launch** (#47284; `ios/EXUpdates/AppLoader/EmbeddedAppLoader.swift`,
  `ios/EXUpdates/Database/UpdatesDatabase.swift`). The Podfile property
  `updatesCopyEmbeddedAssets` and its compile flag `EX_UPDATES_COPY_EMBEDDED_ASSETS`
  **do not exist in SDK 56 or 57** — `packages/expo-updates/ios/EXUpdates.podspec` on
  `origin/sdk-57` contains no reference to either, so setting the property in
  **ios/Podfile.properties.json** does nothing. There is no `updates.copyEmbeddedAssets`
  app-config key on any SDK.
- **[Android] `FileDownloader` switching to React Native's `OkHttpClientProvider`** (#46926,
  per the `main` changelog). `OkHttpClientProvider` appears in
  `android/.../loader/FileDownloader.kt` only on `origin/main`, so on SDK 56 **and** 57 the
  downloader still builds its own `OkHttpClient` and **RN interceptors do not apply to update
  downloads**.
- **[Android] `UpdatesLogReader.ONE_DAY_MILLISECONDS` corrected from `86400` to
  `86_400_000`** (#46182). Both release branches still ship `86400`, so the ~86-second log
  purge is live on SDK 56 and 57 — see the caveat in
  [section 3](#3-expo-updates-api-reference).

### Release-notes accounting

The aggregated `## 57.0.0 — 2026-07-08` release notes are a **rollup since 56.0.0**, not a
list of things SDK 56 users lack. Both entries below are already on the SDK 56 line.

- `expo-sqlite` has exactly **one** entry in the aggregated 57.0.0 release notes
  (`CHANGELOG.md` § `## 57.0.0 — 2026-07-08`): "Fixed a fatal JNI crash on Android when using
  `useLibSQL: true`, caused by the libSQL session bindings still declaring `byte[]` signatures
  after #42638 switched Kotlin and the default native bindings to `ByteBuffer`" (#46651).
  **Not an SDK 57 gain:** it shipped in `expo-sqlite` **56.0.5 — 2026-06-10**, which is the
  SDK 56 pin. The package's own `57.0.0` and `57.0.1` both say *"This version does not
  introduce any user-facing changes."*
- `expo-updates` has **zero** entries in the 57.0.0 release notes, and `origin/sdk-57`'s
  `packages/expo-updates/CHANGELOG.md` records exactly **one** user-facing change across the
  whole 57.0.0 – 57.0.10 range: the `isUpdatePending` false-positive fix after a no-op
  check/fetch (#47830), in **57.0.9 — 2026-07-22**. **Also not a delta:** the same fix shipped
  on the SDK 56 line in **56.0.23 — 2026-07-23**, inside the `~56.0.23` pin. Both lines have
  it; upgrade the patch, not the SDK.
- **Do not attribute to 57.0.0** (landed after 2026-07-08): the native update-event state
  rename `downloadComplete → downloadCompleteUnavailable` (#47902, 2026-07-18) and
  `[iOS] expo.updates.devClientPackage` override (#48020, 2026-07-24) — neither is on
  `origin/sdk-57` at all. #47830 is a 57.0.9 / 56.0.23 patch, not part of the 57.0.0 release
  itself.

### Version pins (56 -> 57)

From `packages/expo/bundledNativeModules.json` on `origin/sdk-56` / `origin/sdk-57` — this
file ships inside the `expo` package and is what `expo install` actually resolves against.
Do **not** use `docs/public/static/schemas/v5x.0.0/native-modules.json` (stale) or
`packages/*/package.json` on `main` (that is SDK 58 in progress).

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-sqlite` | `~56.0.5` | `~57.0.1` |
| `expo-updates` | `~56.0.23` | `~57.0.10` |
| `@expo/fingerprint` | `~0.19.9` | `~0.20.6` |

Both SDK lines are still being patched, so the `~` ranges keep moving; the numbers above are
the pins as of the SDK 57 line's current state.

### Docs-snapshot warning

The `v57.0.0/sdk/sqlite` page is **stale relative to v56** in two places — treat the v56
forms as correct for both SDKs: `customBuildFlags` is a **string**, not an array
(`packages/expo-sqlite/plugin/src/withSQLite.ts`), and `getFirstAsync()` returns the row
directly (`result['COUNT(*)']`), not `result.rows[0]['COUNT(*)']` (PR #46439). The
`v57.0.0/sdk/updates` page differs from v56 only in prose wording ("bare React Native app"
vs "existing React Native project") and table padding.
