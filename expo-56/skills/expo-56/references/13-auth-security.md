# Expo SDK 56 — Authentication & Security Reference

Knowledge base reference covering authentication and security packages in Expo SDK 56. Each section links its official source. Doc paths use the pinned SDK 56 version (`v56.0.0`).

> Security principle (applies across all sections): **Never store secret keys (client secrets) in application code — there is no secure way to do this.** Exchange authorization codes and hold secrets on a backend server.

---

## Table of Contents
1. [expo-auth-session](#1-expo-auth-session)
2. [AuthSession guides — providers & redirect URIs](#2-authsession-guides--providers--redirect-uris)
3. [expo-secure-store](#3-expo-secure-store)
4. [expo-local-authentication](#4-expo-local-authentication)
5. [expo-crypto](#5-expo-crypto)
6. [expo-apple-authentication](#6-expo-apple-authentication)
7. [expo-web-browser](#7-expo-web-browser)
8. [Expo Router authentication pattern](#8-expo-router-authentication-pattern)
9. [SDK 56 notes summary](#9-sdk-56-notes-summary)
10. [SDK 57 delta](#sdk-57-delta)

---

## Gotchas — read first

Things a model reliably gets wrong from memory in this domain:

- **`TokenResponse.refreshAsync` is an instance method, not static.** `await token.refreshAsync({ clientId }, discovery)` — its config type *excludes* `refreshToken` (it reads `this.refreshToken`). The static/module-level `AuthSession.refreshAsync(config, discovery)` is the one that takes `refreshToken`. (`packages/expo-auth-session/src/TokenRequest.ts:124`)
- **`AuthSessionResult` is not narrowable between `success` and `error`.** They share one object shape, and the no-payload branch includes `'opened'` and `'locked'` besides `'cancel' | 'dismiss'`. See [section 1](#result-type--authsessionresult).
- **Expo Go cannot complete an OAuth redirect.** The module loads there, but the scheme is fixed, so use a development build. (`docs/pages/guides/authentication.mdx:19`)
- **`expo-auth-session/providers/Google` and `/providers/Facebook` still exist but are deprecated** — use `@react-native-google-signin/google-signin` / `react-native-fbsdk-next`.
- **`CodeChallengeMethod.Plain` throws.** `new AuthRequest(...)` asserts against it; `S256` is the only usable method.
- **`SecurityLevel.BIOMETRIC` is a deprecated alias** that logs a warning and resolves to `BIOMETRIC_WEAK` on Android but `BIOMETRIC_STRONG` on iOS. Use the explicit members.
- **`expo-crypto` ships a full AES-GCM API** (`aesEncryptAsync` / `aesDecryptAsync` / `AESEncryptionKey` / `AESSealedData`), not just hashing — and its string inputs must be **base64**. See [section 5](#aes-gcm-encryption).
- **`TokenResponse.isTokenFresh`'s `secondsMargin` defaults to `-600`**, so a token is reported stale ten minutes before its real expiry.

---

## 1. expo-auth-session

Source: https://docs.expo.dev/versions/v56.0.0/sdk/auth-session/

Browser-based OAuth 2.0 and OpenID Connect flows built on top of `WebBrowser` and `Crypto`. Platforms (docs frontmatter): Android, iOS, Web, Expo Go — but the module merely *loads* in Expo Go; you cannot complete an OAuth redirect there because the app scheme cannot be customised. Use a development build for any real flow (`docs/pages/guides/authentication.mdx:19`).

### Installation
```sh
npx expo install expo-auth-session expo-crypto
```
`expo-crypto` is a required peer dependency.

### Configuration (deep linking)
```sh
npx uri-scheme add mycoolredirect
npx uri-scheme list
npx uri-scheme open mycoolredirect://some/redirect
```
```json
{
  "expo": {
    "scheme": "mycoolredirect"
  }
}
```
Without a configured scheme, authentication completes but cannot redirect back to the app.

### Hooks

#### `useAuthRequest(config, discovery)`
Returns `[request | null, response | null, promptAsync]`. `config` is an `AuthRequestConfig`; `discovery` is a `DiscoveryDocument | null` (only `authorizationEndpoint` is required).
```ts
const [request, response, promptAsync] = useAuthRequest(
  {
    clientId: 'YOUR_CLIENT_ID',
    redirectUri: makeRedirectUri(),
    scopes: ['openid', 'profile'],
  },
  discovery
);

const handleLogin = async () => {
  const result = await promptAsync();
};
```

#### `useAuthRequestResult(request, discovery, customOptions?)`
Executes an auth request. Returns `[AuthSessionResult | null, PromptMethod]`.

#### `useAutoDiscovery(issuerOrDiscovery)`
Fetches an OpenID Connect discovery document from an issuer URL (`https` scheme, no query/fragment). Returns `DiscoveryDocument | null` (null until fetched).
```ts
const discovery = useAutoDiscovery('https://accounts.google.com');
```

#### `useLoadedAuthRequest(config, discovery, AuthRequestInstance)`
Loads a pre-configured request. Returns `AuthRequest | null`.

### Classes

#### `AuthRequest`
Manages the OAuth authorization request lifecycle (RFC 6749 §4.1.1).
- Properties: `codeVerifier?` (PKCE), `state` (CSRF token), `url` (`string | null`).
- Methods:
  - `getAuthRequestConfigAsync(): Promise<AuthRequestConfig>`
  - `makeAuthUrlAsync(discovery): Promise<string>`
  - `parseReturnUrl(url): AuthSessionResult`
  - `promptAsync(discovery, promptOptions?): Promise<AuthSessionResult>`
```ts
const request = new AuthRequest({
  clientId: 'YOUR_CLIENT_ID',
  redirectUri: makeRedirectUri(),
  scopes: ['openid', 'profile'],
});

const result = await request.promptAsync(discovery);
```

#### `AccessTokenRequest` (extends `TokenRequest`)
Exchanges authorization code for access token (RFC 6749 §4.1.3). `grantType: GrantType`. Methods: `getHeaders()`, `getQueryBody()`, `getRequestConfig()`, `performAsync(discovery): Promise<TokenResponse>`.

#### `RefreshTokenRequest` (extends `TokenRequest`)
Refreshes access tokens (RFC 6749 §6). Same method surface as above; `performAsync` returns `Promise<TokenResponse>`.

#### `RevokeTokenRequest` (extends `Request`)
Revokes tokens (RFC 7009 §2.1). `performAsync(discovery): Promise<boolean>` — requires `revocationEndpoint`.

#### `TokenResponse`
OAuth token response (RFC 6749 §5.1).
- **Instance properties**: `accessToken: string`, `tokenType: TokenType` (**required**, defaults to `'bearer'` in the constructor), `issuedAt: number` (**required**, defaults to now), `expiresIn?`, `refreshToken?`, `idToken?`, `scope?`, `state?`, `rawResponse?`. Only the *input* type `TokenResponseConfig` makes `tokenType` and `issuedAt` optional.
- **Static methods**:
  - `TokenResponse.isTokenFresh(token: Pick<TokenResponse, 'expiresIn' | 'issuedAt'>, secondsMargin: number = 60 * 10 * -1): boolean` — the default margin is **−600 seconds**, so a token is reported stale ten minutes *before* it actually expires. Returns `true` for any token with no `expiresIn` (assumed never to expire), and `false` for a falsy token.
  - `TokenResponse.fromQueryParams(params): TokenResponse` — used for implicit grant.
- **Instance methods**:
  - `getRequestConfig(): TokenResponseConfig`
  - `shouldRefresh(): boolean` — `true` only when a `refreshToken` exists *and* the token is not fresh.
  - `refreshAsync(config: Omit<TokenRequestConfig, 'grantType' | 'refreshToken'>, discovery: Pick<DiscoveryDocument, 'tokenEndpoint'>): Promise<TokenResponse>` — **instance method**. You do not pass `refreshToken`; it reads `this.refreshToken`. It mutates and returns `this`, and reuses the old refresh token when the server omits one.
```ts
// Instance method — no `refreshToken` in the config, and `refreshed === oldToken`.
const refreshed = await oldToken.refreshAsync({ clientId: 'YOUR_CLIENT_ID' }, discovery);
```
> Do **not** write `TokenResponse.refreshAsync({ clientId, refreshToken }, discovery)` — that overload does not exist. For a standalone refresh use the module-level `AuthSession.refreshAsync(config, discovery)` in the table below, which *does* take `refreshToken`.

#### Error classes
- `AuthError` (extends `ResponseError`): `code`, `description?`, `info?`, `params`, `state?`, `uri?` (RFC 6749 §5.2).
- `TokenError` (extends `ResponseError`): `code`, `description?`, `info?`, `params`, `uri?` (RFC 6749 §4.1.2.1).
- `ResponseError` (extends `CodedError`): base — `code`, `description?`, `info?`, `params`, `uri?`.

#### Base request classes
- `Request`: `getQueryBody()`, `getRequestConfig()`, `performAsync(discovery)`.
- `TokenRequest` (extends `Request`): adds `grantType`, `getHeaders()`.

### Static methods (`AuthSession.*`)

| Method | Signature / Returns | Notes |
|--------|--------------------|-------|
| `dismiss()` | `void` | Cancels active session. |
| `exchangeCodeAsync(config, discovery)` | `Promise<TokenResponse>` | `config: AccessTokenRequestConfig`; discovery needs `tokenEndpoint`. |
| `fetchDiscoveryAsync(issuer)` | `Promise<DiscoveryDocument>` | Fetch OpenID discovery doc. |
| `fetchUserInfoAsync(config, discovery)` | `Promise<Record<string, any>>` | `config` has `accessToken`; discovery needs `userInfoEndpoint`. |
| `getCurrentTimeInSeconds()` | `number` | Unix timestamp. |
| `getDefaultReturnUrl(urlPath?, options?)` | `string` | **Deprecated** — use `makeRedirectUri()`. |
| `getRedirectUrl(path?)` | `string` | Throws in bare workflow without config. |
| `issuerWithWellKnownUrl(issuer)` | `string` | Appends `.well-known/openid-configuration`. |
| `loadAsync(config, issuerOrDiscovery)` | `Promise<AuthRequest>` | Builds + loads request. |
| `makeRedirectUri(options?)` | `string` | Platform-aware redirect URI. |
| `refreshAsync(config, discovery)` | `Promise<TokenResponse>` | `config: RefreshTokenRequestConfig`. |
| `resolveDiscoveryAsync(issuerOrDiscovery)` | `Promise<DiscoveryDocument>` | |
| `revokeAsync(config, discovery)` | `Promise<boolean>` | `config: RevokeTokenRequestConfig`; needs `revocationEndpoint`. |
| `requestAsync(requestUrl, fetchRequest)` | `Promise<T>` | Generic request executor. |

```ts
// exchangeCodeAsync
const token = await AuthSession.exchangeCodeAsync(
  {
    clientId: 'YOUR_CLIENT_ID',
    clientSecret: 'YOUR_CLIENT_SECRET',
    code: authCode,
    redirectUri: makeRedirectUri(),
  },
  discovery
);

// fetchDiscoveryAsync
const discovery = await AuthSession.fetchDiscoveryAsync('https://accounts.google.com');

// fetchUserInfoAsync
const userInfo = await AuthSession.fetchUserInfoAsync(
  { accessToken: token.accessToken },
  discovery
);

// refreshAsync
const refreshed = await AuthSession.refreshAsync(
  { clientId: 'YOUR_CLIENT_ID', refreshToken: token.refreshToken },
  discovery
);

// revokeAsync
const revoked = await AuthSession.revokeAsync(
  { clientId: 'YOUR_CLIENT_ID', token: token.refreshToken },
  discovery
);
```

#### `makeRedirectUri` behavior & options
Platform behavior: Web → current window location; Managed → app config `scheme`; Bare → falls back to `native` option.
```ts
// Development Build: my-scheme://redirect
// Expo Go: exp://127.0.0.1:8081/--/redirect
// Web dev: https://localhost:19006/redirect   <- stale upstream JSDoc, see note
// Web prod: https://yourwebsite.com/redirect
const uri = AuthSession.makeRedirectUri({ scheme: 'my-scheme', path: 'redirect' });

const uri2 = AuthSession.makeRedirectUri({
  scheme: 'scheme2',
  preferLocalhost: true,
  isTripleSlashed: true,
});
// Development Build: scheme2:///
// Expo Go: exp://localhost:8081
```
`AuthSessionRedirectUriOptions`: `scheme?`, `path?`, `preferLocalhost?` (iOS simulator only, default `false`), `isTripleSlashed?` (default `false`), `native?` (manual scheme for **Bare and Standalone** native app contexts — takes precedence over all other properties), `queryParams?`.

> The `19006` in the examples above is copied verbatim from the upstream JSDoc and is a stale Webpack-era artifact. Metro web dev serves on `8081`, and on web `makeRedirectUri` derives the URL from `window.location`, so it reflects whatever port you are actually on. `native` is only honoured when `Constants.executionEnvironment` is `Standalone` or `Bare`.

### Configuration types

**`AuthRequestConfig`**: `clientId` (req), `clientSecret?`, `redirectUri` (req), `scopes?`, `state?` (default: a random 10-char string generated by `PKCE.generateRandom(10)`), `responseType?` (default `ResponseType.Code`), `prompt?`, `codeChallenge?`, `codeChallengeMethod?` (default `S256`), `usePKCE?` (default `true`), `extraParams?`.

**`AccessTokenRequestConfig`** (extends `TokenRequestConfig`): + `code` (req), `redirectUri` (req).

**`RefreshTokenRequestConfig`** (extends `TokenRequestConfig`): + `refreshToken?`.

**`RevokeTokenRequestConfig`** (extends `Partial<TokenRequestConfig>`): + `token` (req), `tokenTypeHint?`.

**`TokenRequestConfig`** (base): `clientId` (req), `clientSecret?`, `scopes?`, `extraParams?`, `extraHeaders?`.

**`TokenResponseConfig`**: `accessToken` (req), `expiresIn?`, `refreshToken?`, `idToken?`, `tokenType?`, `scope?`, `state?`, `issuedAt?`.

**`ServerTokenResponseConfig`** (provider snake_case form): `access_token` (req), `expires_in?`, `refresh_token?`, `id_token?`, `token_type?`, `scope?`, `issued_at?`.

**`AuthRequestPromptOptions`** (extends `Omit<AuthSessionOpenOptions, 'windowFeatures'>`): `url?`, `windowFeatures?` (web only).

**`DiscoveryDocument`**: `authorizationEndpoint?`, `tokenEndpoint?`, `revocationEndpoint?`, `userInfoEndpoint?`, `endSessionEndpoint?`, `registrationEndpoint?`, `discoveryDocument?` (`ProviderMetadata`). `AuthDiscoveryDocument` = `Pick<DiscoveryDocument, 'authorizationEndpoint'>`. `ProviderMetadata` carries the full OpenID provider metadata (`authorization_endpoint`, `token_endpoint`, `jwks_uri`, `scopes_supported`, `code_challenge_methods_supported`, etc.).

### Result type — `AuthSessionResult`
Two-member union — **`success` and `error` share one object shape**, so TypeScript cannot narrow between them (`packages/expo-auth-session/src/AuthSession.types.ts:12-43`):
```ts
type AuthSessionResult =
  | { type: 'cancel' | 'dismiss' | 'opened' | 'locked' }
  | {
      type: 'error' | 'success';
      /** @deprecated Legacy error code query param — use `error` instead. */
      errorCode: string | null;
      error?: AuthError | null;
      params: Record<string, string>;
      authentication: TokenResponse | null;
      url: string;
    };
```
- Narrowing on `type === 'success'` still exposes `errorCode` / `error`; narrowing on `'error'` still exposes `authentication`. Check `type === 'success'` **and** inspect `params` / `error` yourself.
- The no-payload branch has four members, not two — `'opened'` and `'locked'` are commonly missed.
- `errorCode` is deprecated; read `error` (an `AuthError`).

### Enums
- **`ResponseType`**: `Code = "code"`, `Token = "token"`, `IdToken = "id_token"`.
- **`CodeChallengeMethod`**: `Plain = "plain"` — the enum value exists, but `new AuthRequest(...)` throws an invariant if you pass it (`` `AuthRequest` does not support `CodeChallengeMethod.Plain` as it's not secure. ``); `S256 = "S256"` is the only usable method. The constructor also throws when `redirectUri` is missing/empty, with a platform-specific example (`https://yourwebsite.com/` on web, `com.your.app:/oauthredirect` elsewhere). (`packages/expo-auth-session/src/AuthRequest.ts:95-105`)
- **`GrantType`**: `AuthorizationCode`, `Implicit`, `RefreshToken`, `ClientCredentials`.
- **`Prompt`**: `None`, `Login`, `Consent`, `SelectAccount` (can pass an array).
- **Type aliases**: `IssuerOrDiscovery = Issuer | DiscoveryDocument`; `Issuer = string`; `TokenType = 'bearer' | 'mac'`; `TokenTypeHint` (`AccessToken | RefreshToken`); `PromptMethod(options?) => Promise<AuthSessionResult>`.

### Deprecated provider subpath modules
These still ship and still work, but their config types are marked `@deprecated` and the docs point elsewhere. Named here so they are recognised rather than emitted from memory:
```ts
import * as Google from 'expo-auth-session/providers/Google';
import * as Facebook from 'expo-auth-session/providers/Facebook';
```
| Module | Exports | Deprecation |
|--------|---------|-------------|
| `expo-auth-session/providers/Google` | `discovery`, `useAuthRequest(config?, redirectUriOptions?)`, `useIdTokenAuthRequest(config, redirectUriOptions?)`, type `GoogleAuthRequestConfig` (`clientId`, `webClientId`, `iosClientId`, `androidClientId`, …) | `GoogleAuthRequestConfig` carries `@deprecated See [Google authentication](/guides/google-authentication/)`. Replacement: `@react-native-google-signin/google-signin`. |
| `expo-auth-session/providers/Facebook` | `discovery`, `useAuthRequest(config?, redirectUriOptions?)`, type `FacebookAuthRequestConfig` | `FacebookAuthRequestConfig` carries `@deprecated See [Facebook authentication](/guides/facebook-authentication/)`. Replacement: `react-native-fbsdk-next`. |

Note the hook functions themselves are not annotated `@deprecated` — only the config types — so TypeScript will not always flag usage. The package root (`expo-auth-session`) re-exports only the two config *types*, not the hooks; the hooks must be imported from the subpath.

### Advanced notes
- **Web popup close**: call `WebBrowser.maybeCompleteAuthSession()` to close the popup after auth.
- **Implicit grant (legacy only)**: `ResponseType.Token` is still supported "for legacy code purposes" but the guide states it is **no longer recommended** because of access-token-injection risk — use authorization code + PKCE. If you must, pass `response.params` to `TokenResponse.fromQueryParams()`.
- **Filtering linking events**: AuthSession appends `+expo-auth-session` to the default redirect URL — filter `Linking` events on this string. With React Navigation, use a custom `getStateFromPath`.
- **Provider libraries recommended** instead of raw AuthSession where available: Google → `@react-native-google-signin/google-signin`; Facebook → `react-native-fbsdk-next`.

---

## 2. AuthSession guides — providers & redirect URIs

Sources:
- https://docs.expo.dev/guides/authentication/
- https://docs.expo.dev/guides/google-authentication/

### General AuthSession rules (from the Authentication guide)
- Dismiss popups with `WebBrowser.maybeCompleteAuthSession()`.
- Create redirects with `AuthSession.makeRedirectUri()` (`scheme`, `path`, `native` parameters; e.g. `your.app://redirect`).
- Build requests with `AuthSession.useAuthRequest()`; keep the prompt disabled until `request` is defined.
- On web you can only invoke `promptAsync` from a user interaction.
- Client secrets must be exchanged on a server: "Since your client application code is not a secure place to store secrets, it is necessary to exchange the authorization code in a server." (API routes or React Server Components are the suggested places.)
- **Expo Go cannot be used** for local development/testing of OAuth or OpenID Connect apps, "due to the inability to customize your app scheme" — use a development build.
- The Authentication guide ships complete worked examples for **GitHub** and **Okta**. (Google/Apple are no longer documented as raw AuthSession recipes here — see below.)

### UX & security patterns from the guide

**Warming the browser (Android).** Pre-initialise the Custom Tabs browser in the background to significantly speed up the auth prompt:
```tsx
import { useEffect } from 'react';
import * as WebBrowser from 'expo-web-browser';

function App() {
  useEffect(() => {
    // Android only: start loading the default browser app in the background.
    WebBrowser.warmUpAsync();
    // Android only: cool down on unmount to help memory on low-end devices.
    return () => {
      WebBrowser.coolDownAsync();
    };
  }, []);
}
```

**Implicit login is legacy.** The guide frames `ResponseType.Token` as "no longer recommended … due to inherent security risks, including the risk of access token injection"; authorization code + PKCE supersedes it. `expo-auth-session` keeps Implicit support only for existing code.

**Store tokens with `expo-secure-store`** (see [section 3](#3-expo-secure-store)), never in plain `AsyncStorage`.

### Google
The SDK 56 Google guide now recommends `@react-native-google-signin/google-signin` rather than a raw `expo-auth-session` recipe. Key points:
- Requires a **development build** (not Expo Go).
- Configure a Firebase or Google Cloud Console project.
- Provide SHA-1 certificate fingerprints (from your `.apk` build / Play Console App Integrity, and from the production app-signing key).
- Firebase approach uploads `google-services.json` + `GoogleService-Info.plist` to EAS; a Google Cloud Console approach exists as an alternative without Firebase.
- Detailed client-ID / scope / code samples live in the external `@react-native-google-signin/google-signin` library docs (linked from the guide).

### Apple
For native Sign In with Apple, use [`expo-apple-authentication`](#6-expo-apple-authentication) (section 6) rather than AuthSession. AuthSession can be used for Apple's web OAuth, but the native button flow is the recommended path on iOS.

---

## 3. expo-secure-store

Source: https://docs.expo.dev/versions/v56.0.0/sdk/securestore/

Encrypted key-value storage. Android: `SharedPreferences` encrypted via Android Keystore. iOS: keychain services (`kSecClassGenericPassword`).

### Installation
```sh
npx expo install expo-secure-store
```

### Config plugin (app.json)
```json
{
  "expo": {
    "plugins": [
      [
        "expo-secure-store",
        {
          "configureAndroidBackup": true,
          "faceIDPermission": "Allow $(PRODUCT_NAME) to access your Face ID biometric data."
        }
      ]
    ]
  }
}
```
- `configureAndroidBackup` (boolean, default `true`) — Android only; manages Auto Backup compatibility.
- `faceIDPermission` (string) — iOS only; sets `NSFaceIDUsageDescription`.

Bare RN: add `NSFaceIDUsageDescription` to **Info.plist**:
```xml
<key>NSFaceIDUsageDescription</key>
<string>Allow $(PRODUCT_NAME) to access your Face ID biometric data.</string>
```

### Storage details & limits
- iOS data **persists across app uninstalls** with the same bundle ID (not guaranteed — don't rely on it). Android data is **not** preserved on uninstall.
- Size limits: large payloads can be rejected by the platform; historically some iOS releases refused values above ~2048 bytes.
- Data protected with `requireAuthentication: true` becomes inaccessible if the user changes biometric settings.

### Methods
| Method | Returns |
|--------|---------|
| `setItemAsync(key, value, options?)` | `Promise<void>` |
| `getItemAsync(key, options?)` | `Promise<string \| null>` |
| `deleteItemAsync(key, options?)` | `Promise<void>` |
| `setItem(key, value, options?)` (sync, blocks JS thread) | `void` |
| `getItem(key, options?)` (sync, blocks JS thread) | `string \| null` |
| `isAvailableAsync()` | `Promise<boolean>` |
| `canUseBiometricAuthentication()` | `boolean` (always `false` on tvOS) |

### `SecureStoreOptions`
| Property | Type | Platform | Description |
|----------|------|----------|-------------|
| `requireAuthentication` | boolean | Android, iOS | Require device user authentication to access data. Android: `setUserAuthenticationRequired(true)` (API 23+). iOS: `biometryCurrentSet`. |
| `authenticationPrompt` | string | Android, iOS | Message shown during authentication. |
| `keychainAccessible` | KeychainAccessibilityConstant | iOS | When the entry is accessible. **Default `SecureStore.WHEN_UNLOCKED`.** |
| `keychainService` | string | Android, iOS | Service name / keychain alias. If set on write, it is **required** to read the value back. |
| `accessGroup` | string | iOS | Access group for sharing between apps. |

`requireAuthentication` caveats (the top cause of bugs, all stated inline in `packages/expo-secure-store/src/SecureStore.ts:66-103`):
- **Platform asymmetry**: on Android, user authentication is required for *all* operations; on iOS the user is prompted only when **reading or updating an existing value**, not when creating a new one.
- "Complete functionality is unlocked only with a freshly generated key" — it **will not work in tandem with a `keychainService` value already used for non-authenticated operations**. Use a dedicated `keychainService` for authenticated entries.
- Requires a **real device** to test: emulators/simulators do not require biometric authentication when retrieving secrets, unlike real iOS devices.
- Not supported in Expo Go when biometrics are available (missing `NSFaceIDUsageDescription`) — use the config plugin in a dev/release build.

### Keychain accessibility constants
- Recommended: `WHEN_UNLOCKED`, `WHEN_UNLOCKED_THIS_DEVICE_ONLY`, `AFTER_FIRST_UNLOCK`, `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY`.
- Deprecated (less secure): `ALWAYS`, `ALWAYS_THIS_DEVICE_ONLY`.
- Passcode-specific: `WHEN_PASSCODE_SET_THIS_DEVICE_ONLY` (deleted if passcode removed).

### Example
```jsx
import { useState } from 'react';
import { Text, View, StyleSheet, TextInput, Button } from 'react-native';
import * as SecureStore from 'expo-secure-store';

async function save(key, value) {
  await SecureStore.setItemAsync(key, value);
}

async function getValueFor(key) {
  let result = await SecureStore.getItemAsync(key);
  if (result) {
    alert("🔐 Here's your value 🔐 \n" + result);
  } else {
    alert('No values stored under that key.');
  }
}

export default function App() {
  const [key, onChangeKey] = useState('Your key here');
  const [value, onChangeValue] = useState('Your value here');

  return (
    <View style={styles.container}>
      <Text style={styles.paragraph}>Save an item, and grab it later!</Text>
      <Button
        title="Save this key/value pair"
        onPress={() => {
          save(key, value);
          onChangeKey('Your key here');
          onChangeValue('Your value here');
        }}
      />
      <Text style={styles.paragraph}>🔐 Enter your key 🔐</Text>
      <TextInput
        style={styles.textInput}
        onSubmitEditing={event => {
          getValueFor(event.nativeEvent.text);
        }}
        placeholder="Enter the key for the value you want to get"
      />
    </View>
  );
}
```

### SDK 56 notes
- **Expo Go**: `requireAuthentication` is not supported when biometric auth is available, due to a missing `NSFaceIDUsageDescription` key.
- App Store encryption compliance: set `ios.config.usesNonExemptEncryption` to `false` if applicable.
- Android Auto Backup must exclude SecureStore shared preferences or encrypted data becomes unrecoverable (handled by `configureAndroidBackup`).

---

## 4. expo-local-authentication

Source: https://docs.expo.dev/versions/v56.0.0/sdk/local-authentication/

Biometric authentication: Fingerprint (Android), Face ID / Touch ID (iOS). Platforms: Android, iOS, Expo Go (with caveats).

### Installation
```sh
npx expo install expo-local-authentication
```

### Config plugin (app.json)
```json
{
  "expo": {
    "plugins": [
      [
        "expo-local-authentication",
        {
          "faceIDPermission": "Allow $(PRODUCT_NAME) to use Face ID."
        }
      ]
    ]
  }
}
```
- `faceIDPermission` (iOS only, default `"Allow $(PRODUCT_NAME) to use Face ID"`).

Manual iOS (non-CNG) — add to Info.plist:
```xml
<key>NSFaceIDUsageDescription</key>
<string>Allow $(PRODUCT_NAME) to use FaceID</string>
```

### Methods
```js
import * as LocalAuthentication from 'expo-local-authentication';
```
| Method | Platform | Returns |
|--------|----------|---------|
| `authenticateAsync(options?)` | Android, iOS | `Promise<LocalAuthenticationResult>` |
| `cancelAuthenticate()` | Android | `Promise<void>` |
| `getEnrolledLevelAsync()` | Android, iOS | `Promise<SecurityLevel>` |
| `hasHardwareAsync()` | Android, iOS | `Promise<boolean>` |
| `isEnrolledAsync()` | Android, iOS | `Promise<boolean>` |
| `supportedAuthenticationTypesAsync()` | Android, iOS | `Promise<AuthenticationType[]>` |

### `LocalAuthenticationOptions`
| Property | Type | Platform | Notes |
|----------|------|----------|-------|
| `biometricsSecurityLevel` | `'weak' \| 'strong'` | Android | Default `'weak'`. Class 3 vs Class 2-3 biometrics. |
| `cancelLabel` | string | Both | Custom cancel button label. |
| `disableDeviceFallback` | boolean | Both | Prevents passcode fallback after failures. |
| `fallbackLabel` | string | iOS | Custom passcode button label (empty string hides it). |
| `promptDescription` | string | Android | Description in auth dialog. |
| `promptMessage` | string | Both | Message alongside the biometric prompt. |
| `promptSubtitle` | string | Android | Subtitle below the main message. |
| `requireConfirmation` | boolean | Android | Default `true`. Confirmation step after auth. |

### Result & error types
- **`LocalAuthenticationResult`**: success `{ success: true }`; failure `{ success: false, error: LocalAuthenticationError, warning?: string }`.
- **`LocalAuthenticationError`** values: `'not_enrolled' | 'user_cancel' | 'app_cancel' | 'not_available' | 'lockout' | 'no_space' | 'timeout' | 'unable_to_process' | 'unknown' | 'system_cancel' | 'user_fallback' | 'invalid_context' | 'passcode_not_set' | 'authentication_failed'`.
- **`BiometricsSecurityLevel`**: `'weak' | 'strong'` (Android only).

### Enums
- **`AuthenticationType`**: `FINGERPRINT = 1`, `FACIAL_RECOGNITION = 2`, `IRIS = 3` (Android only).
- **`SecurityLevel`**: `NONE = 0`, `SECRET = 1` (PIN/Pattern), `BIOMETRIC_WEAK = 2` (e.g. 2D image-based face unlock; no weak options exist on iOS), `BIOMETRIC_STRONG = 3` (fingerprint, 3D face unlock).
  - `BIOMETRIC` — **deprecated alias**, defined via `Object.defineProperty` with a getter that `console.warn`s and resolves to `BIOMETRIC_WEAK` on Android but `BIOMETRIC_STRONG` on iOS. A comparison like `level >= SecurityLevel.BIOMETRIC` therefore means different things per platform. Always write `BIOMETRIC_WEAK` or `BIOMETRIC_STRONG` explicitly. (`packages/expo-local-authentication/src/LocalAuthentication.types.ts:36-70`)

### Permissions
- Android: `USE_BIOMETRIC`, `USE_FINGERPRINT` (auto-added).
- iOS: `NSFaceIDUsageDescription` required.

### SDK 56 notes
- Face ID auth is **unsupported in Expo Go**; requires a development build.
- On Android devices prior to M, `SECRET` may be returned if only the SIM lock has been enrolled.
- **Patch drift (56.0.5, 2026-07-23)**: "[Android] Ensure concurrent `authenticateAsync` calls don't resolve the active prompt's promise" ([#45954](https://github.com/expo/expo/pull/45954)). Before 56.0.5, calling `authenticateAsync()` again before the first call resolved could cross-resolve the active prompt's promise on Android. Pin `>= 56.0.5` if you call it from more than one place.

---

## 5. expo-crypto

Source: https://docs.expo.dev/versions/v56.0.0/sdk/crypto/

Cryptographic hashing, secure random values, **and AES-GCM encryption**. Platforms: Android, iOS, tvOS, Web.

### Installation
```sh
npx expo install expo-crypto
```

### Methods
| Method | Returns | Notes |
|--------|---------|-------|
| `digestStringAsync(algorithm, data, options?)` | `Promise<string>` | Hash a string; output HEX by default. |
| `digest(algorithm, data)` | `Promise<ArrayBuffer>` | Hash a TypedArray of bytes. |
| `getRandomBytes(byteCount)` | `Uint8Array` | Sync, 0–1024 bytes (anything else throws `TypeError`). Falls back to `Math.random` **only** when `__DEV__` and remote JS debugging is attached (`!global.nativeCallSyncHook \|\| global.__REMOTEDEV__`) — not in normal dev. |
| `getRandomBytesAsync(byteCount)` | `Promise<Uint8Array>` | Async secure random bytes. |
| `getRandomValues(typedArray)` | `T` | Fills in place **and returns the same array** (web-standard behavior). |
| `randomUUID()` | `string` | RFC 4122 v4 UUID. |

```ts
const digest = await Crypto.digestStringAsync(
  Crypto.CryptoDigestAlgorithm.SHA512,
  '🥓 Easy to Digest! 💙'
);

const array = new Uint8Array([1, 2, 3, 4, 5]);
const digest2 = await Crypto.digest(Crypto.CryptoDigestAlgorithm.SHA512, array);

const byteArray = new Uint8Array(16);
Crypto.getRandomValues(byteArray);

const UUID = Crypto.randomUUID();
```

### Enums & options
- **`CryptoDigestAlgorithm`**: `SHA1` (160-bit), `SHA256`, `SHA384`, `SHA512` (collision resistant), `MD5` (Android/iOS only), `MD2`/`MD4` (iOS only).
- **`CryptoEncoding`**: `HEX`, `BASE64` (padded, no line wrapping).
- **`CryptoDigestOptions`**: `{ encoding: CryptoEncoding }` (default `HEX`).

### AES-GCM encryption

Shipped since `expo-crypto` 55.0.0 ([#41249](https://github.com/expo/expo/pull/41249)) and re-exported from the package root (`src/Crypto.ts` does `export * from './aes'`), so `import { aesEncryptAsync } from 'expo-crypto'` works. Docs: `docs/pages/versions/v56.0.0/sdk/crypto.mdx:61-160`.

> **`BinaryInput = string | Uint8Array | ArrayBuffer` — when you pass a string it must be base64-encoded.** This applies to plaintext, IVs, tags and `additionalData`. Passing a raw UTF-8 string silently produces garbage.

**Functions**
| Function | Signature |
|----------|-----------|
| `aesEncryptAsync` | `(plaintext: BinaryInput, key: AESEncryptionKey, options?: AESEncryptOptions) => Promise<AESSealedData>` |
| `aesDecryptAsync` | `(sealedData: AESSealedData, key: AESEncryptionKey, options?: AESDecryptOptions) => Promise<Uint8Array \| string>` (`string` when `output: 'base64'`) |

**`AESEncryptionKey`**
- `static generate(size?: AESKeySize): Promise<AESEncryptionKey>` (default 256).
- `static import(bytes: Uint8Array): Promise<AESEncryptionKey>` / `static import(str: string, encoding: 'hex' | 'base64'): Promise<AESEncryptionKey>` — validates key size.
- `size: AESKeySize`; `bytes(): Promise<Uint8Array>`; `encoded(encoding: 'hex' | 'base64'): Promise<string>` (async because web uses `SubtleCrypto.exportKey`).

**`AESSealedData`**
- `static fromCombined(combined: BinaryInput, config?: AESSealedDataConfig): AESSealedData`
- `static fromParts(iv, ciphertext, tag)` — separate tag; or `static fromParts(iv, ciphertextWithTag, tagLength?)` — tag appended, `tagLength` defaults to 16.
- `ciphertext(options?: { includeTag?: boolean; encoding?: 'base64' | 'bytes' })`, `iv(encoding?)`, `tag(encoding?)`, `combined(encoding?)` — all `Promise`, all default to `'bytes'`.
- Readonly `combinedSize`, `ivSize`, `tagSize`.

**Types & enums**
- `enum AESKeySize { AES128 = 128, AES192 = 192, AES256 = 256 }` — **`AES192` is unsupported on web**.
- `type GCMTagByteLength = 16 | 15 | 14 | 13 | 12 | 8 | 4` — default and recommended 16; **on Apple, 16 is the only supported value for encryption**.
- `AESEncryptOptions`: `nonce?: { length: number } | { bytes: BinaryInput }` (default `{ length: 12 }`), `tagLength?: GCMTagByteLength` (Android/web only, ignored on Apple, default 16), `additionalData?: BinaryInput` (AAD).
- `AESDecryptOptions`: `output?: 'bytes' | 'base64'` (default `'bytes'`), `additionalData?: BinaryInput`.
- `AESSealedDataConfig`: `ivLength` (default 12), `tagLength` (default 16) — used when parsing combined data.

```ts
import { AESEncryptionKey, AESSealedData, aesEncryptAsync, aesDecryptAsync } from 'expo-crypto';
import * as SecureStore from 'expo-secure-store';

// Encrypt, then persist the key in SecureStore (the canonical security pattern).
const key = await AESEncryptionKey.generate();
const sealed = await aesEncryptAsync(btoa('Hello, world!'), key); // string input must be base64
const encryptedBytes = await sealed.combined();                   // Uint8Array: iv + ciphertext + tag
await SecureStore.setItemAsync('aes-encryption-key', await key.encoded('hex'));

// Later: reload the key and decrypt.
const keyHex = await SecureStore.getItemAsync('aes-encryption-key');
const restored = await AESEncryptionKey.import(keyHex!, 'hex');
const plaintext = atob(await aesDecryptAsync(AESSealedData.fromCombined(encryptedBytes), restored, {
  output: 'base64',
}));
```

### Web / errors
- On web, hashing requires a secure origin (HTTPS) or an error is thrown.
- Error codes: `ERR_CRYPTO_UNAVAILABLE` (web, non-secure origin), `ERR_CRYPTO_DIGEST` (invalid encoding type), `ERR_CRYPTO` (generic `CryptoError`, a `TypeError` subclass).

---

## 6. expo-apple-authentication

Source: https://docs.expo.dev/versions/v56.0.0/sdk/apple-authentication/

Native "Sign In with Apple". Platforms: iOS, tvOS, Expo Go.

### Installation
```sh
npx expo install expo-apple-authentication
```

### Configuration
```json
{
  "expo": {
    "ios": { "usesAppleSignIn": true },
    "plugins": ["expo-apple-authentication"]
  }
}
```
Manual iOS (non-EAS): enable the Apple Sign In capability and add entitlements to `ios/[app]/[app].entitlements`:
```xml
<key>com.apple.developer.applesignin</key>
<array>
  <string>Default</string>
</array>
```
Set `CFBundleAllowMixedLocalizations` to `true` in Info.plist for localization.

### Component — `AppleAuthenticationButton`
Native Apple button; renders nothing when unavailable. **Do not** set `backgroundColor` or `borderRadius` via `style` (App Store guideline) — use `buttonStyle` and `cornerRadius`. Must specify height and width.

Props: `buttonType` (`AppleAuthenticationButtonType`), `buttonStyle` (`AppleAuthenticationButtonStyle`), `cornerRadius?` (number), `onPress: () => void`, `style?` (ViewStyle, excludes backgroundColor/borderRadius).

```jsx
import * as AppleAuthentication from 'expo-apple-authentication';
import { View, StyleSheet } from 'react-native';

export default function App() {
  return (
    <View style={styles.container}>
      <AppleAuthentication.AppleAuthenticationButton
        buttonType={AppleAuthentication.AppleAuthenticationButtonType.SIGN_IN}
        buttonStyle={AppleAuthentication.AppleAuthenticationButtonStyle.BLACK}
        cornerRadius={5}
        style={styles.button}
        onPress={async () => {
          try {
            const credential = await AppleAuthentication.signInAsync({
              requestedScopes: [
                AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
                AppleAuthentication.AppleAuthenticationScope.EMAIL,
              ],
            });
          } catch (e) {
            if (e.code === 'ERR_REQUEST_CANCELED') {
              // handle user cancellation
            }
          }
        }}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  button: { width: 200, height: 44 },
});
```

### Methods
| Method | Returns | Notes |
|--------|---------|-------|
| `signInAsync(options?)` | `Promise<AppleAuthenticationCredential>` | Presents sign-in modal. Full name & email only on first sign-in. |
| `refreshAsync(options)` | `Promise<AppleAuthenticationCredential>` | `options.user` required. |
| `signOutAsync(options)` | `Promise<AppleAuthenticationCredential>` | `options.user` required; not recommended for typical logout. |
| `getCredentialStateAsync(user)` | `Promise<AppleAuthenticationCredentialState>` | Real device only — Simulator throws. |
| `isAvailableAsync()` | `Promise<boolean>` | OS support check. |
| `formatFullName(fullName, formatStyle?)` | `string` | Locale-aware name formatting. |

`signInAsync` options: `{ requestedScopes?: AppleAuthenticationScope[]; state?: string; nonce?: string }`.
```javascript
const credential = await AppleAuthentication.signInAsync({
  requestedScopes: [
    AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
    AppleAuthentication.AppleAuthenticationScope.EMAIL,
  ],
});
```

### Event subscription
```javascript
const subscription = AppleAuthentication.addRevokeListener(() => {
  console.log('Credentials revoked');
});
subscription.remove();
```

### Enums
- **`AppleAuthenticationButtonType`**: `SIGN_IN = 0`, `CONTINUE = 1`, `SIGN_UP = 2` (iOS 13.2+).
- **`AppleAuthenticationButtonStyle`**: `WHITE = 0`, `WHITE_OUTLINE = 1`, `BLACK = 2`.
- **`AppleAuthenticationScope`**: `FULL_NAME = 0`, `EMAIL = 1`.
- **`AppleAuthenticationCredentialState`**: `REVOKED = 0`, `AUTHORIZED = 1`, `NOT_FOUND = 2`, `TRANSFERRED = 3`.
- **`AppleAuthenticationUserDetectionStatus`**: `UNSUPPORTED = 0`, `UNKNOWN = 1`, `LIKELY_REAL = 2`.
- **`AppleAuthenticationOperation`**: `IMPLICIT = 0`, `LOGIN = 1`, `REFRESH = 2`, `LOGOUT = 3`.
- **`AppleAuthenticationFullNameFormatStyle`**: `'default' | 'short' | 'medium' | 'long' | 'abbreviated'`.

### Types
```typescript
// AppleAuthenticationCredential
{
  user: string;                    // stable user identifier
  authorizationCode: string | null;
  identityToken: string | null;    // JWT with user info
  email: string | null;            // null if not first sign-in
  fullName: AppleAuthenticationFullName | null;
  realUserStatus: AppleAuthenticationUserDetectionStatus;
  state: string | null;            // echo of request state
}

// AppleAuthenticationFullName
{ givenName: string | null; familyName: string | null; middleName: string | null;
  namePrefix: string | null; nameSuffix: string | null; nickname: string | null; }

// AppleAuthenticationSignInOptions
{ requestedScopes?: AppleAuthenticationScope[]; state?: string; nonce?: string }

// AppleAuthenticationRefreshOptions
{ user: string; requestedScopes?: AppleAuthenticationScope[]; state?: string }

// AppleAuthenticationSignOutOptions
{ user: string; state?: string }
```

### Error codes
`ERR_REQUEST_CANCELED`, `ERR_INVALID_OPERATION`, `ERR_INVALID_RESPONSE`, `ERR_INVALID_SCOPE`, `ERR_REQUEST_FAILED`, `ERR_REQUEST_NOT_HANDLED`, `ERR_REQUEST_NOT_INTERACTIVE`, `ERR_REQUEST_UNKNOWN`.

### SDK 56 notes
- App Store compliance: if you offer third-party auth, you must also offer Apple authentication.
- Scopes (name/email) only provided on initial sign-in; subsequent requests return `null`.
- Store credentials server-side or via `expo-secure-store`.
- Verify JWT identity tokens with Apple's public keys at `https://appleid.apple.com/auth/keys`.
- `getCredentialStateAsync` requires a real device (Simulator throws).

---

## 7. expo-web-browser

Source: https://docs.expo.dev/versions/v56.0.0/sdk/webbrowser/

System browser access with redirect handling. Android uses ChromeCustomTabs; iOS uses `SFSafariViewController` (general browsing) or `ASWebAuthenticationSession` (auth). Note: as of iOS 11, `SFSafariViewController` no longer shares cookies with Safari.

Use `openAuthSessionAsync` for auth flows, `openBrowserAsync` for general browsing.

### Installation
```sh
npx expo install expo-web-browser
```
### Config plugin
The docs page advertises a built-in config plugin for CNG but auto-generates no property list for it. The only option that exists in SDK 56 is:
```json
{ "plugins": [["expo-web-browser", { "experimentalLauncherActivity": true }]] }
```
- `experimentalLauncherActivity` (boolean, Android only, undocumented). When `true`, the plugin injects a `.BrowserLauncherActivity` with a `MAIN`/`LAUNCHER` intent filter into **AndroidManifest.xml** and writes a `BrowserLauncherActivity.kt` into the project. When falsy (or the plugin is passed no props) the plugin is a no-op. (`packages/expo-web-browser/plugin/src/withWebBrowser.ts`, `withWebBrowserAndroid.ts`)

### Methods
| Method | Platform | Returns |
|--------|----------|---------|
| `openBrowserAsync(url, browserParams?)` | All | `Promise<WebBrowserResult>` |
| `openAuthSessionAsync(url, redirectUrl?, options?)` | All | `Promise<WebBrowserAuthSessionResult>` |
| `dismissBrowser()` | iOS | `Promise<{ type: DISMISS }>` |
| `dismissAuthSession()` | iOS, Web | `void` |
| `maybeCompleteAuthSession(options?)` | Web | `WebBrowserCompleteAuthSessionResult` |
| `warmUpAsync(browserPackage?)` | Android | `Promise<WebBrowserWarmUpResult>` |
| `coolDownAsync(browserPackage?)` | Android | `Promise<WebBrowserCoolDownResult>` |
| `mayInitWithUrlAsync(url, browserPackage?)` | Android | `Promise<WebBrowserMayInitWithUrlResult>` |
| `getCustomTabsSupportingBrowsersAsync()` | Android | `Promise<WebBrowserCustomTabsResults>` |

**`openAuthSessionAsync`** — on iOS the default path uses the legacy `callbackURLScheme` API, so the redirect must be a **custom scheme** from `expo.scheme` (e.g. `demo://`), not HTTPS. Set `preferUniversalLinks: true` to use HTTPS universal-link callbacks on **iOS 17.4+**; that requires the Associated Domains entitlement configured for the redirect host. On web: requires HTTPS/localhost, must be called immediately after user interaction, uses localStorage for state verification (same-origin), and polls every 1s for window close. `redirectUrl` defaults to `Linking.createURL("")` on web.

**`maybeCompleteAuthSession`** failure reasons: not supported on platform; non-browser environment; no auth session in progress; URL mismatch (use `skipRedirectCheck: true` in dev).

```jsx
import { useState } from 'react';
import { Button, Text, View, StyleSheet } from 'react-native';
import * as WebBrowser from 'expo-web-browser';

export default function App() {
  const [result, setResult] = useState(null);

  const _handlePressButtonAsync = async () => {
    let result = await WebBrowser.openBrowserAsync('https://expo.dev');
    setResult(result);
  };

  return (
    <View style={styles.container}>
      <Button title="Open WebBrowser" onPress={_handlePressButtonAsync} />
      <Text>{result && JSON.stringify(result)}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#ecf0f1',
  },
});
```

### Option types
**`WebBrowserOpenOptions`**: `toolbarColor` (all), `secondaryToolbarColor` (Android), `controlsColor` (iOS), `browserPackage` (Android), `showTitle` (Android), `showInRecents` (Android, **default `false`**, requires `createTask: true`), `enableBarCollapsing` (all), `enableDefaultShareMenuItem` (Android), `createTask` (Android, default `true`), `useProxyActivity` (Android, default `true`), `dismissButtonStyle` (`'done' | 'close' | 'cancel'`, iOS), `readerMode` (iOS), `presentationStyle` (iOS, default `OverFullScreen`), `windowName` (Web), `windowFeatures` (Web).

- `useProxyActivity` (default `true`) launches the browser through a transparent proxy activity with a different task affinity so the browser is not destroyed when the app is backgrounded. It only applies when `createTask` is `true`, and **when it is `true`, `showInRecents` is always treated as `true`** regardless of what you pass. Set it to `false` for the legacy direct-launch behavior.

**`AuthSessionOpenOptions`** (extends `WebBrowserOpenOptions`): `preferEphemeralSession` (iOS, default `false` — asks the browser not to share cookies with the user's normal session; honouring it is up to the browser), `preferUniversalLinks` (iOS 17.4+, default `false` — see `openAuthSessionAsync` above). On Android there is no native AuthSession implementation, so the inherited `WebBrowserOpenOptions` are used by the browser polyfill; on iOS they are ignored.

### Result types
```typescript
WebBrowserResult            // { type: WebBrowserResultType }
WebBrowserRedirectResult    // { type: 'success'; url: string }
WebBrowserAuthSessionResult // WebBrowserRedirectResult | WebBrowserResult
WebBrowserCompleteAuthSessionResult // { type: 'success' | 'failed'; message: string }
```

### Enums
- **`WebBrowserResultType`**: `OPENED` (Android), `CANCEL` (iOS), `DISMISS` (iOS), `LOCKED`.
- **`WebBrowserPresentationStyle`** (iOS): `AUTOMATIC`, `FULL_SCREEN`, `PAGE_SHEET`, `FORM_SHEET`, `POPOVER`, `OVER_FULL_SCREEN` (default), `OVER_CURRENT_CONTEXT`, `CURRENT_CONTEXT`.

### Error codes (web)
`ERR_WEB_BROWSER_REDIRECT` (parent window lost reference), `ERR_WEB_BROWSER_BLOCKED` (popup blocked / `window.open` too late), `ERR_WEB_BROWSER_CRYPTO` (insecure origin).

### Deep links
With Expo Router, deep links are handled automatically — manual `Linking` listeners are not required for auth flows.

### SDK 56 notes
- **Patch drift (56.0.6, 2026-07-23)**: "[iOS] Fixed `openAuthSessionAsync` hanging forever when the authentication session fails to start" ([#47896](https://github.com/expo/expo/pull/47896), issue [#47653](https://github.com/expo/expo/issues/47653)). Pin `expo-web-browser >= 56.0.6` — earlier 56.x builds leave the promise unresolved instead of rejecting.

---

## 8. Expo Router authentication pattern

Sources (both unversioned — the router docs are not per-SDK):
- https://docs.expo.dev/router/advanced/authentication/
- https://docs.expo.dev/router/advanced/protected/

Primary SDK 56 approach uses **`Stack.Protected`** with a `guard` prop. When a guard evaluates false, the router redirects to the navigator's **anchor route** (usually `index`); if the anchor is itself protected, it falls through to the **first available screen**. When a screen's guard flips `true → false`, all of its history entries are removed. For general router concepts see `references/02-expo-router.md`.

### `Protected` is not Stack-only
`Protected` is a single component (`ProtectedProps = { guard: boolean; children?: ReactNode }`) attached to every navigator, so `Tabs.Protected`, `Drawer.Protected`, top-tabs `.Protected`, the experimental modal stack, and any custom navigator built with `withLayoutContext` all work identically. `Protected` wrappers nest, so you can compose guards (for example logged-in **and** admin).

> **Static rendering caveat**: protected screens are evaluated **client-side only**. During static export no HTML is generated for them, but a user who knows the URL can still fetch the corresponding HTML/JS. `Protected` is not a replacement for server-side authentication or access control.

### File structure
```
src/app/
  _layout.tsx        (root with SessionProvider)
  sign-in.tsx        (always accessible)
  (app)/             (protected group)
    _layout.tsx
    index.tsx
```

### `ctx.tsx` — SessionProvider / AuthContext
```tsx
import { use, createContext, type PropsWithChildren } from 'react';

import { useStorageState } from './useStorageState';

// Default MUST be `null` — with a non-null default the `useSession` guard below can never
// fire and the "must be wrapped in a <SessionProvider />" developer error is swallowed.
const AuthContext = createContext<{
  signIn: () => void;
  signOut: () => void;
  session?: string | null;
  isLoading: boolean;
} | null>(null);

// Use this hook to access the user info.
export function useSession() {
  const value = use(AuthContext);
  if (!value) {
    throw new Error('useSession must be wrapped in a <SessionProvider />');
  }

  return value;
}

export function SessionProvider({ children }: PropsWithChildren) {
  const [[isLoading, session], setSession] = useStorageState('session');

  return (
    <AuthContext.Provider
      value={{
        signIn: () => {
          // Perform sign-in logic here
          setSession('xxx');
        },
        signOut: () => {
          setSession(null);
        },
        session,
        isLoading,
      }}>
      {children}
    </AuthContext.Provider>
  );
}
```

### `useStorageState.ts` — SecureStore (native) / localStorage (web)
```tsx
import  { useEffect, useCallback, useReducer } from 'react';
import * as SecureStore from 'expo-secure-store';
import { Platform } from 'react-native';

type UseStateHook<T> = [[boolean, T | null], (value: T | null) => void];

function useAsyncState<T>(
  initialValue: [boolean, T | null] = [true, null],
): UseStateHook<T> {
  return useReducer(
    (state: [boolean, T | null], action: T | null = null): [boolean, T | null] => [false, action],
    initialValue
  ) as UseStateHook<T>;
}

export async function setStorageItemAsync(key: string, value: string | null) {
  if (Platform.OS === 'web') {
    try {
      if (value === null) {
        localStorage.removeItem(key);
      } else {
        localStorage.setItem(key, value);
      }
    } catch (e) {
      console.error('Local storage is unavailable:', e);
    }
  } else {
    if (value == null) {
      await SecureStore.deleteItemAsync(key);
    } else {
      await SecureStore.setItemAsync(key, value);
    }
  }
}

export function useStorageState(key: string): UseStateHook<string> {
  // Public
  const [state, setState] = useAsyncState<string>();

  // Get
  useEffect(() => {
    if (Platform.OS === 'web') {
      try {
        if (typeof localStorage !== 'undefined') {
          setState(localStorage.getItem(key));
        }
      } catch (e) {
        console.error('Local storage is unavailable:', e);
      }
    } else {
      SecureStore.getItemAsync(key).then((value: string | null) => {
        setState(value);
      });
    }
  }, [key]);

  // Set
  const setValue = useCallback(
    (value: string | null) => {
      setState(value);
      setStorageItemAsync(key, value);
    },
    [key]
  );

  return [state, setValue];
}
```

### Root `_layout.tsx` (with `Stack.Protected`)
```tsx
import { Stack } from 'expo-router';

import { SessionProvider, useSession } from '@/ctx';
import { SplashScreenController } from '@/splash';

export default function Root() {
  // Set up the auth context and render your layout inside of it.
  return (
    <SessionProvider>
      <SplashScreenController />
      <RootNavigator />
    </SessionProvider>
  );
}

// Create a new component that can access the SessionProvider context later.
function RootNavigator() {
  const { session } = useSession();

  return (
    <Stack>
      <Stack.Protected guard={!!session}>
        <Stack.Screen name="(app)" />
      </Stack.Protected>

      <Stack.Protected guard={!session}>
        <Stack.Screen name="sign-in" />
      </Stack.Protected>
    </Stack>
  );
}
```

### `splash.tsx` — SplashScreenController
```tsx
import { SplashScreen } from 'expo-router';
import { useSession } from './ctx';

SplashScreen.preventAutoHideAsync();

export function SplashScreenController() {
  const { isLoading } = useSession();

  if (!isLoading) {
    SplashScreen.hide();
  }

  return null;
}
```

### `sign-in.tsx`
```tsx
import { router } from 'expo-router';
import { Text, View } from 'react-native';

import { useSession } from '@/ctx';

export default function SignIn() {
  const { signIn } = useSession();
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text
        onPress={() => {
          signIn();
          // Navigate after signing in. You may want to tweak this to ensure sign-in is successful before navigating.
          router.replace('/');
        }}>
        Sign In
      </Text>
    </View>
  );
}
```

### `(app)/_layout.tsx`
```tsx
import { Stack } from 'expo-router';

export default function AppLayout() {
  // This renders the navigation stack for all authenticated app routes.
  return <Stack />;
}
```

### `(app)/index.tsx`
```tsx
import { Text, View } from 'react-native';

import { useSession } from '@/ctx';

export default function Index() {
  const { signOut } = useSession();
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text
        onPress={() => {
          // The guard in `RootNavigator` redirects back to the sign-in screen.
          signOut();
        }}>
        Sign Out
      </Text>
    </View>
  );
}
```

### Alternative — Modal sign-in pattern
Render sign-in as a modal over the app instead of redirecting, which preserves deep links when auth completes:
```tsx
import { Stack } from 'expo-router';

export const unstable_settings = {
  initialRouteName: '(root)',
};

export default function AppLayout() {
  return (
    <Stack>
      <Stack.Screen name="(root)" />
      <Stack.Screen
        name="sign-in"
        options={{
          presentation: 'modal',
        }}
      />
    </Stack>
  );
}
```

### SDK 56 notes
- The `Stack.Protected guard={...}` API is the **current** recommended pattern. The older redirect-based approach (using `<Redirect>` / `useRouter` inside layouts) is documented separately as "Authentication (redirects)" for **SDK 52 and earlier**.
- Web currently lacks server-side middleware support, so route protection on web relies on client-side rendering/redirects.

---

## 9. SDK 56 notes summary

- **expo-auth-session**: `getDefaultReturnUrl()` is deprecated — use `makeRedirectUri()`. PKCE on by default (`usePKCE: true`, `S256`). Provider libraries (`@react-native-google-signin/google-signin`, `react-native-fbsdk-next`) recommended over raw AuthSession for Google/Facebook.
- **expo-secure-store**: `requireAuthentication` unsupported in Expo Go when biometrics are available; `configureAndroidBackup` plugin option (default `true`) prevents Auto Backup from breaking encrypted data; sync `getItem`/`setItem` and `canUseBiometricAuthentication()` available.
- **expo-local-authentication**: Face ID unsupported in Expo Go (use a dev build); `biometricsSecurityLevel` (Android) selects weak vs strong class.
- **expo-crypto**: hashing + secure random + AES-GCM (`aesEncryptAsync`/`aesDecryptAsync`, since 55.0.0) across Android/iOS/tvOS/Web; web hashing requires a secure origin.
- **expo-apple-authentication**: `usesAppleSignIn: true` in app config; `addRevokeListener`; scopes only on first sign-in.
- **expo-web-browser**: `AuthSessionOpenOptions.preferUniversalLinks` (iOS 17.4+); `preferEphemeralSession` for private auth sessions. Pin `>= 56.0.6` for the iOS `openAuthSessionAsync` hang fix.
- **Expo Router**: `Protected` guard prop is the current authentication pattern (replaces the SDK 52-and-earlier redirect approach) and is available on all navigators, not just `Stack`.

---

## SDK 57 delta

**Nothing in this domain changed in SDK 57.** All six packages this file owns — `expo-auth-session`, `expo-secure-store`, `expo-local-authentication`, `expo-crypto`, `expo-apple-authentication`, `expo-web-browser` — plus the router `Protected` API, are API-identical between SDK 56 and SDK 57. Everything above applies unchanged; the version pins move, and one CLI-level `prebuild` behaviour change (below) can affect hand-edited native files.

Evidence (release branches only — `origin/sdk-56` vs `origin/sdk-57`): `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-{auth-session,secure-store,local-authentication,crypto,apple-authentication,web-browser}` touches only CHANGELOGs, `package.json`/`android/build.gradle` version strings, and four prose-only JSDoc hunks in `expo-auth-session`. Every `## 57.0.0 — 2026-06-25` entry in those six CHANGELOGs reads *"This version does not introduce any user-facing changes."* Each release branch keeps its own SDK's docs under `docs/pages/versions/unversioned/`, and `sdk/{auth-session,securestore,local-authentication,apple-authentication,webbrowser,crypto}.mdx`, `docs/pages/router/advanced/{authentication,protected}.mdx` and `docs/pages/guides/authentication.mdx` are all byte-identical between the two branches.

### Version pins (56 → 57)
From `packages/expo/bundledNativeModules.json` on the release branches (`origin/sdk-56` / `origin/sdk-57`) — this is what `expo install` actually resolves against. Do **not** read the frozen `docs/public/static/schemas/v5*.0.0/native-modules.json` files; they are stale (they still say `expo-router ~56.2.9`).

| Package | SDK 56 | SDK 57 |
|---------|--------|--------|
| `expo-auth-session` | `~56.0.16` | `~57.0.5` |
| `expo-secure-store` | `~56.0.4` | `~57.0.1` |
| `expo-local-authentication` | `~56.0.5` | `~57.0.2` |
| `expo-crypto` | `~56.0.4` | `~57.0.1` |
| `expo-apple-authentication` | `~56.0.4` | `~57.0.1` |
| `expo-web-browser` | `~56.0.6` | `~57.0.2` |
| `expo-router` | `~56.2.16` | `~57.0.8` |

The 57 pins are **not** a flat `~57.0.0` — each package is on its own patch. Nothing in this domain's dependency graph pulls in a third-party version change either: `react-native-screens` is `~4.26.0` on **both** branches.

**Backports, not 57 deltas.** The two behaviour fixes noted inline above shipped on the SDK 56 line as well, so "upgrade to 57 to get them" is wrong advice — bump the patch on the line you are already on:

| Fix | Minimum SDK 56 patch | Also in |
|-----|----------------------|---------|
| iOS `openAuthSessionAsync` no longer hangs when the session fails to start ([#47896](https://github.com/expo/expo/pull/47896)) | `expo-web-browser >= 56.0.6` | `expo-web-browser` 57.0.2 |
| Android concurrent `authenticateAsync` no longer resolves the active prompt's promise ([#45954](https://github.com/expo/expo/pull/45954)) | `expo-local-authentication >= 56.0.5` | `expo-local-authentication` 57.0.2 |

### Terminology rename (docs prose only)
SDK 57 renames "bare workflow"/"standalone" to "existing React Native project"/"production build" throughout the `expo-auth-session` API reference. No behaviour change; the option names are unchanged. Concretely (`packages/expo-auth-session/src/AuthSession.ts:46-52`, `AuthSession.types.ts:74-80`):
- `makeRedirectUri`'s bullets now read **"CNG projects:** Uses the `scheme` property" / **"Existing React Native apps:** Falls back to using the `native` option".
- `AuthSessionRedirectUriOptions.native` now reads "Manual scheme to use in existing React Native projects and production builds".
- `getRedirectUrl` now says it throws "in an existing React Native project on native" (was "if you're using the bare workflow on native").
- `GoogleAuthRequestConfig.iosClientId` / `androidClientId` and the Facebook equivalents get the same rewording.

### The one real 57 change that touches this domain
**`npx expo prebuild` cleans by default in SDK 57** ([#47209](https://github.com/expo/expo/pull/47209), `@expo/cli` — 57-only, not backported to the 56 line). It wipes `ios/` and `android/`, but the config plugins in this file (`expo-secure-store`, `expo-local-authentication`, `expo-apple-authentication`, `expo-web-browser`) are all CNG-managed and regenerate. Only hand-edited native entitlements/Info.plist entries — the manual, non-CNG paths documented in sections 3, 4 and 6 — are at risk; pass `--no-clean` or move them into a config plugin.

### Explicitly *not* affected
So you do not go hunting:
- **iOS scene-based lifecycle / `SceneDelegate` / `UIApplicationSceneManifest`** — not in 57; landed after the 57 cut. `templates/expo-template-bare-minimum/ios` is byte-identical between `origin/sdk-56` and `origin/sdk-57`, and `AppDelegate.swift` on `origin/sdk-57` still overrides `application(_:open:options:)` and `application(_:continue:restorationHandler:)` with `RCTLinkingManager`. OAuth custom-scheme and universal-link redirects reach the app exactly as in 56.
- **Android R8/ProGuard defaults** — not in 57; landed after the 57 cut. `origin/sdk-57:templates/expo-template-bare-minimum/android/app/build.gradle` still uses `getDefaultProguardFile("proguard-android.txt")` (not `proguard-android-optimize.txt`); no new reflection/Keystore risk for SecureStore.
- **`Protected` gaining a `redirectTo` prop** — not in 57; landed after the 57 cut. Verified against the published tarballs: `expo-router@56.2.16` and `expo-router@57.0.8` both declare `build/views/Protected.d.ts` as exactly `export type ProtectedProps = { guard: boolean; children?: ReactNode }`.
