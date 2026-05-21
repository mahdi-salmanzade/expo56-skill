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

---

## 1. expo-auth-session

Source: https://docs.expo.dev/versions/v56.0.0/sdk/auth-session/

Browser-based OAuth 2.0 and OpenID Connect flows built on top of `WebBrowser` and `Crypto`. Full support on Android, iOS, Web, and Expo Go.

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
- Properties: `accessToken`, `expiresIn?`, `refreshToken?`, `idToken?`, `tokenType?`, `scope?`, `state?`, `issuedAt?`, `rawResponse?`.
- Methods:
  - `fromQueryParams(params): TokenResponse` (static-style; used for implicit grant)
  - `getRequestConfig(): TokenResponseConfig`
  - `isTokenFresh(token, secondsMargin?): boolean`
  - `shouldRefresh(): boolean`
  - `refreshAsync(config, discovery): Promise<TokenResponse>`
```ts
const newToken = await TokenResponse.refreshAsync(
  {
    clientId: 'YOUR_CLIENT_ID',
    refreshToken: oldToken.refreshToken,
  },
  discovery
);
```

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
// Web dev: https://localhost:19006/redirect
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
`AuthSessionRedirectUriOptions`: `scheme?`, `path?`, `preferLocalhost?` (iOS simulator only, default `false`), `isTripleSlashed?` (default `false`), `native?` (bare/standalone scheme), `queryParams?`.

### Configuration types

**`AuthRequestConfig`**: `clientId` (req), `clientSecret?`, `redirectUri` (req), `scopes?`, `state?`, `responseType?` (default `ResponseType.Code`), `prompt?`, `codeChallenge?`, `codeChallengeMethod?` (default `S256`), `usePKCE?` (default `true`), `extraParams?`.

**`AccessTokenRequestConfig`** (extends `TokenRequestConfig`): + `code` (req), `redirectUri` (req).

**`RefreshTokenRequestConfig`** (extends `TokenRequestConfig`): + `refreshToken?`.

**`RevokeTokenRequestConfig`** (extends `Partial<TokenRequestConfig>`): + `token` (req), `tokenTypeHint?`.

**`TokenRequestConfig`** (base): `clientId` (req), `clientSecret?`, `scopes?`, `extraParams?`, `extraHeaders?`.

**`TokenResponseConfig`**: `accessToken` (req), `expiresIn?`, `refreshToken?`, `idToken?`, `tokenType?`, `scope?`, `state?`, `issuedAt?`.

**`ServerTokenResponseConfig`** (provider snake_case form): `access_token` (req), `expires_in?`, `refresh_token?`, `id_token?`, `token_type?`, `scope?`, `issued_at?`.

**`AuthRequestPromptOptions`** (extends `Omit<AuthSessionOpenOptions, 'windowFeatures'>`): `url?`, `windowFeatures?` (web only).

**`DiscoveryDocument`**: `authorizationEndpoint?`, `tokenEndpoint?`, `revocationEndpoint?`, `userInfoEndpoint?`, `endSessionEndpoint?`, `registrationEndpoint?`, `discoveryDocument?` (`ProviderMetadata`). `AuthDiscoveryDocument` = `Pick<DiscoveryDocument, 'authorizationEndpoint'>`. `ProviderMetadata` carries the full OpenID provider metadata (`authorization_endpoint`, `token_endpoint`, `jwks_uri`, `scopes_supported`, `code_challenge_methods_supported`, etc.).

### Result type — `AuthSessionResult` (discriminated union)
```ts
// cancel / dismiss
{ type: 'cancel' | 'dismiss' }

// success
{ type: 'success'; authentication: TokenResponse | null; params: Record<string, string>; url: string }

// error
{ type: 'error'; error: AuthError | null; errorCode: string | null; params: Record<string, string>; url: string }
```

### Enums
- **`ResponseType`**: `Code = "code"`, `Token = "token"`, `IdToken = "id_token"`.
- **`CodeChallengeMethod`**: `Plain = "plain"` (not recommended), `S256 = "S256"` (recommended).
- **`GrantType`**: `AuthorizationCode`, `Implicit`, `RefreshToken`, `ClientCredentials`.
- **`Prompt`**: `None`, `Login`, `Consent`, `SelectAccount` (can pass an array).
- **Type aliases**: `IssuerOrDiscovery = Issuer | DiscoveryDocument`; `Issuer = string`; `TokenType = 'bearer' | 'mac'`; `TokenTypeHint` (`AccessToken | RefreshToken`); `PromptMethod(options?) => Promise<AuthSessionResult>`.

### Advanced notes
- **Web popup close**: call `WebBrowser.maybeCompleteAuthSession()` to close the popup after auth.
- **Implicit grant**: with `ResponseType.Token`, pass `response.params` to `TokenResponse.fromQueryParams()`.
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
- Client secrets must be exchanged on a server: "Since your client application code is not a secure place to store secrets, it is necessary to exchange the authorization code in a server."
- The Authentication guide ships complete worked examples for **GitHub** and **Okta**. (Google/Apple are no longer documented as raw AuthSession recipes here — see below.)

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
| `requireAuthentication` | boolean | Android, iOS | Require biometric auth to access data. |
| `authenticationPrompt` | string | Android, iOS | Message shown during authentication. |
| `keychainAccessible` | KeychainAccessibilityConstant | iOS | When the entry is accessible. |
| `keychainService` | string | Android, iOS | Service name / keychain alias. |
| `accessGroup` | string | iOS | Access group for sharing between apps. |

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
- **`SecurityLevel`**: `NONE = 0`, `SECRET = 1` (PIN/Pattern), `BIOMETRIC_WEAK = 2`, `BIOMETRIC_STRONG = 3`.

### Permissions
- Android: `USE_BIOMETRIC`, `USE_FINGERPRINT` (auto-added).
- iOS: `NSFaceIDUsageDescription` required.

### SDK 56 notes
- Face ID auth is **unsupported in Expo Go**; requires a development build.
- On Android devices prior to M, `SECRET` may be returned if only the SIM lock has been enrolled.

---

## 5. expo-crypto

Source: https://docs.expo.dev/versions/v56.0.0/sdk/crypto/

Cryptographic hashing and secure random values. Platforms: Android, iOS, tvOS, Web.

### Installation
```sh
npx expo install expo-crypto
```

### Methods
| Method | Returns | Notes |
|--------|---------|-------|
| `digestStringAsync(algorithm, data, options?)` | `Promise<string>` | Hash a string; output HEX by default. |
| `digest(algorithm, data)` | `Promise<ArrayBuffer>` | Hash a TypedArray of bytes. |
| `getRandomBytes(byteCount)` | `Uint8Array` | Sync, 0–1024 bytes; falls back to `Math.random` in dev. |
| `getRandomBytesAsync(byteCount)` | `Promise<Uint8Array>` | Async secure random bytes. |
| `getRandomValues(typedArray)` | (fills in place) | Web-standard behavior. |
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

### Web / errors
- On web, hashing requires a secure origin (HTTPS) or an error is thrown.
- Error codes: `ERR_CRYPTO_UNAVAILABLE` (web, non-secure origin), `ERR_CRYPTO_DIGEST` (invalid encoding type).

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
Supports built-in config plugins for CNG; manual config otherwise.

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

**`openAuthSessionAsync`** — iOS requires the redirect URI to use a custom scheme from `expo.scheme` (e.g. `demo://`), not HTTPS. On web: requires HTTPS/localhost, must be called immediately after user interaction, uses localStorage for state verification (same-origin), and polls every 1s for window close. `redirectUrl` defaults to `Linking.createURL("")` on web.

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
**`WebBrowserOpenOptions`**: `toolbarColor` (all), `secondaryToolbarColor` (Android), `controlsColor` (iOS), `browserPackage` (Android), `showTitle` (Android), `showInRecents` (Android, requires `createTask: true`), `enableBarCollapsing` (all), `enableDefaultShareMenuItem` (Android), `createTask` (Android, default `true`), `useProxyActivity` (Android, default `true`), `dismissButtonStyle` (`'done' | 'close' | 'cancel'`, iOS), `readerMode` (iOS), `presentationStyle` (iOS, default `OverFullScreen`), `windowName` (Web), `windowFeatures` (Web).

**`AuthSessionOpenOptions`** (extends `WebBrowserOpenOptions`): `preferEphemeralSession` (iOS, default `false`), `preferUniversalLinks` (iOS 17.4+, default `false`).

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

---

## 8. Expo Router authentication pattern

Source: https://docs.expo.dev/router/advanced/authentication/

Primary SDK 56 approach uses **`Stack.Protected`** with a `guard` prop. When a guard evaluates false, the user is redirected to the anchor / first available route.

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

const AuthContext = createContext<{
  signIn: () => void;
  signOut: () => void;
  session?: string | null;
  isLoading: boolean;
}>({
  signIn: () => null,
  signOut: () => null,
  session: null,
  isLoading: false,
});

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
- **expo-crypto**: full API stable across Android/iOS/tvOS/Web; web hashing requires a secure origin.
- **expo-apple-authentication**: `usesAppleSignIn: true` in app config; `addRevokeListener`; scopes only on first sign-in.
- **expo-web-browser**: `AuthSessionOpenOptions.preferUniversalLinks` (iOS 17.4+); `preferEphemeralSession` for private auth sessions.
- **Expo Router**: `Stack.Protected` guard prop is the current authentication pattern (replaces the SDK 52-and-earlier redirect approach).

### Pages that could not be fully captured
- **Google AuthSession recipe** (https://docs.expo.dev/guides/google-authentication/): the page no longer contains an inline `useAuthRequest`/client-ID/scope code recipe — SDK 56 redirects readers to the external `@react-native-google-signin/google-signin` library docs. Captured what the guide does cover (dev build requirement, Firebase/Google Cloud setup, SHA-1 fingerprints).
- **Apple AuthSession recipe**: the general Authentication guide ships worked examples for **GitHub and Okta only**; Apple is covered via `expo-apple-authentication` (section 6), not as a raw AuthSession recipe.
