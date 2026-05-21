# Expo SDK 56 — expo-sqlite, expo-updates / EAS Update, Hermes Bytecode Diffing, Convex

Official documentation reference for the Expo SDK 56 knowledge base. Captures APIs, config
keys, and code examples for `expo-sqlite`, `expo-updates` / EAS Update (including Hermes
bytecode diffing), and the Convex integration.

---

## 1. SDK 56 Changelog Highlights

Source: https://expo.dev/changelog/sdk-56

### expo-sqlite (new in SDK 56)

- **Native ArrayBuffer for blob columns** — replaces the legacy `JavaScriptArrayBuffer`
  with native blob support on both Android (PR #42640) and iOS (PR #42642).
- **Statement bind params** (PR #42639) — parameterized query execution on prepared statements.
- **Session changesets** (PR #42638) — track database modifications within sessions
  (via the SQLite session extension).

### Hermes Bytecode Diffing (expo-updates / EAS Update)

- Originally opt-in in SDK 55, **now enabled by default in SDK 56**.
- EAS Update served diffed Hermes bundles that were **on average 58% smaller** than the full
  bundle they replaced.
- Current mechanism: downloads binary patches against the previously installed bytecode
  rather than full bundles.
- Planned enhancement: Expo is extending bytecode diffing to **patch against the embedded
  bundle shipped in your native build**, targeted as an **opt-in feature in an SDK 56 patch
  release** in the coming months.
- Opt-out: set `"enableBsdiffPatchSupport": false` in the `updates` block of **app.json**.

### Convex Integration

EAS automates Convex backend provisioning and linking. Run:

```sh
eas integrations:convex:connect
```

This command:
- Installs the `convex` package.
- Creates or reuses a Convex team linked to your EAS account.
- Sets up a project with a dev deployment.
- Writes `CONVEX_DEPLOY_KEY` and `EXPO_PUBLIC_CONVEX_URL` to **.env.local**.
- Creates `EXPO_PUBLIC_CONVEX_URL` as an EAS environment variable across Production, Preview,
  and Development environments for automatic EAS Build detection.

---

## 2. expo-sqlite API Reference

Source: https://docs.expo.dev/versions/v56.0.0/sdk/sqlite/

### Opening a database

```ts
SQLite.openDatabaseAsync(databaseName, options?, directory?)  // Promise<SQLiteDatabase>
SQLite.openDatabaseSync(databaseName, options?, directory?)   // SQLiteDatabase (blocks thread)
```

Common options (`SQLiteOpenOptions`):

| Option | Purpose |
|--------|---------|
| `enableFTS` | Enable full-text search (default: `true`) |
| `useSQLCipher` | Encrypted database support |
| `enableChangeListener` | Enable database change events |

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

### Tagged template literals

```ts
const users = await sql<User>`SELECT * FROM users WHERE age > ${21}`;
const user  = await sql<User>`SELECT * FROM users WHERE id = ${id}`.first();
```

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

Relevant types:
- `SQLiteBindBlobValue` — acceptable blob binding parameter type (supports `Uint8Array`).
- `Changeset` — type alias for `Uint8Array`.

### Session extension / changesets (SDK 56)

Create a session from the database:

```ts
db.createSessionAsync(dbName?)  // dbName default: 'main' → Promise<SQLiteSession>
db.createSessionSync(dbName?)   // → SQLiteSession (blocks thread)
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

- `expo-sqlite/kv-store` — drop-in AsyncStorage replacement with synchronous methods.
- `expo-sqlite/localStorage/install` — web-compatible `localStorage` implementation.
- `loadExtensionAsync()` — load extensions (e.g. sqlite-vec).
- `serializeAsync()` / `deserializeDatabaseAsync()` — serialize/restore a database.

---

## 3. expo-updates API Reference

Source: https://docs.expo.dev/versions/latest/sdk/updates/

### `useUpdates()` hook

Returns `UseUpdatesReturnType`:

- `currentlyRunning: CurrentlyRunningInfo`
- `isUpdateAvailable: boolean`
- `isUpdatePending: boolean`
- `isChecking: boolean`
- `isDownloading: boolean`
- `downloadProgress?: number`
- `availableUpdate?: UpdateInfo`
- `downloadedUpdate?: UpdateInfo`
- `checkError?: Error`
- `downloadError?: Error`

```tsx
const { currentlyRunning, isUpdateAvailable } = Updates.useUpdates();
if (isUpdateAvailable) {
  await Updates.fetchUpdateAsync();
}
```

### Core methods

| Method | Signature | Notes |
|--------|-----------|-------|
| `checkForUpdateAsync()` | `Promise<UpdateCheckResult>` | Queries server availability without downloading. Rejects in dev mode, Expo Go, or on network errors. |
| `fetchUpdateAsync()` | `Promise<UpdateFetchResult>` | Downloads latest update to device storage. Call `reloadAsync()` to apply immediately. Rejects in dev mode/Expo Go/network errors. |
| `reloadAsync(options?)` | `options?: { reloadScreenOptions: ReloadScreenOptions }` → `Promise<void>` | Reloads app using newest downloaded update. Rejects in dev mode, Expo Go, or when updates disabled. |

### App config keys (`expo.updates` in app.json)

**Required:**
- `updates.url` (string) — remote update service endpoint.
- `runtimeVersion` (string | object) — build compatibility identifier.

**Optional:**
- `updates.enabled` (boolean, default `true`)
- `updates.checkAutomatically` (`"ON_LOAD" | "ON_ERROR_RECOVERY" | "NEVER" | "WIFI_ONLY"`, default `"ON_LOAD"`)
- `updates.fallbackToCacheTimeout` (number, default `0`) — ms before using cached update
- `updates.requestHeaders` (object) — custom HTTP headers
- `updates.useEmbeddedUpdate` (boolean, default `true`)
- `updates.codeSigningCertificate` (string)
- `updates.codeSigningMetadata` (object)
- `updates.disableAntiBrickingMeasures` (boolean, default `false`)
- **`updates.enableBsdiffPatchSupport` (boolean, default `true`)** — controls Hermes bytecode
  diffing / bsdiff patching; set to `false` to opt out.

**Runtime version policies:**
- `"appVersion"` — uses the app `version` field.
- `"nativeVersion"` — combines `version` with `buildNumber`/`versionCode`.
- `"fingerprint"` — auto-calculated from project state.

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
# Install the EAS CLI
npm install --global eas-cli

# Ensure project has the expo package
npx install-expo-modules@latest

# Authenticate
eas login
eas whoami

# Configure updates
eas update:configure
```

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

### Publishing updates

```sh
eas update --channel [channel-name] --message "[message]" --environment [environment-name]
```

> Note: the EAS Update introduction/getting-started and the optimize-assets pages do not
> themselves document Hermes bytecode diffing; that information comes from the SDK 56
> changelog and the `expo-updates` config reference (`enableBsdiffPatchSupport`). See sections
> 1 and 3.

---

## 5. Convex Integration (SDK 56)

Source: https://expo.dev/changelog/sdk-56

EAS provisions and links Convex backends via a single command:

```sh
eas integrations:convex:connect
```

Behavior:
- Installs the `convex` package.
- Creates or reuses a Convex **team linked to your EAS account**.
- Sets up a Convex **project with a dev deployment**.
- Writes `CONVEX_DEPLOY_KEY` and `EXPO_PUBLIC_CONVEX_URL` to **.env.local**.
- Creates `EXPO_PUBLIC_CONVEX_URL` as an **EAS environment variable** across Production,
  Preview, and Development environments, so EAS Build picks it up automatically.

---

## Sources

- SDK 56 changelog: https://expo.dev/changelog/sdk-56
- expo-sqlite SDK 56: https://docs.expo.dev/versions/v56.0.0/sdk/sqlite/
- expo-updates: https://docs.expo.dev/versions/latest/sdk/updates/
- EAS Update introduction: https://docs.expo.dev/eas-update/introduction/
- EAS Update getting started: https://docs.expo.dev/eas-update/getting-started/
- EAS Update optimize assets (checked, not relevant to bytecode diffing): https://docs.expo.dev/eas-update/optimize-assets/
