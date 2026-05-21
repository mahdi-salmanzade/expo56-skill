# EAS (Expo Application Services) — Full Workflow Reference (SDK 56)

Official Expo SDK 56 documentation covering the end-to-end EAS workflow: Build configuration, internal distribution, app credentials, Submit, Update (channels/branches/migration), Workflows (CI/CD), Metadata, and environment variables.

> Scope note: This document complements the build-performance reference already captured. It focuses on configuration fields, commands, and verbatim examples across the EAS surface. SDK 56-specific items are flagged inline where applicable.

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
Configure the build VM with `resourceClass`: `"medium"` (default) or `"large"` (requires a paid plan). Set per platform:
```json
{
  "android": { "resourceClass": "medium" },
  "ios": { "resourceClass": "large" }
}
```

### Base image selection
Controls default dependency versions (Node.js, Yarn, CocoaPods, OS version, Xcode). EAS automatically selects the appropriate image based on the SDK version for Expo projects.

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

### CI/CD limitations
The `--non-interactive` flag enables automated builds but prevents adding new devices to iOS ad hoc provisioning profiles. Interactive builds with Apple authentication are required for device registration.

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
1. **Distribution Certificate** — single per Apple Developer account; used across all apps. Expiration doesn't affect production apps but prevents new submissions. Not app-specific.
2. **Push Notification Keys (APN)** — max 2 per account; usable across multiple apps. Revocation stops push notifications until replaced. Keys don't expire; tokens remain valid when replaced.
3. **Provisioning Profiles** — app-specific; one per App Store submission. Expire after 12 months but don't affect production apps. Associated with the distribution certificate.

### EAS credential management
- `eas credentials` — inspect, download, and manage all credentials (select platform and profile).
- `eas build:resign` — re-sign existing iOS builds with new provisioning profiles, reusing the application artifact without a full rebuild (useful for adding test devices).
- Deleting credentials via EAS removes them only from Expo's servers, not Apple's. Full deletion requires the Apple Developer Console.

---

## 4. EAS Submit

Sources:
- https://docs.expo.dev/submit/introduction/
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
- **Android (Google Play)**: uploads to a specified track (`internal`, `alpha`, `beta`, `production`). Requires a manual first submission before API-based submissions work. Non-production tracks limit availability to testers.
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
- Common Android fields: `serviceAccountKeyPath`, `track`, `releaseStatus`, `changesNotSentForReview`.
- Common iOS fields: `appleId`, `ascAppId`, `appleTeamId`, `ascApiKeyPath`, `ascApiKeyId`, `ascApiKeyIssuerId`, `sku`, `language`, `companyName`.
- See the EAS Submit schema reference (`/eas/json#eas-submit`) for the complete field list.

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

---

## 7. EAS Update — migrate from CodePush

Sources:
- https://docs.expo.dev/eas-update/codepush/ (the original `/eas-update/migrate-to-eas-update/` URL 404s; resolved via WebSearch)
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
- Gradual rollouts: use EAS Update's rollout strategies.
- Direct update control: `Updates.checkForUpdateAsync()`.

Recommendation: use the latest Expo SDK version — migration instructions are not available for older Expo SDK / React Native versions.

---

## 8. EAS Metadata

Source: https://docs.expo.dev/eas/metadata/

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

Source: https://docs.expo.dev/eas/environment-variables/

### Visibility levels
- **Plain text** — visible on the website, in EAS CLI, and in logs.
- **Sensitive** — obfuscated in EAS Build and Workflows job logs, with toggle visibility on the website.
- **Secret** — not readable outside the EAS servers, including on the website and in EAS CLI.

### Environments
Three by default: `development`, `preview`, `production`. Custom names are available on Enterprise and Production plans.

### Commands & usage

Create a variable:
```sh
eas env:create --name EXPO_PUBLIC_API_URL --value https://api.example.com \
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

Other listed commands: `eas env:list`, `eas env:pull`.

### Notes
- Local `.env` files are **not** available to remote EAS jobs.
- "Anything that is included in your client-side code should be considered public." The `EXPO_PUBLIC_` prefix exposes a variable to client-side code.
- Secret variables don't secure values embedded in the application itself.
- Scopes: variables can be project-wide (a single EAS project) or account-wide (across all projects).

---

## 10. EAS Workflows — get started

Source: https://docs.expo.dev/eas/workflows/get-started/

### Overview
EAS Workflows automate React Native CI/CD development and release processes using YAML configuration files triggered by various events.

### Project setup
1. Create an Expo account.
2. Initialize a project (SDK 56-specific template):
   ```sh
   npx create-expo-app@latest --template default@sdk-56
   ```
3. Link to EAS: `npx eas-cli@latest init`
4. Add `eas.json` to the project root.

### Directory structure
Create `.eas/workflows/` at the project root containing YAML workflow files (e.g. `create-production-builds.yml`).

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
npx eas-cli@latest workflow:run create-production-builds.yml
```

### Triggers (high level)
- GitHub events via `on.push`.
- App Store Connect events via `on.app_store_connect` (e.g. `ready_for_review`).

### Tooling
The Expo Tools VS Code extension provides descriptions and autocompletions for workflow files.

---

## 11. EAS Workflows — syntax reference

Source: https://docs.expo.dev/eas/workflows/syntax/

### Top-level keys
Workflow YAML files live in `.eas/workflows/` alongside `eas.json`.
- `name` — human-friendly identifier shown on the EAS dashboard.
- `on` — GitHub/ASC events that trigger the workflow (can also be run manually via `eas workflow:run`).
- `jobs` — the tasks comprising a run.
- `defaults` — global parameters applied to all jobs.
- `concurrency` — controls execution when multiple runs occur simultaneously.

### Event triggers (`on`)
1. **`push`** — `branches` (globs, exclude with `!`), `tags`, `paths`.
2. **`pull_request`** — `branches`, `types` (`opened`, `ready_for_review`, `reopened`, `synchronize`, `labeled`), `paths`.
3. **`pull_request_labeled`** — `labels`.
4. **`app_store_connect`** — `app_version.states`, `build_upload.states`, `external_beta.states`, `beta_feedback.types` (`crash`, `screenshot`).
5. **`schedule`** — `cron` (Unix-cron, GMT, default branch only).
6. **`workflow_dispatch`** — `inputs` accepting `string`, `boolean`, `number`, `choice`, `environment` types.

### Job definition
```yaml
jobs:
  job_id:
    name: Human-friendly job title
    environment: production | preview | development   # defaults to production
    env:
      VAR_NAME: value
    needs: [other_job_id]   # runs only if dependencies succeed
    after: [other_job_id]   # runs after dependencies complete (regardless of status)
    if: ${{ conditional_expression }}
```

### Pre-packaged job types (`type`)
- **`build`** — params: `platform`, `profile`, `message`; outputs: `build_id`, `app_version`, `platform`, `distribution`, etc.
- **`deploy`** — params: `alias`, `prod`; outputs: `deploy_url`, `deploy_identifier`, `deploy_dashboard_url`.
- **`update`** — params: `message`, `platform`, `branch`, `channel`, `private_key_path`; outputs: `first_update_group_id`, `updates_json`.
- **`submit`** — params: `build_id`, `profile`, `groups`; hooks: `before_submit`, `after_submit`.
- **`testflight`** — params: `build_id`, `internal_groups`, `external_groups`, `changelog`.
- **`maestro`** — params: `build_id`, `flow_path`, `shards`, `retries`.
- **`maestro-cloud`** — params: `maestro_project_id`, `flows`, `device_model`, `device_os`.
- **`fingerprint`** — calculates the project fingerprint.
- **`get-build`** — retrieves existing builds matching criteria.
- **`slack`** — posts to Slack via webhook.
- **`github-comment`** — comments on GitHub PRs with reports.
- **`require-approval`** — waits for manual approval.
- **`doc`** — displays Markdown in workflow logs.
- **`repack`** — repackages existing builds.

### Custom jobs (steps-based)
```yaml
jobs:
  custom_job:
    runs_on: linux-medium | linux-large | macos-medium | macos-large
    image: auto | specific_image
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

### Built-in functions (`eas/` prefix)
`eas/checkout`, `eas/install_node_modules`, `eas/prebuild`, `eas/download_build`, `eas/restore_cache`, `eas/save_cache`, `eas/use_npm_token`, `eas/send_slack_message`, `eas/download_artifact`.

Shell helpers: `set-output NAME VALUE`, `set-env NAME VALUE`.

### Interpolation & contexts
Expression syntax: `${{ context.property }}`. Contexts: `github`, `inputs`, `needs`, `after`, `steps`, `env`, `workflow`, `app_store_connect`.
Functions: `success()`, `failure()`, `contains()`, `startsWith()`, `endsWith()`, `hashFiles()`, `fromJSON()`, `toJSON()`, `replaceAll()`, `substring()`.

### Defaults
```yaml
defaults:
  run:
    working_directory: ./path
  tools:
    node: version
    yarn: version
    pnpm: version
    bundler: version
    fastlane: version
```

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

## SDK 56-specific notes

- **Project template**: `npx create-expo-app@latest --template default@sdk-56` is the SDK 56 starter referenced in the Workflows getting-started guide.
- **Per-step timing / precompiled artifacts / EAS Observe (coming soon)**: The build-configuration and eas-json pages reviewed for this section did not surface SDK 56-specific text on per-step timing, precompiled artifacts, or "EAS Observe coming soon." These items are likely documented on the build-performance / changelog pages already captured separately (or announced in the SDK 56 release notes) rather than on the configuration-reference pages above. Flagged here as not found in the pages fetched for this domain.

---

## Pages that could not be accessed
- `https://docs.expo.dev/eas-update/migrate-to-eas-update/` — returned HTTP 404. The current canonical page is `https://docs.expo.dev/eas-update/codepush/` (resolved via WebSearch); content for §7 was pulled from there. Note that the CodePush page itself defers installation/configuration command details to the EAS Update "Get started" guide.
