# Expo SDK 56 — File System & Networking (expo-file-system, expo/fetch)

> Knowledge-base reference for Expo SDK 56. Captures the new task-based transfer APIs,
> the object-oriented File/Directory/Paths API, the async copy/move breaking change,
> multi-file picking, experimental file-system watching, and the `expo/fetch` changes
> (decompression, AbortSignal improvements, and `expo/fetch` becoming the default
> `globalThis.fetch`).

**Verified against:** `expo-file-system` `~56.0.8` (SDK 56 pin) / `~57.0.1` (SDK 57 pin) — both from
`packages/expo/bundledNativeModules.json` on `origin/sdk-56` / `origin/sdk-57`, which is what
`expo install` resolves against — and `expo` `56.0.17` / `57.0.8`.
This whole domain is **API-identical between SDK 56 and SDK 57** —
see [SDK 57 delta](#sdk-57-delta) at the end.

**Primary sources**

- Changelog: https://expo.dev/changelog/sdk-56
- Beta changelog: https://expo.dev/changelog/sdk-56-beta
- FileSystem API: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/ (SDK 57: `/versions/v57.0.0/sdk/filesystem/`)
- Legacy FileSystem API: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem-legacy/
- expo/fetch API: https://docs.expo.dev/versions/v56.0.0/sdk/expo/ (and `/versions/latest/sdk/expo/`)
- Monorepo (authoritative for signatures): `packages/expo-file-system/src/`, `packages/expo/src/winter/`
- Package CHANGELOGs: `packages/expo-file-system/CHANGELOG.md`, `packages/expo/CHANGELOG.md`

> **Caveat on the published docs page.** `docs/public/static/data/v56.0.0/expo-file-system.json`
> is frozen at the SDK 56 beta cut and over-lists members on `DownloadTask` / `UploadTask`
> (it documents the internal `SharedObject` subclass, not the JS wrapper). Trust
> `packages/expo-file-system/src/NetworkTasks.ts` over that page. See §3.

---

## 0. What models most often get wrong

- **The legacy names re-exported from the package root THROW at runtime.**
  `import * as FileSystem from 'expo-file-system'; FileSystem.readAsStringAsync(...)` type-checks
  and then throws. Use `expo-file-system/legacy`. See §2 "Legacy API".
- `documentDirectory` / `cacheDirectory` string constants exist **only** on the legacy entry point.
  The new API uses `Paths.document` / `Paths.cache`, which are `Directory` objects, not strings.
- **`copy()` and `move()` are async** in SDK 56+. `copySync()` / `moveSync()` are the sync forms.
- `DownloadTask.downloadAsync()` / `resumeAsync()` resolve to **`File | null`** (`null` = paused).
- `DownloadTask` has **no** `resume()`, `removeListener()`, `removeAllListeners()` or
  `listenerCount()` — those are on the native shared object, not the JS class.
- The `md5` property is **not** deprecated; `modificationTime` is (use `lastModified`).

---

## 1. expo-file-system — What changed in SDK 56

Source: https://expo.dev/changelog/sdk-56 and https://expo.dev/changelog/sdk-56-beta

- **Task-based upload/download APIs** added: `file.createUploadTask()` and `File.createDownloadTask()`.
  > "The new API also adds task-based upload and download APIs: `file.createUploadTask()` and `File.createDownloadTask()`."
  > "These bring back support for long-running transfers from the legacy file-system module, including upload progress, cancellation, and resumable downloads."
- **Convenience upload wrapper:**
  > "For simpler uploads, `File.upload()` provides a convenience wrapper when you do not need to manage an upload task directly."
- **Overwrite option** for copy/move:
  > "copy/move operations accept an `overwrite` option."
- **Multi-file picking:**
  > "`File.pickFileAsync()` supports selecting multiple files and multiple MIME types" — closer parity with `expo-document-picker`.
- **Experimental file-system event watching:**
  > "We've also added experimental file-system event watching with `File.watch()` and `Directory.watch()`" — subscribe to changes without polling.
- **Reliability fixes:** large-file `md5` hashing memory usage, Android SAF copy/move support, and `totalDiskSpace` reporting on iOS.

### BREAKING — copy() and move() are now async

Source: https://expo.dev/changelog/sdk-56-beta

> "Async `copy()` and `move()` in `expo-file-system`: these methods on `File` and `Directory`
> are now asynchronous and return a Promise. Use `copySync()` and `moveSync()` for synchronous behavior."

- `File.copy()` / `File.move()` → return `Promise<void>`.
- `Directory.copy()` / `Directory.move()` → return `Promise<void>`.
- Synchronous equivalents: `copySync()` / `moveSync()`.

### SDK 56 patch-level changes after 56.0.0

Source: `git show origin/sdk-56:packages/expo-file-system/CHANGELOG.md`. The SDK 56 pin is
`~56.0.8` (`bundledNativeModules.json` on `origin/sdk-56`); **treat 56.0.7 as the practical
floor** (56.0.6 carries Android security fixes, 56.0.7 the Expo Go directory change).

| Version | Change |
| --- | --- |
| 56.0.6 (2026-05-19) | **New:** `File.json()` and `File.formData()` (#45685). **Security [Android]:** path-traversal fix in `createFile` / `createDirectory` / `rename`; missing permission checks added to `delete()` and `openHandle()` (#45967). |
| 56.0.7 (2026-05-20) | **BREAKING [Android][Expo Go]:** `Paths.cache` and `Paths.document` now resolve to per-experience isolated directories (#45977). Data written by an app running under an older Expo Go appears to "vanish" after the update — it is in the old, now-unreferenced directory. Does not affect dev builds or standalone builds. |
| 56.0.8 (2026-06-10) | No user-facing changes. |

---

## 2. FileSystem API Reference (SDK 56)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/

```ts
import { File, Directory, Paths } from 'expo-file-system';
```

### Class: `File`

Constructor accepts path segments and/or a `Directory`. The file need not already exist.

```ts
const file = new File(Paths.cache, 'subdirName', 'file.txt');
```

**Properties**

| Property | Type | Notes |
| --- | --- | --- |
| `exists` | `boolean` | |
| `size` | `number` | bytes; `0` if nonexistent |
| `uri` | `string` | read-only |
| `name` | `string` | includes extension |
| `type` | `string` | MIME type |
| `extension` | `string` | e.g. `.png` |
| `lastModified` | `number \| null` | ms since epoch |
| `modificationTime` | `number \| null` | **DEPRECATED** — use `lastModified` (still works) |
| `creationTime` | `number \| null` | Android API 26+ |
| `md5` | `string \| null` | **not** deprecated; `null` if the file is absent or unreadable |
| `parentDirectory` | `Directory` | |
| `contentUri` | `string` | Android only |

**Core methods**

```ts
create(options?: FileCreateOptions): void
write(content: string | Uint8Array, options?: FileWriteOptions): void
text(): Promise<string>
textSync(): string
bytes(): Promise<Uint8Array>
bytesSync(): Uint8Array
base64(): Promise<string>
base64Sync(): string
arrayBuffer(): Promise<ArrayBuffer>
json(): Promise<any>                        // added 56.0.6 (#45685)
formData(): ReturnType<Response['formData']>  // added 56.0.6 — File satisfies web Blob augmentations
delete(): void
rename(newName: string): void
copy(destination: File | Directory, options?: RelocationOptions): Promise<void>     // async (BREAKING)
copySync(destination: File | Directory, options?: RelocationOptions): void
move(destination: File | Directory, options?: RelocationOptions): Promise<void>     // async (BREAKING)
moveSync(destination: File | Directory, options?: RelocationOptions): void
info(options?: InfoOptions): FileInfo
open(mode?: FileMode): FileHandle
slice(start?: number, end?: number, contentType?: string): Blob
stream(): ReadableStream<Uint8Array>
readableStream(): ReadableStream<Uint8Array>
writableStream(): WritableStream<Uint8Array>
```

**Network methods**

```ts
static downloadFileAsync(url: string, destination: File | Directory, options?: DownloadOptions): Promise<File>
createDownloadTask(url: string, destination: File | Directory, options?: DownloadTaskOptions): DownloadTask
createUploadTask(url: string, options?: UploadOptions): UploadTask
upload(url: string, options?: UploadOptions): Promise<UploadResult>
```

Note: `createDownloadTask` is shown as a `static` method on `File`; `createUploadTask`
and `upload` are instance methods (the instance is the file being uploaded).
`downloadFileAsync` takes `DownloadOptions`; `headers` / `idempotent` date back to SDK 55, and
`onProgress` / `signal` were added in 56.0.0 (#43053) — see §4. Without `idempotent: true` it
**throws** if the target file already exists (unchanged since 55).

**File picking** (multi-file capable in SDK 56)

```ts
static pickFileAsync(options?: PickSingleFileOptions): Promise<PickSingleFileResult>
static pickFileAsync(options: PickMultipleFilesOptions): Promise<PickMultipleFilesResult>   // overload
static pickFileAsync(initialUri?: string, mimeType?: string): Promise<File | File[]>        // deprecated overload
```

**Watching** (experimental)

```ts
watch(callback: (event: WatchEvent<File>) => void, options?: WatchOptions): WatchSubscription
```

### Class: `Directory`

```ts
const directory = new Directory(Paths.cache, 'subdirName');
```

**Properties**

| Property | Type | Notes |
| --- | --- | --- |
| `exists` | `boolean` | |
| `size` | `number \| null` | bytes; `null` if nonexistent |
| `uri` | `string` | read-only |
| `name` | `string` | |
| `parentDirectory` | `Directory` | |

**Methods**

```ts
create(options?: DirectoryCreateOptions): void
list(): (File | Directory)[]
createFile(name: string, mimeType: string | null): File
createDirectory(name: string): Directory
delete(): void                                                                       // recursive
rename(newName: string): void
copy(destination: File | Directory, options?: RelocationOptions): Promise<void>      // async (BREAKING)
copySync(destination: File | Directory, options?: RelocationOptions): void
move(destination: File | Directory, options?: RelocationOptions): Promise<void>      // async (BREAKING)
moveSync(destination: File | Directory, options?: RelocationOptions): void
info(): DirectoryInfo
watch(callback: (event: WatchEvent<File | Directory>) => void, options?: WatchOptions): WatchSubscription
static pickDirectoryAsync(initialUri?: string): Promise<Directory>   // counterpart to File.pickFileAsync
```

`Directory.pickDirectoryAsync()` caveats (from the TSDoc in
`packages/expo-file-system/src/internal/NativeFileSystem.types.ts`):

- **iOS** — the selected directory grants read/write access **for the current app session only**.
  After a restart you must prompt the user again to regain access.
- **Android** — the returned `uri` is a SAF `content://` URI, not a `file://` path.

### Class: `Paths`

```ts
import { Paths } from 'expo-file-system';
```

**Properties**

```ts
cache: Directory                                  // system cache, may be deleted by OS
document: Directory                               // persistent storage
// [Android][Expo Go] both are experience-isolated as of expo-file-system 56.0.7 — see §1
bundle: Directory                                 // bundled assets
appleSharedContainers: Record<string, Directory>  // iOS
totalDiskSpace: number                            // bytes (iOS reporting fixed in SDK 56)
availableDiskSpace: number                        // bytes
```

**Path utility methods**

```ts
basename(path: string | File | Directory, ext?: string): string
dirname(path: string | File | Directory): string
extname(path: string | File | Directory): string
normalize(path: string | File | Directory): string
join(...paths: (string | File | Directory)[]): string
isAbsolute(path: string | File | Directory): boolean
relative(from: string | File | Directory, to: string | File | Directory): string
parse(path: string | File | Directory): { base; dir; ext; name; root }
info(...uris: string[]): PathInfo
```

### Class: `FileHandle` (returned by `File.open()`)

```ts
// Properties
offset: number | null   // current byte position; null if closed
size: number | null     // file size in bytes; null if closed

// Methods
readBytes(length: number): Uint8Array   // synchronous in SDK 56 AND 57
writeBytes(bytes: Uint8Array): void     // synchronous in SDK 56 AND 57
close(): void
```

### Legacy API — `expo-file-system/legacy`

Source: `packages/expo-file-system/src/legacyWarnings.ts`, `package.json` `exports['./legacy']`.
Docs: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem-legacy/

The package root still **exports the legacy function names**, but they are throwing deprecation
shims — they compile, log a warning, and then throw
`Method <name> imported from "expo-file-system" is deprecated.` at runtime:

```
getInfoAsync           readAsStringAsync     writeAsStringAsync   deleteAsync
moveAsync              copyAsync             makeDirectoryAsync   readDirectoryAsync
getContentUriAsync     downloadAsync         uploadAsync          createDownloadResumable
createUploadTask       getFreeDiskStorageAsync                    getTotalDiskCapacityAsync
deleteLegacyDocumentDirectoryAndroid
```

The working legacy API is a real subpath export (identical in SDK 56 and 57):

```ts
import * as FileSystem from 'expo-file-system/legacy';

FileSystem.documentDirectory;   // string — ONLY on the legacy entry point
FileSystem.cacheDirectory;      // string
FileSystem.bundleDirectory;     // string
await FileSystem.readAsStringAsync(uri);
```

### Config plugin

Source: `docs/pages/versions/v56.0.0/sdk/filesystem.mdx` (identical in v57.0.0).

```json app.json
{
  "expo": {
    "plugins": [
      ["expo-file-system", { "supportsOpeningDocumentsInPlace": true, "enableFileSharing": true }]
    ]
  }
}
```

| Prop | Platform | Default | Effect |
| --- | --- | --- | --- |
| `supportsOpeningDocumentsInPlace` | iOS | `false` | Sets `LSSupportsOpeningDocumentsInPlace` in **Info.plist** — app can open documents in place. |
| `enableFileSharing` | iOS | `false` | Sets `UIFileSharingEnabled` in **Info.plist** — exposes the app's Documents directory in the Files app / iTunes File Sharing. |

Without CNG, add those two keys to `ios/[app]/Info.plist` manually.

---

## 3. Task-Based Transfer APIs

Source: `packages/expo-file-system/src/NetworkTasks.ts` (`UploadTask` at :82, `DownloadTask` at :220)
— byte-identical on `sdk-56` and `sdk-57`.

> **Do not trust the published v56 API page for these two classes.** It is generated from a
> beta-era snapshot that documented the internal `SharedObject` subclass and therefore lists
> `emit`, `listenerCount`, `removeAllListeners`, `removeListener`, `resume`, `start`,
> `startObserving`, `stopObserving`. **None of those exist on the JS wrapper** — calling them
> throws `TypeError: ... is not a function`. The lists below are the complete public surface.

### Class: `DownloadTask`

Supports progress, pause/resume, cancellation, and resumable downloads.

```ts
// Property
state: DownloadTaskState  // 'idle' | 'active' | 'paused' | 'completed' | 'cancelled' | 'error'

// Methods
downloadAsync(): Promise<File | null>   // null = paused before completion; rejects on failure/cancel
pause(): void                           // only from 'active'; resolves the pending promise with null
pauseAsync(): Promise<void>             // pause() + wait until resume data is available
resumeAsync(): Promise<File | null>     // only from 'paused'; null if paused again
cancel(): void                          // no-op once completed/cancelled/error
savable(): DownloadPauseState           // only from 'paused'; persist across restarts
static fromSavable(state: DownloadPauseState, options?: DownloadTaskOptions): DownloadTask
addListener(eventName: 'progress', listener: (data: DownloadProgress) => void): EventSubscription
release(): void                         // manually release the native handle
```

Notes:

- `savable()` and `pause()` assert on state — calling `pause()` when not `active`, or `savable()` /
  `resumeAsync()` when not `paused`, throws.
- `fromSavable()` throws if the saved state has no `resumeData` (i.e. the download was paused
  before any bytes arrived). New `options` headers override saved headers of the same name.
- Prefer the `onProgress` option over `addListener` unless you need manual subscription control.

### Class: `UploadTask`

```ts
// Property
state: UploadTaskState   // 'idle' | 'active' | 'completed' | 'cancelled' | 'error' (no 'paused')

// Methods
uploadAsync(): Promise<UploadResult>   // callable once, only from 'idle'
cancel(): void
addListener(eventName: 'progress', listener: (data: UploadProgress) => void): EventSubscription
release(): void
```

`uploadAsync()` resolves for **any** completed HTTP response including non-2xx — check
`result.status`. It rejects only when the file cannot be read, the request fails, or the task is
cancelled.

---

## 4. FileSystem Type Definitions & Option Shapes

Source: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/

```ts
// FileCreateOptions
{
  overwrite?: boolean;        // Default: false
  intermediates?: boolean;    // Create parent dirs. Default: false
}

// DirectoryCreateOptions
{
  overwrite?: boolean;        // Default: false
  intermediates?: boolean;    // Default: false
  idempotent?: boolean;       // No error if exists. Default: false
}

// FileWriteOptions
{
  encoding?: EncodingType | 'utf8' | 'base64';  // Default: EncodingType.UTF8
  append?: boolean;                              // Default: false
}

// RelocationOptions  (used by copy/copySync/move/moveSync)
{
  overwrite?: boolean;  // Default: false
}

// DownloadOptions  (File.downloadFileAsync only — NOT the same type as DownloadTaskOptions)
{
  headers?: Record<string, string>;
  idempotent?: boolean;                          // Default: false. true = overwrite an existing
                                                 // target; false = throw if it already exists
  onProgress?: (data: DownloadProgress) => void;
  signal?: AbortSignal;                          // aborting rejects with AbortError
}

// DownloadTaskOptions
{
  headers?: Record<string, string>;
  onProgress?: (data: DownloadProgress) => void;
  signal?: AbortSignal;                          // cancellation via AbortSignal
  sessionType?: 'background' | 'foreground';     // iOS only. Default: 'background'
}

// DownloadProgress
{
  bytesWritten: number;
  totalBytes: number;   // -1 if Content-Length unavailable
}

// DownloadPauseState
{
  url: string;
  fileUri: string;
  isDirectory: boolean;
  resumeData?: string;
  headers?: Record<string, string>;
}

// UploadOptions
{
  uploadType?: UploadType.BINARY_CONTENT | UploadType.MULTIPART;
  httpMethod?: 'POST' | 'PUT' | 'PATCH';   // Default: 'POST'
  mimeType?: string;
  fieldName?: string;                       // Default: 'file'
  parameters?: Record<string, string>;      // Multipart only
  headers?: Record<string, string>;
  onProgress?: (data: UploadProgress) => void;
  signal?: AbortSignal;                     // cancellation via AbortSignal
  sessionType?: 'background' | 'foreground'; // iOS only. Default: 'background'
}

// UploadProgress
{
  bytesSent: number;
  totalBytes: number;
}

// UploadResult
{
  status: number;
  headers: Record<string, string>;
  body: string;
}

// InfoOptions
{
  md5?: boolean;  // Default: false
}

// FileInfo
{
  exists: boolean;
  size?: number;
  modificationTime?: number;   // ms since epoch
  creationTime?: number;       // ms since epoch (Android 26+)
  md5?: string;                // if md5 option was true
  uri?: string;
}

// DirectoryInfo
{
  exists: boolean;
  size?: number;
  modificationTime?: number;
  creationTime?: number;
  files?: string[];
  uri?: string;
}

// PickSingleFileOptions
{
  initialUri?: string;
  mimeTypes?: string | string[];  // Default: '*/*'
  multipleFiles?: false;
}

// PickMultipleFilesOptions
{
  initialUri?: string;
  mimeTypes?: string | string[];  // Default: '*/*'
  multipleFiles: true;
}

// PickSingleFileResult
{ canceled: false; result: File } | { canceled: true; result: null }

// PickMultipleFilesResult
{ canceled: false; result: File[] } | { canceled: true; result: null }

// WatchOptions
{
  debounce?: number;            // ms. Default: 100
  events?: WatchEventType[];    // filter event types
}

// WatchEvent<T>
{
  type: 'created' | 'modified' | 'deleted' | 'renamed';
  target: T;                    // File or Directory that changed
  newTarget?: T;                // for rename events (Android)
  nativeEventFlags?: number;    // platform-specific flags
}

// WatchSubscription
{
  remove(): void;               // stop watching
}

// PathInfo
{
  exists: boolean;
  isDirectory: boolean | null;  // null if path doesn't exist
}
```

### Enums

```ts
// FileMode
FileMode.ReadOnly  = "r"    // read-only
FileMode.ReadWrite = "rw"   // read/write (not SAF content://)
FileMode.WriteOnly = "w"    // write-only
FileMode.Append    = "wa"   // append-only
FileMode.Truncate  = "wt"   // truncate to zero length

// EncodingType
EncodingType.UTF8   = "utf8"
EncodingType.Base64 = "base64"

// UploadType
UploadType.BINARY_CONTENT = 0
UploadType.MULTIPART      = 1
```

### Constants

```ts
import { DEFAULT_DEBOUNCE_MS } from 'expo-file-system';   // === 100 (ms, watch debouncing)
```

There is **no** `FileSystem` namespace export on the new entry point — `DEFAULT_DEBOUNCE_MS` is a
named top-level export: `src/index.ts` re-exports it from `./FileSystem.types`, which barrels
`FileSystemWatcher.types.ts` where it is defined (`= 100`). It is tagged `@hidden`, so it does not
appear on the docs page.

---

## 5. FileSystem Code Examples

Only the recursive-listing snippet below is verbatim from
https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/ (Collapsible "Listing directory contents
recursively"); the download, upload and watch snippets are authored against
`packages/expo-file-system/src/NetworkTasks.ts` and `src/FileSystemWatcher.types.ts`.

**Download with progress**

```ts
const dest = new File(Paths.document, 'video.mp4');
const downloadTask = File.createDownloadTask(url, dest, {
  onProgress: ({ bytesWritten, totalBytes }) => {
    console.log(`${bytesWritten}/${totalBytes} bytes`);
  }
});
const file = await downloadTask.downloadAsync();   // File | null — null if paused mid-flight
if (file) {
  console.log(file.uri);
}
```

**Upload with progress**

```ts
const file = new File(Paths.document, 'photo.jpg');
const uploadTask = file.createUploadTask(url, {
  uploadType: UploadType.MULTIPART,
  headers: { Authorization: 'Bearer token' },
  onProgress: ({ bytesSent, totalBytes }) => {
    console.log(`${bytesSent}/${totalBytes} bytes`);
  }
});
const result = await uploadTask.uploadAsync();
```

**File watching (experimental)**

```ts
const subscription = file.watch((event) => {
  console.log(`${event.type}: ${event.target.uri}`);
});
subscription.remove();
```

**Recursive directory listing**

```ts
import { Directory, Paths } from 'expo-file-system';

function printDirectory(directory: Directory, indent: number = 0) {
  console.log(`${' '.repeat(indent)} + ${directory.name}`);
  const contents = directory.list();
  for (const item of contents) {
    if (item instanceof Directory) {
      printDirectory(item, indent + 2);
    } else {
      console.log(`${' '.repeat(indent + 2)} - ${item.name} (${item.size} bytes)`);
    }
  }
}

try {
  printDirectory(new Directory(Paths.cache));
} catch (error) {
  console.error(error);
}
```

---

## 6. expo/fetch — Networking (SDK 56)

Sources:
https://docs.expo.dev/versions/v56.0.0/sdk/expo/ ,
https://expo.dev/changelog/sdk-56 ,
`git show origin/sdk-56:packages/expo/CHANGELOG.md` (do **not** read the `main` copy — `main` is
SDK 58 in progress and its `## Unpublished` entries have never shipped in any SDK)

`expo/fetch` provides a **WinterCG/WinterTC-compliant** Fetch API (per the WHATWG Fetch
spec) that works consistently across web and mobile.

```ts
import { fetch } from 'expo/fetch';
```

### BREAKING — expo/fetch becomes the default `globalThis.fetch`

Source: https://expo.dev/changelog/sdk-56

> "`expo/fetch` is now installed as the default implementation of `globalThis.fetch`,
> providing a WinterTC-compliant API and improved performance."

> "To opt out, set `EXPO_PUBLIC_USE_RN_FETCH=1` in your **.env** file."

Behavior notes:
- On native (Android/iOS), any unimported `fetch(...)` call now uses the WinterCG-compliant
  `expo/fetch` implementation.
- Named imports from `expo/fetch` keep working **regardless** of `EXPO_PUBLIC_USE_RN_FETCH`.
- The runtime accepts **two** values — `packages/expo/src/winter/runtime.native.ts:41-42` reads
  `process.env.EXPO_PUBLIC_USE_RN_FETCH === '1' || === 'true'`. Anything else (e.g. `yes`) is
  ignored and the global stays `expo/fetch`.
- **Requires `babel-preset-expo >= 56.0.16` (SDK 56) / `>= 57.0.0`.** Before that, the env var was
  not inlined inside `node_modules`, so in a production build the opt-out silently did nothing for
  library code (#46986, `packages/babel-preset-expo/src/configs/expo.ts`). A dev build would honour
  it and the release build would not — a classic "works locally" trap.
- CHANGELOG (preview.0): "Use `expo/fetch` as default fetch."

### Decompression (Android)

Source: expo package CHANGELOG (v56.0.0)

> "[fetch][Android] Added `brotli`, `gzip`, and `zstd` decompression support."

Compressed HTTP responses in these encodings are decompressed automatically on Android.

### AbortSignal / WinterTC additions

Source: https://expo.dev/changelog/sdk-56 and CHANGELOG (v56.0.0)

> "`AbortSignal.timeout` / `AbortSignal.any` support for WinterTC-compatible fetch behavior."

> "Added `AbortSignal.timeout`, `AbortSignal.any`, and `DOMException` to the native runtime."

Example usage pattern:

```ts
// time out the request after 5 seconds
const resp = await fetch(url, { signal: AbortSignal.timeout(5000) });

// combine multiple abort sources
const resp2 = await fetch(url, {
  signal: AbortSignal.any([userAbort.signal, AbortSignal.timeout(10000)]),
});
```

### Other SDK 56 fetch fixes (from `origin/sdk-56:packages/expo/CHANGELOG.md`)

Up to 56.0.0:

- "Implement `Response.clone()` on `expo/fetch`, and throw the spec's `TypeError` when a body is read twice." (preview.12, #45740)
- "Fix `expo/fetch` not threading through `Request#body` for `whatwg-fetch` request inputs." (v56.0.0, #46027)
- "Fix `expo/fetch` not respecting its own `NativeRequest` as `RequestInit` inputs." (preview.13, #45958)
- "Accept `credentials: 'same-origin'` in `expo/fetch` mirroring `include`." (preview.13, #45958)

After 56.0.5 — this is SDK 56 patch drift, and the reason to stay current on the `expo` patch line.
Every row is also on the SDK 57 line, so none of them is an upgrade reason; the required minimum
`expo` version is in the left column (see the delta section for the 57-line equivalents):

| Version | Fix |
| --- | --- |
| 56.0.8 | [Android] Decompress `gzip`/`br`/`zstd` even when the caller sets their own `Accept-Encoding` header (#46398) |
| 56.0.8 | `bodyUsed` no longer leaks across siblings when a `Response` is cloned twice (#46397) |
| 56.0.8 | No more fatal `The stream is not in a state that permits close` when native delivers `didComplete`/`didFailWithError` after the consumer cancelled the body stream (#44909) |
| 56.0.10 | [Android] Body-less `POST`/`PUT`/`PATCH` no longer send a single `0x00` byte instead of an empty body (#46678) |
| 56.0.16 | `Response.blob()` no longer throws when the global `Blob` is react-native's implementation (#47538) |
| 56.0.17 | [Android] Fixed race condition causing out-of-order delivery of initial chunks (#42161) |

### fetch() type reference

Sources: docs + expo source (`packages/expo/src/winter/fetch/`).

```ts
// FetchRequestInit — a fetch RequestInit-compatible structure
interface FetchRequestInit {
  body?: BodyInit | null;
  credentials?: RequestCredentials;   // 'same-origin' is normalized to 'include' — see note below
  headers?: HeadersInit;
  method?: string;
  signal?: AbortSignal | null;        // supports AbortSignal.timeout / AbortSignal.any
  redirect?: RequestRedirect;
  integrity?: string;
  keepalive?: boolean;
  mode?: RequestMode;
  referrer?: string;
  window?: any;
}
```

> `credentials: 'same-origin'` **is** accepted: `packages/expo/src/winter/fetch/fetch.ts:48-51`
> does `if (credentials === 'same-origin') credentials = 'include'`. The inline comment in
> `fetch.types.ts` that says "same-origin is not supported" is stale upstream — trust the
> implementation, not the type comment.

```ts
// FetchResponse
class FetchResponse {
  // Properties
  body: ReadableStream<Uint8Array<ArrayBuffer>> | null;
  bodyUsed: boolean;
  headers: Headers;
  ok: boolean;
  redirected: boolean;
  status: number;
  statusText: string;
  type: 'default';
  url: string;

  // Methods
  json(): Promise<any>;
  text(): Promise<string>;
  bytes(): Promise<Uint8Array<ArrayBuffer>>;
  blob(): Promise<Blob>;
  arrayBuffer(): Promise<ArrayBuffer>;
  formData(): Promise<UniversalFormData>;
  clone(): FetchResponse;             // throws TypeError if body already read twice
}
```

### Streaming example (verbatim from docs)

```ts
import { fetch } from 'expo/fetch';

const resp = await fetch('https://httpbin.org/drip?numbytes=512&duration=2', {
  headers: { Accept: 'text/event-stream' },
});
const reader = resp.body.getReader();
const chunks = [];
while (true) {
  const { done, value } = await reader.read();
  if (done) {
    break;
  }
  chunks.push(value);
}
const buffer = new Uint8Array(chunks.reduce((acc, chunk) => acc + chunk.length, 0));
console.log(buffer.length); // 512
```

`value` is a `Uint8Array`; `resp.body` is `ReadableStream<Uint8Array> | null`, so guard it in
strict TypeScript.

---

## 7. Migration Notes (SDK 55 → 56)

- Replace synchronous `copy()` / `move()` calls with `await file.copy(...)` / `await file.move(...)`,
  or switch to `copySync()` / `moveSync()` if you depend on synchronous behavior.
- Prefer `File.createDownloadTask()` / `file.createUploadTask()` (or `File.upload()`) for
  long-running transfers needing progress, cancellation, or resumable downloads.
- Pass `overwrite: true` (`RelocationOptions`) where you previously relied on implicit overwrite.
- Repoint every legacy `FileSystem.*` call at `expo-file-system/legacy` (or migrate to
  `File`/`Directory`) — the root re-exports throw at runtime. See §2 "Legacy API".
- If a dependency breaks on the new global fetch (e.g. WinterCG-strict body/credential handling),
  set `EXPO_PUBLIC_USE_RN_FETCH=1` (or `=true`) in `.env` to restore React Native's fetch globally;
  named `expo/fetch` imports still use the new implementation. Ensure `babel-preset-expo >= 56.0.16`
  or the flag will be ignored inside `node_modules` in production builds.
- Use `AbortSignal.timeout()` / `AbortSignal.any()` for request timeouts and combined cancellation.

> There is no `https://docs.expo.dev/guides/network/` page in any SDK version. Networking guidance
> lives in the `expo/fetch` section of `/versions/v56.0.0/sdk/expo/` (and `/v57.0.0/sdk/expo/`);
> `expo-network` (device connectivity state) is a different package — see reference 15.

---

## SDK 57 delta

**Nothing in this domain changed in SDK 57.** `expo-file-system` and `expo/fetch` are API-identical
between SDK 56 and SDK 57; only the package pin moves. Everything above applies unchanged to 57.

Verification (monorepo, `origin/sdk-56` = 56.x line, `origin/sdk-57` = 57.x line):

- `git diff origin/sdk-56..origin/sdk-57 -- packages/expo-file-system/` touches exactly three
  files: `CHANGELOG.md`, `android/build.gradle` (version string), `package.json` (version field).
  Every file under `src/` — `File.ts`, `Directory.ts`, `Paths.ts`, `NetworkTasks.ts`,
  `internal/NativeFileSystem.types.ts`, `*.types.ts`, `index.ts`, `legacyWarnings.ts`,
  `legacy/` — is byte-identical.
- `git diff origin/sdk-56..origin/sdk-57 -- packages/expo/src/winter` is **empty**. The whole
  WinterCG surface (`fetch/fetch.ts`, `FetchResponse.ts`, `fetch.types.ts`, `runtime.native.ts`,
  `AbortSignal.ts`, `DOMException.ts`) is identical between `expo` 56.0.17 and 57.0.8.
- `expo-file-system` 57.0.0 (2026-06-25) and 57.0.1 (2026-07-15) both read
  _"This version does not introduce any user-facing changes."_ Same for `expo` 57.0.0 (2026-06-30).
- `docs/pages/versions/v56.0.0/sdk/filesystem.mdx` vs `v57.0.0/sdk/filesystem.mdx` differ only on
  the `sourceCodeUrl` frontmatter line. Same for `sdk/expo.mdx`. The config-plugin props
  (`supportsOpeningDocumentsInPlace`, `enableFileSharing`) are unchanged.

### Version pins (56 -> 57)

Source: `git show origin/sdk-56:packages/expo/bundledNativeModules.json` and the `origin/sdk-57`
copy. That file ships inside the `expo` package and is what `expo install` resolves against.
Do **not** use `docs/public/static/schemas/v*/native-modules.json` — those are frozen at the beta
cut and understate several pins.

| Package | SDK 56 | SDK 57 |
| --- | --- | --- |
| `expo-file-system` | `~56.0.8` | `~57.0.1` |

`expo` itself (which ships `expo/fetch`) is `56.0.17` on the SDK 56 line and `57.0.8` on the SDK 57
line. Note the first-party pin is **not** a flat `~57.0.0` — `expo install` gives you `~57.0.1`.
Exactly seven third-party pins moved 56 -> 57 (`react-native` 0.85.3 -> 0.86.0 is the only one that
touches networking, via its own `fetch`/networking stack); none of the rest is relevant here, and
`react-native-screens` did **not** move — it is `~4.26.0` on both lines. See reference 01.

### Not in 57 — do not write this code

These land on `main` but are absent from `origin/sdk-57`. They are SDK 58 (unversioned) material:

- `File.preview()` / `File.canPreview()` (#46867, merged 2026-06-24). `grep preview` on
  `origin/sdk-57:packages/expo-file-system/src/File.ts` returns nothing.
- `File.write()` becoming async with a new `File.writeSync()` (#45992). In 56 **and** 57,
  `write(content, options?)` is **synchronous** and returns `void`.
- `FileHandle.readBytes()` / `writeBytes()` becoming async with `readBytesSync()` /
  `writeBytesSync()` (#46280). In 56 **and** 57 both are **synchronous**.

### Backported, not a 57 delta

Every `expo/fetch` fix people cite as an SDK 57 improvement is also on the SDK 56 line, so the fix
is one patch bump away — not an upgrade reason. Checked in both
`origin/sdk-56:packages/expo/CHANGELOG.md` and `origin/sdk-57:packages/expo/CHANGELOG.md`:

| Fix | On the SDK 56 line | On the SDK 57 line |
| --- | --- | --- |
| Android `Accept-Encoding` decompression (#46398) | `expo` 56.0.8 | pre-dates the 57 cut (same 56.0.8 entry) |
| `bodyUsed` leak across clones (#46397) | `expo` 56.0.8 | pre-dates the 57 cut |
| `stream is not in a state that permits close` (#44909) | `expo` 56.0.8 | pre-dates the 57 cut |
| Android body-less `POST`/`PUT`/`PATCH` `0x00` byte (#46678) | `expo` 56.0.10 | pre-dates the 57 cut |
| `Response.blob()` vs react-native `Blob` (#47538) | `expo` 56.0.16 | `expo` 57.0.5 |
| Android out-of-order initial chunks (#42161) | `expo` 56.0.17 | `expo` 57.0.8 |

Same story for the `EXPO_PUBLIC_USE_RN_FETCH` inlining fix (#46986): `babel-preset-expo` 56.0.16
**and** 57.0.0. Staying current on the 56 patch line gets you all of it.

`expo` 57.0.0 itself reads _"This version does not introduce any user-facing changes."_
