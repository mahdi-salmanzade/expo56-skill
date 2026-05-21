# Expo SDK 56 — File System & Networking (expo-file-system, expo/fetch)

> Knowledge-base reference for Expo SDK 56. Captures the new task-based transfer APIs,
> the object-oriented File/Directory/Paths API, the async copy/move breaking change,
> multi-file picking, experimental file-system watching, and the `expo/fetch` changes
> (decompression, AbortSignal improvements, and `expo/fetch` becoming the default
> `globalThis.fetch`).

**Primary sources**

- Changelog: https://expo.dev/changelog/sdk-56
- Beta changelog: https://expo.dev/changelog/sdk-56-beta
- FileSystem API: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/
- expo/fetch API: https://docs.expo.dev/versions/v56.0.0/sdk/expo/ (and `/versions/latest/sdk/expo/`)
- expo package CHANGELOG: https://github.com/expo/expo/blob/main/packages/expo/CHANGELOG.md
- Network guide `https://docs.expo.dev/guides/network/` — **does not exist (HTTP 404)**

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
| `creationTime` | `number \| null` | Android API 26+ |
| `md5` | `string \| null` | deprecated |
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
static downloadFileAsync(url: string, destination: File | Directory): Promise<File>
createDownloadTask(url: string, destination: File | Directory, options?: DownloadTaskOptions): DownloadTask
createUploadTask(url: string, options?: UploadOptions): UploadTask
upload(url: string, options?: UploadOptions): Promise<UploadResult>
```

Note: `createDownloadTask` is shown as a `static` method on `File`; `createUploadTask`
and `upload` are instance methods (the instance is the file being uploaded).

**File picking** (multi-file capable in SDK 56)

```ts
static pickFileAsync(options?: PickSingleFileOptions): Promise<PickSingleFileResult>
static pickFileAsync(options: PickMultipleFilesOptions): Promise<PickMultipleFilesResult>   // overload
static pickFileAsync(initialUri?: string, mimeType?: string): Promise<File>                 // deprecated overload
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
```

### Class: `Paths`

```ts
import { Paths } from 'expo-file-system';
```

**Properties**

```ts
cache: Directory                                  // system cache, may be deleted by OS
document: Directory                               // persistent storage
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
readBytes(length: number): Uint8Array
writeBytes(bytes: Uint8Array): void
close(): void
```

---

## 3. Task-Based Transfer APIs

Source: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/

### Class: `DownloadTask`

Supports progress, pause/resume, cancellation, and resumable downloads.

```ts
// Property
state: 'idle' | 'active' | 'paused' | 'completed' | 'cancelled' | 'error'

// Methods
downloadAsync(): Promise<File>
pause(): void
pauseAsync(): Promise<void>
resume(): Promise<void>                 // after pause
resumeAsync(): Promise<File>
cancel(): void
savable(): DownloadPauseState           // persist pause state across restarts
static fromSavable(state: DownloadPauseState, options?: DownloadTaskOptions): DownloadTask
addListener(eventName: 'progress', listener: (data: DownloadProgress) => void): EventSubscription
removeListener(eventName: 'progress', listener: (data: DownloadProgress) => void): void
removeAllListeners(eventName: 'progress'): void
listenerCount(eventName: string): number
release(): void
```

### Class: `UploadTask`

```ts
// Property
state: UploadTaskState   // 'idle' | 'active' | 'completed' | 'cancelled' | 'error'

// Methods
uploadAsync(): Promise<UploadResult>
cancel(): void
addListener(eventName: 'progress', listener: (data: UploadProgress) => void): EventSubscription
removeListener(eventName: 'progress', listener: (data: UploadProgress) => void): void
removeAllListeners(eventName: 'progress'): void
listenerCount(eventName: string): number
release(): void
```

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
  encoding?: 'utf8' | 'base64';  // Default: 'utf8'
  append?: boolean;               // Default: false
}

// RelocationOptions  (used by copy/copySync/move/moveSync)
{
  overwrite?: boolean;  // Default: false
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
FileSystem.DEFAULT_DEBOUNCE_MS = 100   // ms used for watch debouncing
```

---

## 5. FileSystem Code Examples (verbatim)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/filesystem/

**Download with progress**

```ts
const dest = new File(Paths.document, 'video.mp4');
const downloadTask = File.createDownloadTask(url, dest, {
  onProgress: ({ bytesWritten, totalBytes }) => {
    console.log(`${bytesWritten}/${totalBytes} bytes`);
  }
});
const file = await downloadTask.downloadAsync();
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
function printDirectory(directory: Directory, indent: number = 0) {
  console.log(`${' '.repeat(indent)} + ${directory.name}`);
  for (const item of directory.list()) {
    if (item instanceof Directory) {
      printDirectory(item, indent + 2);
    } else {
      console.log(`${' '.repeat(indent + 2)} - ${item.name}`);
    }
  }
}
```

---

## 6. expo/fetch — Networking (SDK 56)

Sources:
https://docs.expo.dev/versions/v56.0.0/sdk/expo/ ,
https://expo.dev/changelog/sdk-56 ,
https://github.com/expo/expo/blob/main/packages/expo/CHANGELOG.md

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
- Setting `EXPO_PUBLIC_USE_RN_FETCH=1` restores React Native's built-in `fetch` as the global.
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

### Other SDK 56 fetch fixes (from CHANGELOG)

- "Implement `Response.clone()` on `expo/fetch`, and throw the spec's `TypeError` when a body is read twice." (preview.12)
- "Fix `expo/fetch` not threading through `Request#body` for `whatwg-fetch` request inputs." (v56.0.0)
- "Fix `expo/fetch` not respecting its own `NativeRequest` as `RequestInit` inputs." (preview.13)
- "Accept `credentials: 'same-origin'` in `expo/fetch` mirroring `include`." (preview.13)

### fetch() type reference

Sources: docs + expo source (`packages/expo/src/winter/fetch/`).

```ts
// FetchRequestInit — a fetch RequestInit-compatible structure
interface FetchRequestInit {
  body?: BodyInit | null;
  credentials?: RequestCredentials;   // 'same-origin' mirrors 'include' in SDK 56
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
  const { done, value } = await reader.read();   // value: Uint8Array
  if (done) {
    break;
  }
  chunks.push(value);
}
```

---

## 7. Migration Notes (SDK 55 → 56)

- Replace synchronous `copy()` / `move()` calls with `await file.copy(...)` / `await file.move(...)`,
  or switch to `copySync()` / `moveSync()` if you depend on synchronous behavior.
- Prefer `File.createDownloadTask()` / `file.createUploadTask()` (or `File.upload()`) for
  long-running transfers needing progress, cancellation, or resumable downloads.
- Pass `overwrite: true` (`RelocationOptions`) where you previously relied on implicit overwrite.
- If a dependency breaks on the new global fetch (e.g. WinterCG-strict body/credential handling),
  set `EXPO_PUBLIC_USE_RN_FETCH=1` in `.env` to restore React Native's fetch globally; named
  `expo/fetch` imports still use the new implementation.
- Use `AbortSignal.timeout()` / `AbortSignal.any()` for request timeouts and combined cancellation.

---

## Pages that could not be accessed

- `https://docs.expo.dev/guides/network/` — returned **HTTP 404 Not Found** (page does not exist
  in the SDK 56 docs). Networking guidance is consolidated in the `expo/fetch` section of
  https://docs.expo.dev/versions/v56.0.0/sdk/expo/ .
