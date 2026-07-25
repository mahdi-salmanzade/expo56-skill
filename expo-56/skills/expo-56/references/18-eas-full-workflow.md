# EAS (Expo Application Services) — Full Workflow Reference (SDK 56)

Official Expo SDK 56 documentation covering the end-to-end EAS workflow: Build configuration, internal distribution, app credentials, Submit, Update (channels/branches/migration), Workflows (CI/CD), Metadata, and environment variables.

> Scope note: This document complements the build-performance reference already captured. It focuses on configuration fields, commands, and verbatim examples across the EAS surface.
>
> **EAS docs are not SDK-versioned.** `eas.json`, EAS Submit, EAS Update, EAS Workflows and EAS Metadata are unversioned tooling — there is no `v56`/`v57` split for them, and a change to EAS CLI reaches SDK 56 and SDK 57 users on the same day. What *is* SDK-scoped: the **build images** (`sdk-56` vs `sdk-57`), the **project template tag** (`default@sdk-56`), **dependency pins**, and — because EAS shells out to your project's own Expo CLI — the **behaviour of `npx expo prebuild`**, which changed in SDK 57 and therefore changes what `prebuildCommand` (§1) and `eas/prebuild` (§11, §12) actually do. Those are collected in the trailing `SDK 57 delta` section.

Related references: `05` (Expo CLI commands), `08` (expo-updates client API), `10` (build performance / precompiled modules), `16` (config plugins).

### What a model gets wrong from memory

| Trap | Reality |
| --- | --- |
| `eas env:create` / `eas env:update` | Neither exists. `eas env:set` covers both create and update (§9). |
| Workflows live in `.github/workflows/` | They live in `.eas/workflows/`, next to `eas.json`. `.eas/build/` is a *different* DSL (custom builds, §12). |
| GitHub Actions syntax | EAS Workflows uses `runs_on` (underscore), `type:` + `params:` for pre-packaged jobs, and `uses:` + `with:` only inside `steps:`. Not interchangeable with Actions. |
| `eas build` picks a sensible profile | Omitting `--profile` uses `production`. |
| Channels hold updates | Channels are attached to **builds**; **branches** hold updates; a channel points at a branch. |
| `.env` always reaches the build server | Only if it is **not** in `.gitignore`/`.easignore` — then it ships in the project archive and `eas build` / `eas update` pick it up. The recommended setup gitignores it, in which case remote jobs see only EAS environment variables and profile/job `env`. |
| Distribution certificate limit is 1 | The docs contradict themselves: the prose asserts one per Apple Developer account, the summary table says **2**. Treat 2 as the limit, one-and-reuse as the practical recommendation (§3). |

---

## 1. EAS Build configuration — `eas.json`

Sources:
- https://docs.expo.dev/build/eas-json/
- https://docs.expo.dev/build-reference/build-configuration/

### File location & purpose
`eas.json` sits alongside `package.json` at the project root and configures EAS CLI and services. All EAS Build configuration goes under the `build` key.

### Build profiles
Build profiles are named configuration groups describing parameters for a specific build type. The default configuration includes three profiles — `development`, `preview`, and `production` — though custom names are allowed.

Default structure:
```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

- Run a build: `eas build --profile <profile-name>`
- Omitting `--profile` defaults to `production`.

### Profile extension (inheritance)
Profiles can inherit configuration with `extends`:
```json
{ "extends": "production" }
```
Chains up to 5 levels deep are supported; avoid circular dependencies.

### Platform-specific configuration
Each profile supports `android` and `ios` objects for platform-specific settings. Common options work at both the root profile level and the platform-specific level; platform-specific values take precedence.

**Expo projects** require `android.package` and `ios.bundleIdentifier` in `app.json`. **Bare React Native** projects need no additional config steps.

### Profile type behaviors
- **Development**: includes `"developmentClient": true` and `"distribution": "internal"`. Supports iOS Simulator via `"ios": { "simulator": true }`.
- **Preview**: no developer tools; for team testing. Uses internal distribution or non-store packaging (APK for Android).
- **Production**: store-submitted builds. Cannot install directly on emulators/devices except Android APK (AAB recommended).

### Build tool versions
Specify tool versions by field name:
```json
{
  "build": {
    "production": {
      "node": "18.18.0"
    }
  }
}
```
Supported tools include Node.js, npm, Yarn, Ruby, Bundler, CocoaPods, Fastlane, Xcode, and Android NDK.

### Resource class
Configure the build VM with `resourceClass`. The schema enum is `"default" | "medium" | "large"`; the effective default is `medium`, and `large` is not available on the free plan (`docs/public/static/schemas/unversioned/eas-json-build-common-schema.js:65-73`). Set per platform:
```json
{
  "android": { "resourceClass": "medium" },
  "ios": { "resourceClass": "large" }
}
```

### Base image selection
Controls default dependency versions (Node.js, Yarn, CocoaPods, OS version, Xcode). EAS automatically selects the appropriate image based on the SDK version for Expo projects. Aliases: `auto` (resolves from project config + SDK + RN version), `latest` (most up-to-date software), and per-SDK tags `sdk-57`, `sdk-56`, `sdk-55`, `sdk-54`, `sdk-53`, `sdk-52` (`docs/pages/build-reference/infrastructure.mdx:23-35`). Omitting `image` entirely uses `auto`. Check the **Spin up build environment** log section to see which image a build actually used. Concrete image names for SDK 56 vs SDK 57 are in the SDK 57 delta section.

### Common build-profile options (schema)
From `docs/public/static/schemas/unversioned/eas-json-build-common-schema.js`: `withoutCredentials`, `extends`, `credentialsSource`, `releaseChannel`, `channel`, `distribution`, `developmentClient`, `resourceClass`, `prebuildCommand`, `buildArtifactPaths`, `node`, `corepack`, `yarn`, `pnpm`, `bun`, `expoCli`, `env`, `autoIncrement`, `cache` (`disabled` / `key` / `paths`), `config`, `environment`.

- `environment` — enum `development | preview | production`; selects which EAS environment's variables are applied (§9).
- `config` — points at a **custom build config** file in `.eas/build/<name>.yml` (e.g. `"config": "production.yml"`). This is a different DSL from EAS Workflows — see §12.

### App version source (`cli.appVersionSource`)
The single most common `eas.json` versioning gotcha. `"remote"` is the recommended behavior from EAS CLI 12.0.0 onward: EAS stores and increments `android.versionCode` / `ios.buildNumber` server-side. Pair it with `"autoIncrement": true` on the production profile.
```json
{
  "cli": { "appVersionSource": "remote" },
  "build": { "production": { "autoIncrement": true } }
}
```
`"local"` means the app config in your repo is the source of truth and EAS never writes versions back.

Caveats (`docs/pages/build-reference/app-versions.mdx:110-113`): `eas build:version:sync` on Android does not support existing RN projects with multiple flavors; `autoIncrement` does not support the `version` option; remote versioning is not supported with `"runtimeVersion": { "policy": "nativeVersion" }` — use the `appVersion` policy instead.

### Environment variables (inline `env`)
Set with the `env` field; available during `app.config.js` evaluation and on the build servers:
```json
{
  "build": {
    "production": {
      "env": { "API_URL": "https://company.com/api" }
    }
  }
}
```
(See §9 for the managed environment-variable system.)

### Top-level schema structure
```json
{
  "cli": {
    "version": "SEMVER_RANGE",
    "requireCommit": boolean,
    "appVersionSource": string,
    "promptToConfigurePushNotifications": boolean
  },
  "build": {
    "PROFILE_NAME": {
      "android": { /* ... */ },
      "ios": { /* ... */ }
    }
  }
}
```

### Configuration process (`eas build:configure`)
When running `eas build:configure` or `eas build` on an unconfigured project, EAS CLI:
1. Prompts you to choose which platform(s) to configure.
2. Generates `eas.json` in the project root.
3. Performs project-specific setup (varies by project type).

If `cli.requireCommit` is `true`, you are prompted to commit changes with an optional custom git message.

---

## 2. Internal distribution

Source: https://docs.expo.dev/build/internal-distribution/

### Enable
Set `"distribution": "internal"` in your `eas.json` build profile.

### Platform behavior
- **Android**: by default the build generates an APK instead of an AAB. EAS Build generates a new Android keystore for signing the APK, or uses an existing one if the package name matches your development build.
- **iOS**: builds use either ad hoc or enterprise provisioning. With ad hoc, EAS Build generates a provisioning profile containing an allow-list of device UDIDs, restricting installation to registered devices only.

### Access control
Build URLs are available to anybody with the URL, each identified by a 32-character UUID. Require authentication by disabling the "Unauthenticated access to internal builds" option in project settings.

### Device management commands
- `eas device:create` — register a new device
- `eas device:list` — view registered devices
- `eas device:delete` — remove a device (optionally disable it on the Apple Developer Portal)
- `eas device:rename` — assign friendly names

### Automation on CI
`eas build --non-interactive` reuses a valid ad hoc provisioning profile **without updating its device list** — the build can still succeed, but the app may not install on devices registered after the profile was last updated.

There is a documented non-interactive path for refreshing it (`docs/pages/build/internal-distribution.mdx:31-55`):
```sh
eas build --platform ios --profile preview --non-interactive --refresh-ad-hoc-provisioning-profile
```
Requires **EAS CLI 19.1.0+**, plus:
- `"distribution": "internal"` and EAS-managed credentials on the profile.
- At least one device from `eas device:create`.
- An App Store Connect API key in CI via `EXPO_ASC_API_KEY_PATH`, `EXPO_ASC_KEY_ID`, `EXPO_ASC_ISSUER_ID` — or a key stored on EAS for the project.

EAS reads the devices registered on EAS for your Apple team, registers any missing UDIDs on the portal, and refreshes the profile's device list (selecting all matching devices for the target's Apple platform). The EAS Workflows equivalent is `refresh_ad_hoc_provisioning_profile: true` in the `build` job's `params`.

Without the flag, the fallback is still an interactive `eas build` with Apple sign-in.

> **Warning (new/recently renewed memberships)**: registering a device with Expo does not register it with Apple — that happens the first time it is included in a provisioning profile. Apple can take 24–72 h to finish processing a newly registered device, so the first build or `eas build:resign` that includes it may fail. Wait, then rebuild/re-sign.

With EAS Workflows you can pause a run until a tester enrolls a device using the `apple-device-registration-request` job, then pair it with a `build` job that sets `refresh_ad_hoc_provisioning_profile: true`.

### Constraints
- Ad hoc distribution is subject to Apple's limit of 100 devices annually per app.
- Enterprise distribution requires membership in the Apple Developer Enterprise Program.

---

## 3. App credentials & signing

Source: https://docs.expo.dev/app-signing/app-credentials/

### Android credentials (keystore)
Android apps require signing with a certificate stored in a keystore. Google offers two approaches:
- **Upload Certificate Method (recommended)**: upload an APK signed with an upload certificate; Google Play replaces it with the app signing certificate. If the upload keystore is lost, Google Play support can reset the key.
- **App Signing Certificate Method (legacy)**: direct signing with the app signing certificate; no recovery if lost.

Management:
- Run `eas credentials` to download your keystore via `credentials.json`.
- > "Your application's keystore should be kept private. Under no circumstances should you check it into your repository." (Debug keystores are the exception.)
- Syncing with Google Play: export the keystore to PEM with `keytool` using the `keyAlias` from `credentials.json`, then contact Google Support to update your account with the new upload key.

### iOS credentials
1. **Distribution Certificate** — not app-specific; reused across all your apps. **The docs contradict themselves here**: the prose states "You may only have one distribution certificate associated with your Apple Developer account" (`docs/pages/app-signing/app-credentials.mdx:79`), while the summary table lists a Limit Per Account of **2** (`:96-102`). Treat 2 as the limit and one-and-reuse as the practical recommendation. Expiration doesn't affect production apps but prevents new uploads/updates to the App Store.
2. **Push Notification Keys (APN)** — max 2 per account; usable across multiple apps. Revocation stops push notifications until replaced. Keys don't expire; tokens remain valid when replaced.
3. **Provisioning Profiles** — app-specific; one per App Store submission. Expire after 12 months but don't affect production apps. Associated with the distribution certificate.

### EAS credential management
- `eas credentials` — inspect, download, and manage all credentials (select platform and profile).
- `eas build:resign` — re-sign existing iOS builds with new provisioning profiles, reusing the application artifact without a full rebuild (useful for adding test devices).
- Deleting credentials via EAS removes them only from Expo's servers, not Apple's. Full deletion requires the Apple Developer Console.

---

## 4. EAS Submit

Sources (the old `/submit/introduction/` page was deleted and now 301s to the first entry — `docs/pages/submit/index.js`):
- https://docs.expo.dev/deploy/submit-to-app-stores/ — overview
- https://docs.expo.dev/submit/android/ , https://docs.expo.dev/submit/ios/
- https://docs.expo.dev/submit/android-manual/ , https://docs.expo.dev/submit/ios-manual/
- https://docs.expo.dev/submit/testflight/
- https://docs.expo.dev/submit/eas-json/

### What it does
> "EAS Submit is a hosted service for submitting Android and iOS app binaries to the Google Play Store and Apple App Store from the command line."

- Automates the final distribution step without manual uploads via Google Play Console or Transporter.
- Accepts binaries from EAS Build or local builds (`.aab` and `.ipa`).
- Enables iOS uploads from Windows and Linux.
- Supports multiple submission profiles and CI/CD integration.

### Commands
```sh
eas submit --platform android
eas submit --platform ios
eas build --platform ios --auto-submit
```

Local builds:
```sh
eas submit --platform android --path ./my-app.aab
eas submit --platform ios --path ./my-app.ipa
```

CI environments:
```sh
eas submit --platform android --latest --non-interactive
```

### Platform behavior
- **Android (Google Play)**: uploads to a specified track (`internal`, `alpha`, `beta`, `production`). Non-production tracks limit availability to testers.
  - **First submission does *not* require a manual upload.** For a brand-new app the default `eas submit` works out of the box and creates the first release on the **internal testing track**. The app stays in *draft* status in Play Console until you complete the store listing and setup tasks, which are required before promoting to production. (`docs/pages/submit/android.mdx:86-88`)
  - Prerequisites still apply: the app must exist in Play Console and EAS needs a Google Service Account key.
  - Prefer doing the first upload yourself? `/submit/android-manual/`. Want to upload without rolling out? Set `"releaseStatus": "draft"` in the submit profile and finish the release in Play Console.
  - The default `production` build profile produces an **.aab**; a profile only produces an **.apk** when it sets `android.buildType: "apk"`, which cannot be submitted to Play.
- **iOS (App Store Connect)**: uploads to App Store Connect; appears in TestFlight. Does not auto-release to production — manual ASC steps required for App Store release.

### Prerequisites
- Android: a Google Service Account Key with Play Console access.
- iOS: an Apple Developer account, `ascAppId`, and credentials or an App Store Connect API Key.
- Correctly signed binaries (release keystore for Android; distribution certificate + provisioning profile for iOS).

### Limitations
EAS Submit does not handle metadata, screenshots, or release-notes management (see EAS Metadata, §8).

### `submit` config in `eas.json`
```json
{
  "cli": {
    "version": ">= 0.34.0"
  },
  "submit": {
    "production": {
      "android": {
        "track": "internal"
      },
      "ios": {
        "ascAppId": "your-app-store-connect-app-id"
      }
    }
  }
}
```
- Running `eas submit` without a profile uses `production` if defined, or prompts interactively if values are absent.
- The `submit` object supports arbitrary named profiles.
- Profiles can extend others: `"extends": "SUBMIT_PROFILE_NAME"` (up to 5 levels of chaining; no circular dependencies).
- Android fields (complete, `docs/public/static/schemas/unversioned/eas-json-submit-android-schema.js`): `serviceAccountKeyPath`, `track`, `releaseStatus`, `rollout`, `changesNotSentForReview`, `applicationId`.
- iOS fields (complete, `.../eas-json-submit-ios-schema.js`): `appleId`, `ascAppId`, `appleTeamId`, `sku`, `language`, `companyName`, `appName`, `ascApiKeyPath`, `ascApiKeyIssuerId`, `ascApiKeyId`, `bundleIdentifier`, `metadataPath`, `groups`.
- See the EAS Submit schema reference (`/eas/json#eas-submit`) for descriptions.

---

## 5. EAS Update — how it works (channels & branches)

Source: https://docs.expo.dev/eas-update/how-it-works/

### Channels
Channels are naming conventions for builds, defined in `eas.json`. A single channel can identify multiple builds across platforms (e.g. `production` and `staging` channels organize builds for public stores vs. internal testing tracks).

### Branches
Branches store ordered lists of updates on EAS servers, similar to Git branches. Publishing via `eas update --auto` uploads a bundle to a branch and makes it the active update on the branch. The most recent update on a branch is what gets distributed.

### Channel–branch relationship
The link between channels and branches determines which updates reach which builds. By default, channels auto-link to branches sharing the same name. The relationship is configurable:
```sh
eas channel:edit production --branch version-2.0
```

### Runtime versions
Runtime versions describe the JS–native interface defined by the native code layer. They must match exactly between builds and updates for compatibility. Native code changes that affect this interface require a runtime version bump in the app config.

> **Patch-drift warning — `appVersion` policy, `@expo/config-plugins` 56.0.12 (2026-07-07, #47416).** Landed *after* the 56.0.5 snapshot this file was first written against, and also in 57.0.3. `Updates.getAppVersion`, `Updates.getNativeVersion`, and the `appVersion` runtime-version policy now honour `ios.version` / `android.version`. Previously the platform-specific overrides were ignored and these helpers fell back to `package.json` (or `"1.0.0"`).
> - **Impact**: a project that sets only `ios.version` / `android.version` with no top-level `version` gets a **different runtime version** after upgrading past 56.0.12 — previously published updates will no longer match existing builds. Republish, or add a top-level `version` matching the old fallback.
> - `Updates.getAppVersion` gained an optional `platform` argument; calls without it keep the previous behavior.
> - Android `getNativeVersion` previously used the **iOS** version for the `${version}` component; that is now fixed.
> Source: `git show origin/sdk-56:packages/@expo/config-plugins/CHANGELOG.md`.

### Update process flow
1. **Build creation** — `eas build` packages native code with channel, runtime version, and platform metadata.
2. **Update publishing** — `eas update --auto` creates local bundles and uploads them to EAS branches.
3. **Matching** — updates distribute to builds with matching platforms, runtime versions, and linked channels.
4. **Download phases** — `expo-updates` first fetches manifests, then required assets before `fallbackToCacheTimeout`.

---

## 6. EAS Update — deployment patterns

Source: https://docs.expo.dev/eas-update/deployment-patterns/

### Patterns
- **Two-Command Flow (simplest)** — single production build; test in Expo Go or development builds; publish to one branch:
  ```sh
  eas update --branch production
  ```
  Fastest delivery, minimal safety checks.
- **Persistent Staging Flow (environment-based)** — separate staging and production builds; test on TestFlight / Play Store Internal Track; two persistent branches (`staging`, `production`); uses `expo-github-action` to publish updates on merge. Lets you control the pace of deploying to production independent of the pace of development.
- **Platform-Specific Flow** — platform-based branches like `ios-staging`, `ios-production`, `android-staging`, `android-production`; separate commands per platform; full control over which updates apply to each platform.
- **Branch Promotion Flow (version-managed)** — version-based branches like `version-1.0`, `version-2.0`; dynamically maps branches to channels via the website or CLI to point a channel at an EAS Update branch; requires manual runtime version specification (cannot use automatic policies); preserves historical versions on GitHub.

### Key operations
- Channel-to-branch mapping via web interface or CLI (`eas channel:edit <channel> --branch <branch>`).
- Merging into GitHub branches triggers automated updates.
- Promote tested updates by redirecting channels to different branches.
- Runtime version management for compatibility tracking.

### Rollouts
Source: https://docs.expo.dev/eas-update/rollouts/ (`docs/pages/eas-update/rollouts.mdx:12-71`). Two independent mechanisms.

**Per-update rollouts** — a percentage of users receive a newly published update:
```sh
eas update --rollout-percentage=10   # publish to 10% of end users
eas update:edit                      # progress: pick the update, set a new percentage
eas update:revert-update-rollout     # revert to the previous state
```
- Only one update can be rolled out on a branch at a time.
- A rollout **must be ended** (progressed to 100 or reverted) before a new update on the same runtime version can be published — this prevents clobbering it.
- Inspect with `eas update:list` / `eas update:view`.
- Reverting on a branch that already had an update republishes the control update; reverting a rollout started on an empty branch creates a rollback-to-embedded update.

**Branch-based rollouts** — split a channel's traffic across two branches:
```sh
eas channel:rollout   # interactive: select channel, branch, percentage; re-run to Edit or End
```
- Only one branch rollout per channel at a time.
- Ending offers *Republish and revert* (keep the new branch's work) or *Revert* (discard it).
- While a rollout is in progress, `eas update --branch <branch>` still works but **`eas update --channel <channel>` is rejected** — EAS cannot tell which of the two branches to associate the update with.

These are what the Workflows `update` (`rollout_percentage`) and `update-rollout` job types drive (§11).

---

## 7. EAS Update — migrate from CodePush

Sources (`/eas-update/migrate-to-eas-update/` no longer exists — it 301s to the second entry, `docs/public/_redirects:180`):
- https://docs.expo.dev/eas-update/codepush/ (`docs/pages/eas-update/codepush.mdx`)
- Related: https://docs.expo.dev/eas-update/migrate-from-classic-updates/

### Conceptual difference
With CodePush, the client controls the target update deployment at runtime. With EAS Update, the default path is controlled server-side by mapping channels to branches.

### Migration steps
1. **Uninstall CodePush**: `npm uninstall react-native-code-push`, and remove all CodePush references from JS and native code.
2. **Prepare the project**: ensure `app.json` contains an `expo` object:
   ```json
   { "expo": {} }
   ```
3. **Follow the EAS Update "Get started" guide** (https://docs.expo.dev/eas-update/getting-started/) for installation (`expo-updates`), `eas update:configure`, runtime version, and `updates.url` setup.
4. **Rebuild and resubmit** your app to the stores, since the update provider changed.

### API mapping (CodePush → EAS Update)
- Mandatory updates: implement with logic built on EAS Update (no dedicated flag).
- Update messages: use the `extra` field in app config.
- Runtime deployment switching: `Updates.setUpdateURLAndRequestHeadersOverride()`.
- Gradual rollouts: use EAS Update's rollout strategies (§6, Rollouts).
- Direct update control: `Updates.checkForUpdateAsync()`.

Recommendation: use the latest Expo SDK version — migration instructions are not available for older Expo SDK / React Native versions.

---

## 8. EAS Metadata

Source: https://docs.expo.dev/eas/metadata/

> **EAS Metadata is in [beta](/more/release-statuses/#beta) and subject to breaking changes.** (`docs/pages/eas/metadata/index.mdx:15`)

### Overview
A command-line tool that automates app store presence management, letting you provide store information via configuration files instead of multiple dashboard forms.

### Capabilities
- Automate app store information management.
- Validate metadata before submission to prevent rejections (instant feedback before any review).
- Maintain store presence from the CLI without leaving the project.

### Commands
```sh
eas metadata:push   # push configuration to the app stores
eas metadata:pull   # pull current configuration from the app stores
```

### Configuration
Uses a `store.config.json` file to centralize app store information. Supports:
- **Static** store config (JSON files).
- **Dynamic** store config (asynchronous functions for external data sources).

### Current limitations
- Google Play Store management is not yet implemented.
- Screenshot uploads are not supported.
- Works best with unrestricted app store accounts.

### IDE support
The Expo Tools VS Code extension provides auto-complete, suggestions, and warnings for `store.config.json`.

---

## 9. EAS environment variables

Sources (this page was split into a five-page section):
- https://docs.expo.dev/eas/environment-variables/ — overview & visibility
- https://docs.expo.dev/eas/environment-variables/manage/ — CLI commands
- https://docs.expo.dev/eas/environment-variables/usage/ — Build/Update/Workflows usage, built-in variables
- https://docs.expo.dev/eas/environment-variables/without-eas/
- https://docs.expo.dev/eas/environment-variables/faq/

### Variable types
- **Strings** — standard key/value pairs, usable across builds, updates, workflows, and hosting.
- **Files** — values uploaded as files (e.g. `google-services.json`, a certificate) exposed to jobs as file paths on the runner.

### Visibility levels
(`docs/pages/eas/environment-variables/index.mdx:119-123`)
- **Plain text** — visible on the website, in EAS CLI, and in logs.
- **Sensitive** — obfuscated in EAS Build and Workflows job logs; toggle to reveal on the website; still readable in EAS CLI.
- **Secret** — not readable outside the EAS servers, including on the website and in EAS CLI. **Also obfuscated in Build/Workflows job logs** — do not expect to debug a secret's value by echoing it.

Secret variables are also unavailable during build-configuration resolution in EAS CLI (they are not readable off the EAS servers).

### Environments
Three by default: `development`, `preview`, `production`. Custom names are available on Enterprise and Production plans.

### Commands & usage

Create **or update** a variable — `eas env:set` covers both; there is no `env:create` or `env:update`:
```sh
eas env:set --name EXPO_PUBLIC_API_URL --value https://api.example.com \
  --environment production --visibility plaintext
```

Use with Build — set the environment in `eas.json`:
```json
{ "build": { "production": { "environment": "production" } } }
```

Use with Updates:
```sh
eas update --environment production
```

Use with Hosting:
```sh
eas env:pull --environment production
npx expo export --platform web
eas deploy --environment production
```

Full documented command set (`docs/pages/eas/environment-variables/manage.mdx:163-180`):
```sh
eas env:set --name NAME --value VALUE --environment production --visibility plaintext
eas env:list --environment production
eas env:pull --environment production   # writes a local .env
eas env:delete
```

> From **SDK 55 or later**, `--environment` is **required** when running `eas update`. On SDK 54 and earlier, `eas update` falls back to local `.env` files when the flag is omitted. (`docs/pages/eas/environment-variables/usage.mdx`)

### Built-in (system) environment variables
Exposed to every EAS Build/Workflows job; **not** part of any project environment and **not** available when evaluating `app.config.js` locally (`docs/pages/eas/environment-variables/usage.mdx:60-72`):

| Variable | Value |
| --- | --- |
| `CI` | `1` |
| `EAS_BUILD` | `true` |
| `EAS_BUILD_PLATFORM` | `android` \| `ios` |
| `EAS_BUILD_RUNNER` | `eas-build` (cloud) \| `local-build-plugin` (local) |
| `EAS_BUILD_ID` | Build UUID |
| `EAS_BUILD_PROFILE` | Profile name from `eas.json` |
| `EAS_BUILD_PROJECT_ID` | EAS project UUID |
| `EAS_BUILD_GIT_COMMIT_HASH` | Commit hash |
| `EAS_BUILD_NPM_CACHE_URL` | npm cache URL |
| `EAS_BUILD_MAVEN_CACHE_URL` | Maven cache URL |
| `EAS_BUILD_COCOAPODS_CACHE_URL` | CocoaPods cache URL |
| `EAS_BUILD_USERNAME` | Initiating user (undefined for bot users) |
| `EAS_BUILD_WORKINGDIR` | Remote project directory path |
| `EAS_BUILD_DISABLE_BUNDLE_JAVASCRIPT_STEP` | Set to `1` to skip the pre-native JS bundling check |

> **Do not prefix your own variables with `EAS_`** — they can shadow or be shadowed by these (`docs/pages/eas/workflows/environment.mdx:36`).

### Notes
- Local `.env` files reach a remote EAS job **only if they are not listed in `.gitignore` / `.easignore`** — they travel with the uploaded project archive and are then picked up by `eas build`, `eas update`, and so on (`docs/pages/eas/environment-variables/without-eas.mdx:21`). Because the recommended setup gitignores them, the practical default is that remote jobs see only EAS environment variables and profile/job `env`.
- "Anything that is included in your client-side code should be considered public." The `EXPO_PUBLIC_` prefix exposes a variable to client-side code.
- Secret variables don't secure values embedded in the application itself.
- Scopes: variables can be project-wide (a single EAS project) or account-wide (across all projects).

---

## 10. EAS Workflows — get started

Source: https://docs.expo.dev/eas/workflows/get-started/

### Overview
EAS Workflows automate React Native CI/CD development and release processes using YAML configuration files triggered by various events.

### Project setup
1. Create an Expo account and install EAS CLI (`npm install -g eas-cli`).
2. Initialize a project — the template tag is the one SDK-scoped step here:
   ```sh
   npx create-expo-app@latest --template default@sdk-56   # SDK 57: default@sdk-57
   ```
   (yarn / pnpm / bun variants: `yarn create expo-app --template default@sdk-56`, etc.)
3. Link to EAS: `npx eas-cli@latest init`
4. Add `eas.json` to the project root.

### Scaffolding a workflow (`eas workflow:create`)
Prefer this over hand-writing the directory and file (`docs/pages/eas/workflows/get-started.mdx:45,79`):
```sh
eas workflow:create --template build    # -> .eas/workflows/build.yml (development builds; configures the project)
eas workflow:create --template deploy   # -> .eas/workflows/deploy.yml (fingerprint -> build+submit or update)
eas workflow:create                     # prompts
```
The `deploy` template configures EAS Build and EAS Update and sets app identifiers, then writes a release workflow that fingerprints the project and either builds+submits (native changes) or publishes an update (matching build already exists).

For production builds set up signing first: `eas credentials:configure-build -p android -e production` (and `-p ios`).

### Directory structure
Create `.eas/workflows/` at the project root containing YAML workflow files (e.g. `create-production-builds.yml`). The `.eas` directory sits at the same level as `eas.json`. Files must use `.yml` or `.yaml` and be **16 KiB or smaller**.

### Basic workflow example
```yaml
name: Create Production Builds

jobs:
  build_android:
    type: build
    params:
      platform: android
  build_ios:
    type: build
    params:
      platform: ios
```
This builds both platforms in parallel.

### Run a workflow
```sh
eas workflow:run .eas/workflows/create-production-builds.yml
eas workflow:run create-production-builds.yml                  # bare filename also works
eas workflow:run .eas/workflows/deploy.yml -F environment=production -F debug=true
echo '{"environment":"production"}' | eas workflow:run .eas/workflows/deploy.yml
```
Missing required `workflow_dispatch` inputs are prompted for unless `--non-interactive`.

Install the resulting builds with `eas build:run -p android --latest` / `eas build:run -p ios -e development-ios-simulator --latest`.

### Triggers (high level)
- GitHub events via `on.push`, `on.pull_request`, `on.ref_delete`.
- App Store Connect events via `on.app_store_connect` (e.g. `ready_for_review`).

### Tooling
The Expo Tools VS Code extension provides descriptions and autocompletions for workflow files.

### Other Workflows pages worth knowing
- `/eas/workflows/environment/` — job environment resolution, variable precedence, every interpolation context.
- `/eas/workflows/pre-packaged-jobs/` — the long-form per-job reference the syntax page defers to.
- `/eas/workflows/rest-api/` — `POST https://api.expo.dev/v2/workflows/dispatch` with `Authorization: Bearer <EXPO_TOKEN>` (robot-user token recommended for production integrations).
- `/eas/workflows/automating-eas-cli/`, `/eas/workflows/troubleshooting/`
- `/eas/workflows/limitations/` — no shared workflow configs; no matrix builds.
- `/eas/workflows/examples/*` — branch-cleanup, create-development-builds, deploy-to-production, e2e-tests, publish-preview-update.

---

## 11. EAS Workflows — syntax reference

Source: https://docs.expo.dev/eas/workflows/syntax/

### Top-level keys
Workflow YAML files live in `.eas/workflows/` alongside `eas.json`; `.yml`/`.yaml` only, 16 KiB max (`docs/pages/eas/workflows/syntax.mdx:22`).
- `name` — human-friendly identifier shown on the EAS dashboard.
- `on` — GitHub/ASC events that trigger the workflow (can also be run manually via `eas workflow:run`).
- `jobs` — the tasks comprising a run.
- `defaults` — global parameters applied to all jobs (`hooks`, `image`, `run.working_directory`, `tools`).
- `concurrency` — controls execution when multiple runs occur simultaneously.

### Event triggers (`on`)
1. **`push`** — `branches` (globs, exclude with `!`), `tags`, `paths`.
2. **`pull_request`** — `branches`, `types`, `paths`. Full `types` set: `opened`, `edited`, `base_ref_changed`, `ready_for_review`, `reopened`, `synchronize`, `labeled`. Defaults to `['opened','reopened','synchronize']`. `edited` follows GitHub's `pull_request.edited` (title, body, or base branch); use `base_ref_changed` for base-branch-only. `branches` matches the PR's *current* base branch, so retargeting a PR can trigger a workflow.
3. **`pull_request_labeled`** — `labels`.
4. **`ref_delete`** — fires when a GitHub branch or tag is deleted. Fields `branches`, `tags` (globs + `!` negation). When neither is given, `branches` defaults to `['*']` and `tags` to `[]`; if only one is given the other defaults to `[]`. **Workflow files are read from the default branch HEAD at deletion time**, not from the deleted ref. Pairs with the `branch-delete` job.
5. **`app_store_connect`** — `app_version.states`, `build_upload.states`, `external_beta.states`, `beta_feedback.types` (`crash`, `screenshot`).
6. **`schedule`** — a **list** of `- cron: '...'` entries (Unix-cron, GMT, default branch only). A workflow may have multiple crons; runs may be delayed under load, so keep them idempotent.
   ```yaml
   on:
     schedule:
       - cron: '0 0 * * *'   # midnight GMT daily
   ```
7. **`workflow_dispatch`** — `inputs` accepting `string`, `boolean`, `number`, `choice`, `environment` types (`type` required; `description`, `required` (default `false`), `default`, and `options` (required for `choice`)).

> Skip a `push`/`pull_request` run by putting `[eas skip]`, `[skip eas]`, or `[no eas]` in the commit message.

### Job definition
```yaml
jobs:
  job_id:
    name: Human-friendly job title
    environment: production | preview | development   # default is job-type dependent — see below
    env:
      VAR_NAME: value
    hooks:                  # job-type specific; see the job's entry
      after_checkout: []
    outputs:                # values exposed to downstream jobs
      my_output: ${{ steps.step_1.outputs.value }}
    needs: [other_job_id]   # runs only if dependencies succeed
    after: [other_job_id]   # runs after dependencies complete (regardless of status)
    if: ${{ conditional_expression }}
```

**`environment` default is not always `production`** (`docs/pages/eas/workflows/syntax.mdx:448-465`, `environment.mdx:38-50`):

| Job type | Default environment |
| --- | --- |
| `build` | inferred from the build profile's `environment` in `eas.json` |
| `submit` | inherited from the submitted build |
| `maestro`, `maestro-cloud` | `preview` |
| everything else | `production` |

Variable precedence in a job, highest first: job `env` → `eas.json` build-profile `env` (for `build` jobs, from `params.profile`) → EAS environment variables for the job's environment.

**`env` is unavailable** on the pre-packaged jobs that do not run a VM: `apple-device-registration-request`, `branch-delete`, `doc`, `get-build`, `github-comment`, `require-approval`, `slack`, `update-rollout`.

**`hooks`** are supported by `build`, `deploy`, `fingerprint`, `maestro`, `maestro-cloud`, `repack`, `submit`, `testflight`, `update`. Hook keys are job-specific (`after_checkout`, `before_install_node_modules`, `after_install_node_modules`, `before_submit`/`after_submit`, `before_update`/`after_update`, `before_maestro_tests`/`after_maestro_tests`, `before_maestro_cloud`/`after_maestro_cloud`). A job-level key overrides `defaults.hooks`; set it to `[]` to opt that job out.

### Pre-packaged job types (`type`)
Long-form reference: `/eas/workflows/pre-packaged-jobs/`. Params below are complete per `docs/pages/eas/workflows/syntax.mdx:1146-1634`.

```yaml
build:                                  # runs_on + image supported
  platform: ios | android               # required
  profile: string                       # default: production
  message: string
  refresh_ad_hoc_provisioning_profile: boolean
# outputs: build_id, app_build_version, app_identifier, app_version, channel,
#          distribution, fingerprint_hash, git_commit_hash, platform, profile,
#          runtime_version, sdk_version, simulator

deploy:                                 # EAS Hosting
  alias: string
  prod: boolean
  source_maps: boolean
# outputs: deploy_json, deploy_url, deploy_alias_url, deploy_deployment_url,
#          deploy_identifier, deploy_dashboard_url

fingerprint:                            # no params; set `environment` to match your build profile
# outputs: android_fingerprint_hash, ios_fingerprint_hash

get-build:                              # find an existing build
  platform, profile, distribution, channel, app_identifier, app_build_version,
  app_version, git_commit_hash, fingerprint_hash, sdk_version, runtime_version,
  simulator, wait_for_in_progress       # all optional; outputs match `build`

submit:
  build_id: string                      # required
  profile: string                       # default: production
  groups: string[]
# outputs: apple_app_id, ios_bundle_identifier, android_package_id

testflight:                             # provide exactly one of build_id / asc_build_id
  build_id: string                      # upload + submit
  asc_build_id: string                  # submit an already-uploaded build
  profile: string                       # default: production (upload path only)
  wait_processing_timeout_seconds: number  # default 1800
  internal_groups: string[]
  external_groups: string[]
  changelog: string
  submit_beta_review: boolean
# outputs: apple_app_id + ios_bundle_identifier (build_id path) | asc_build_id (asc path)

update:
  message: string
  platform: android | ios | all         # default all
  branch: string
  channel: string                       # cannot be combined with branch
  rollout_percentage: number            # 0-100, default 100
  private_key_path: string
  upload_sentry_sourcemaps: boolean     # default: try, don't fail the job
# outputs: first_update_group_id, updates_json

update-rollout:                         # advance an in-progress rollout
  update_group_id: string               # required
  rollout_percentage: number            # 0-100, default 100
# outputs: update_group_id, rollout_percentage, updates_json

branch-delete:                          # delete an EAS Update branch and its updates
  branch_name: string                   # required
  fail_on_missing: boolean              # default false
# outputs: branch_id, branch_name

maestro:                                # alpha. runs_on + image supported
  build_id: string                      # required
  flow_path: string | string[]          # required
  shards: number                        # default 1
  retries: number                       # default 0
  retry_failed_only: boolean            # default true
  record_screen: boolean                # default false
  include_tags / exclude_tags: string | string[]
  maestro_version: string
  android_system_image_package: string  # use a linux-*-nested-virtualization worker
  device_identifier: string | { android?, ios? }
  output_format: string                 # default junit
  skip_build_check: boolean             # default false

maestro-cloud:                          # requires a Maestro Cloud subscription
  build_id: string                      # required
  maestro_project_id: string            # required
  flows: string                         # required
  maestro_api_key: string               # default: MAESTRO_CLOUD_API_KEY env var
  maestro_config, maestro_version, include_tags, exclude_tags,
  device_locale, device_model, device_os, skip_build_check,
  name, branch, async

slack:
  webhook_url: string                   # required
  message: string                       # required if payload absent
  payload: object                       # required if message absent

github-comment:
  message: string
  build_ids / update_group_ids / deployment_ids: string[]   # default: all from this run
  payload: string                       # raw markdown/HTML, fully overrides the comment
# outputs: comment_url

apple-device-registration-request:      # pause until a device enrolls and is approved
  apple_team_identifier: string

require-approval:                       # no params
doc:
  md: string

repack:                                 # re-bundle JS/metadata without a native rebuild
  build_id: string                      # required
  profile, embed_bundle_assets, js_bundle_only,
  ios_signing_use_source_app_entitlements, ios_signing_app_entitlements_path,
  message, repack_version, repack_package
```

### Custom jobs (steps-based)
```yaml
jobs:
  custom_job:
    runs_on: linux-medium            # see the worker table below
    image: auto | specific_image     # defaults to 'auto'
    steps:
      - name: Step name
        run: command
        shell: bash
        working_directory: ./path
        id: step_identifier
      - uses: eas/built_in_function
        with:
          parameter: value
```
`steps` may only be provided on custom jobs and `build` jobs.

### Workers (`runs_on`)
Available on custom jobs and on the `build`, `maestro`, `maestro-cloud`, and `repack` pre-packaged jobs (`docs/pages/eas/workflows/syntax.mdx:1706-1738`).

| Worker | vCPU / cores | RAM (GiB) | SSD (GiB) | Notes |
| --- | --- | --- | --- | --- |
| `linux-medium` | 4 | 16 | 14 | Default. |
| `linux-large` | 8 | 32 | 28 | |
| `linux-medium-nested-virtualization` | 4 | 16 | 14 | Allows Android Emulators. |
| `linux-large-nested-virtualization` | 4 | 32 | 28 | Allows Android Emulators. |
| `macos-medium` | 5 efficiency cores | 20 | 125 | iOS jobs, including simulators. |
| `macos-large` | 10 efficiency cores | 40 | 125 | iOS jobs, including simulators. |

> Android Emulator jobs **require** a `linux-*-nested-virtualization` worker; iOS builds and iOS Simulator jobs **require** a `macos-*` worker.

### Built-in functions (`eas/` prefix)
`eas/checkout`, `eas/install_node_modules`, `eas/prebuild`, `eas/download_build`, `eas/restore_cache`, `eas/save_cache`, `eas/use_npm_token`, `eas/send_slack_message`, `eas/upload_artifact`, `eas/download_artifact`.

PostHog family: `eas/posthog_capture_event`, `eas/posthog_flag_rollout`, `eas/posthog_wait_for_metric`, `eas/posthog_wait_for_query`, `eas/posthog_annotation`, `eas/posthog_upload_sourcemaps`.

Shell helpers: `set-output NAME VALUE`, `set-env NAME VALUE`.

### Interpolation & contexts
Expression syntax: `${{ context.property }}`. Contexts: `github`, `inputs`, `needs`, `after`, `steps`, `env`, `workflow`, `app_store_connect`, `metadata`.
Functions: `success()`, `failure()`, `contains()`, `startsWith()`, `endsWith()`, `hashFiles()`, `fromJSON()`, `toJSON()`, `replaceAll()`, `substring()`.

- `metadata` — populated on `build` jobs, `{}` elsewhere (including custom jobs): `buildProfile`, `appVersion`, `appBuildVersion`, `sdkVersion`, `runtimeVersion`, `gitCommitHash`, `distribution` (`docs/pages/eas/workflows/environment.mdx:173-188`).
- `workflow` — `id`, `name`, `filename`, `url`.
- `env` — available inside a job (`params`, `if`, `env`, `outputs`, `run`), **not** at the top level of the file (so not in `on` triggers).

### Defaults
```yaml
defaults:
  image: sdk-56          # default VM image for every job; an sdk-XX tag is recommended
  hooks:                 # default hooks; each job runs the keys that apply to it
    before_install_node_modules:
      - run: echo "//npm.pkg.github.com/:_authToken=$GITHUB_TOKEN" >> .npmrc
  run:
    working_directory: ./path
  tools:
    node: version        # installed via nvm
    yarn: version        # npm -g
    corepack: true       # boolean, defaults to false
    pnpm: version        # npm -g
    bun: version
    ndk: version         # sdkmanager
    bundler: version     # gem install -v
    fastlane: version    # gem install -v
    cocoapods: version   # gem install -v
```
Jobs override `defaults.image` with `jobs.<job_id>.image`, and any `defaults.hooks` key with their own (`[]` opts out).

`defaults.image` is unversioned EAS Workflows surface available to both SDK tracks — set it to your own track's tag (`sdk-56` or `sdk-57`); the docs example happens to use `sdk-57` (`docs/pages/eas/workflows/syntax.mdx:543-551`). "Using an `sdk-XX` image tag is recommended" over `latest`, which moves under you at each SDK release.

### Concurrency
```yaml
concurrency:
  cancel_in_progress: true
  group: ${{ workflow.filename }}-${{ github.ref }}
```
`cancel_in_progress: true` cancels in-progress runs on the same branch when a new GitHub-triggered run starts — it is currently the only thing `concurrency` actually lets you set. **Custom concurrency groups are not supported yet**; `group` is a forward-compatibility placeholder, so set the documented value above and your workflow stays compatible when custom groups land (`docs/pages/eas/workflows/syntax.mdx:599-613`).

### Example — multi-job with dependencies
```yaml
jobs:
  test:
    steps:
      - uses: eas/checkout
      - run: npm test

  build_ios:
    needs: [test]
    type: build
    params:
      platform: ios

  deploy:
    needs: [build_ios]
    type: deploy
    params:
      prod: true
```

### Example — manual inputs
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        required: true

jobs:
  deploy:
    steps:
      - run: deploy-to ${{ inputs.environment }}
```

---

## 12. EAS Workflows vs. custom builds — two different YAML DSLs

They look alike and are easy to confuse. Both live under `.eas/`.

| | EAS Workflows | Custom builds |
| --- | --- | --- |
| Path | `.eas/workflows/*.yml` | `.eas/build/*.yml` |
| Selected by | `eas workflow:run <file>` or an `on` trigger | a build profile's `config` field in `eas.json` (e.g. `"config": "production.yml"`) |
| Root shape | `name:` + `jobs:` (each with `type:` or `steps:`) | `build:` with `name:` and `steps:` |
| Scope | orchestrates whole pipelines (build, submit, update, deploy, tests) | customises the steps of **one** EAS Build run |

Custom builds have their own function set (`docs/pages/custom-builds/schema.mdx`) — it overlaps with Workflows on `eas/checkout`, `eas/install_node_modules`, `eas/prebuild`, `eas/use_npm_token`, `eas/send_slack_message`, `eas/upload_artifact`, but adds build-internal functions that do **not** exist in Workflows: `eas/build`, `eas/resolve_build_config`, `eas/resolve_apple_team_id_from_credentials`, `eas/get_credentials_for_build_triggered_by_github_integration`, `eas/configure_eas_update`, `eas/inject_android_credentials`, `eas/configure_ios_credentials`, `eas/configure_android_version`, `eas/configure_ios_version`, `eas/run_gradle`, `eas/generate_gymfile_from_template`, `eas/run_fastlane`, `eas/find_and_upload_build_artifacts`, `eas/restore_build_cache`, `eas/save_build_cache`, `eas/install_maestro`, `eas/start_android_emulator`, `eas/start_ios_simulator`, `eas/maestro_test`.

Minimal custom build config:
```yaml .eas/build/hello-world.yml
build:
  name: Hello World!
  steps:
    - run: echo "Hello, world!"
```

Docs: `/custom-builds/get-started/`, `/custom-builds/schema/`, `/custom-builds/functions/`.

---

## SDK 57 delta

EAS itself is unversioned tooling, so **the `eas.json` schema, EAS CLI commands, submit fields, rollout semantics, and Workflows syntax in §§1–12 are identical on both tracks** — there is no SDK-gated fork of the EAS surface itself. What changes is which **build image** your jobs land on, the **project template tag**, a handful of **dependency pins**, and one genuine **behaviour** change: EAS steps that shell out to your project's Expo CLI inherit SDK 57's new `expo prebuild` semantics (below). Checked: `docs/pages/build-reference/infrastructure.mdx`, `docs/pages/eas/workflows/*`, `docs/pages/build/eas-json.mdx`, `docs/public/static/schemas/unversioned/eas-json-*`, and `packages/expo/bundledNativeModules.json` on `origin/sdk-56` / `origin/sdk-57`.

> Do **not** read pins out of `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json` — those files are frozen at SDK-release time and already stale (they say `react-native-screens` 4.25.2 and `expo-router` ~56.2.9; the shipped values are ~4.26.0 and ~56.2.16). `packages/expo/bundledNativeModules.json` on the release branch ships inside the `expo` package and is what `expo install` actually resolves against.

### Behaviour that IS SDK-gated

**`npx expo prebuild` cleans by default on SDK 57.** `@expo/cli` 57.0.0 breaking change: "_Make `expo prebuild` clear and regenerate the native folders by default. Pass `--no-clean` to apply changes to the existing folders instead._" (#47209). Verified in source: on `origin/sdk-57` `packages/@expo/cli/src/prebuild/index.ts:67` resolves `clean: !args['--no-clean']`, whereas `origin/sdk-56` line 66 resolves `clean: args['--clean']` (opt-in). This is not an EAS change — EAS invokes whichever Expo CLI your project resolves — but it changes what three things documented above actually do:

| Surface | SDK 56 effect | SDK 57 effect | Keep SDK 56 semantics with |
| --- | --- | --- | --- |
| `prebuildCommand` build-profile option (§1) | additive prebuild unless you passed `--clean` | wipes and regenerates `android/` + `ios/` | `"prebuildCommand": "prebuild --no-clean"` |
| `eas/prebuild` Workflows function (§11) | additive | clean | pass `--no-clean` via the profile's `prebuildCommand`; see the caveat below |
| `eas/prebuild` custom-builds function (§12) | additive | clean | same caveat |

> **The `eas/prebuild` `clean` input is not the escape hatch.** Its documented semantics are "_whether the function should use `--clean` flag when running the command. Defaults to false._" (`docs/scenes/eas-functions/_prebuild.mdx`) — it only decides whether `--clean` is *added*, and there is no documented `--no-clean` toggle. So on SDK 57 `clean: false` still lands on the CLI's new clean-by-default path. The reliable lever is the build profile's `prebuildCommand` (`"prebuild --no-clean"`); EAS appends `--platform` and `--non-interactive` itself. Note the two DSLs spell arguments differently: Workflows uses `- uses: eas/prebuild` + `with:` (`docs/pages/eas/workflows/syntax.mdx:1899-1928`), custom builds uses `- eas/prebuild:` + `inputs:` (`docs/pages/custom-builds/schema.mdx:1097-1115`).

If you rely on committed native folders plus hand-edits surviving a managed build, settle this before upgrading — it is a no-op on SDK 56 and load-bearing on SDK 57.

### New in 57

**Build images.** `latest` now resolves to the SDK 57 image (`docs/pages/build-reference/infrastructure.mdx:23-35`). Pin an explicit `sdk-XX` tag if you want reproducibility across the 56→57 boundary.

| | SDK 56 | SDK 57 |
| --- | --- | --- |
| Android image | `ubuntu-26.04-jdk-17-ndk-r27b` (`sdk-56`) | `ubuntu-26.04-jdk-17-ndk-r27b-sdk-57` (`latest`, `sdk-57`) |
| Android Node / npm / pnpm / Bun | 22.22.2 / 10.9.4 / 10.33.3 / 1.3.13 | 22.23.1 / 10.9.8 / 11.9.0 / 1.3.14 |
| Android node-gyp / NDK / Java | 12.3.0 / 27.1.12297006 / 17 | 13.0.0 / 27.1.12297006 / 17 |
| iOS image | `macos-tahoe-26.4-xcode-26.4` (`sdk-56`) | `macos-tahoe-26.5-xcode-26.6` (`latest`, `sdk-57`) |
| macOS / Xcode | Tahoe 26.4.1 / Xcode 26.4 (17E202) | Tahoe 26.5.2 / Xcode 26.6 (17F113) |
| fastlane / CocoaPods / Ruby | 2.233.1 / 1.16.2 / 3.2 | 2.236.1 / 1.16.2 / 3.2 |
| Maestro (both platforms) | 2.5.1 | 2.6.1 |

Source: `docs/pages/build-reference/infrastructure.mdx:93-125` (Android) and `:380-416` (iOS).

**Project template** — the Workflows getting-started guide now scaffolds with `default@sdk-57` (`docs/pages/eas/workflows/get-started.mdx:25-30`). `default@sdk-56` remains correct for the SDK 56 track, and only affects **new** projects:
```sh
npx create-expo-app@latest --template default@sdk-57
```

### Newly documented, but unversioned — applies to SDK 56 too

Both of the following landed in the docs around the SDK 57 release and are easy to mistake for 57 features. Neither is SDK-gated; do not schedule upgrade work for them.

**`defaults.image` Workflows key** (§11) — available on both tracks. The docs example value is `sdk-57` only because the docs track the newest SDK; set your own track's tag. Added to the docs on 2026-07-13 (#47481) — nearly two weeks *after* `expo@57.0.0` was published (2026-06-30), which is why it reads as a 57 feature and is not one.

**Gradle JVM args on the worker** (`docs/pages/build-reference/infrastructure.mdx:66-92`). EAS Build sets `GRADLE_OPTS` on the worker before Gradle runs:

| Resource class | `-Xmx` |
| --- | --- |
| `medium` | `4g` |
| `large` | `8g` |

Plus `-XX:MaxMetaspaceSize=1g`, `-XX:+HeapDumpOnOutOfMemoryError`, `-Dfile.encoding=UTF-8` (via `-Dorg.gradle.jvmargs`), and top-level `-Dorg.gradle.parallel=true`, `-Dorg.gradle.daemon=false`.

> This **overrides any `org.gradle.jvmargs` in your project's `gradle.properties`**. To take it back, set `GRADLE_OPTS` in the build profile's `env`, in a workflow `env`, or as an EAS environment variable — project values take precedence over the worker default.

### Version pins (56 → 57)
From `git show origin/sdk-56:packages/expo/bundledNativeModules.json` vs `origin/sdk-57`. Only the packages that affect build reproducibility and fingerprints are listed here; see reference `01` for the full table.

| Package | SDK 56 | SDK 57 |
| --- | --- | --- |
| `react-native` | `0.85.3` | `0.86.0` |
| `@expo/fingerprint` | `~0.19.9` | `~0.20.6` |
| `expo-updates` | `~56.0.23` | `~57.0.10` |
| `expo-dev-client` | `~56.0.24` | `~57.0.9` |

Two traps in this table:
- **First-party `expo-*` pins are not flat `~57.0.0`.** Both lines keep shipping patches, so each package sits at its own patch level (`expo-router` `~56.2.16` → `~57.0.8`, `expo-video` `~56.1.4` → `~57.0.2`). Do not synthesise a version — read it out of `bundledNativeModules.json` on the release branch.
- **The SDK 56 column moves.** The values above are the current head of the 56 line, not what shipped at 56.0.0. If your lockfile is older, the honest comparison is your pin vs. the 57 pin.

`@expo/fingerprint` 0.20.x carries **no user-facing change exclusive to SDK 57**: 0.20.0 through 0.20.6 are all "_This version does not introduce any user-facing changes._" except 0.20.3's extra default `getConfig` exclusion packages (#47503), and that was backported to the 56 line as 0.19.7 the same day. It is a bumped pin, not a behaviour change. Fingerprints still differ across the SDK boundary because the dependency set does.

### Not a 57 delta (checked, and deliberately excluded)
- **`appVersion` runtime-version policy honouring `ios.version`/`android.version`** (#47416) shipped in **both** `@expo/config-plugins` 56.0.12 and 57.0.3, on the same day. It is SDK 56 patch drift, documented in §5 — not something you gain by upgrading.
- **`SourceSkips.ExpoConfigVersions`** itself is *not* a 57 delta and is perfectly safe to cite — it is enum member `1 << 0` in the released `@expo/fingerprint` source on **both** `origin/sdk-56` and `origin/sdk-57` (`packages/@expo/fingerprint/src/sourcer/SourceSkips.ts:11`), covering `version`, `android.versionCode`, and `ios.buildNumber`. What is unreleased is only the *extension* of that existing flag to also strip `ios.version` / `android.version`, which sits under `## Unpublished` on `main`. Cite the flag; do not promise the extended stripping.
- **Browser-based `expo login`** (#46832) is under `## Unpublished` on `main` and appears in **neither** `origin/sdk-56` nor `origin/sdk-57` changelogs. It is post-57 and unreleased; do not cite it. On both shipped tracks `expo login` is still username/password.
- **The Node.js minimum raise to `^22.13.0`** (#47202) is *not* an SDK 57 delta, despite landing on `main` one day before the 57.0.0 release. The commit (`802b8c3402f`, 2026-06-24) is not an ancestor of `origin/sdk-57`, its changelog entry sits under `## Unpublished` on `main`, `packages/@expo/cli/bin/cli.ts:15` still reads `const NODE_MIN = [20, 19, 4]` on **both** `origin/sdk-56` and `origin/sdk-57`, and `git show origin/sdk-57:packages/expo/package.json` has **no `engines` field at all**. Node >= 20.19.4 still works on SDK 57. So the `node` field in `eas.json` (§1) and `defaults.tools.node` in Workflows (§11) are bounded the same way on both tracks; do not tell people SDK 57 requires Node 22.13. (The EAS *images* ship Node 22.x anyway — see the image table above — but that is an image default, not an SDK floor.)
