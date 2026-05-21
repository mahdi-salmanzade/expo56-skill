# Development Workflow, Debugging & Testing — Expo SDK 56

A knowledge-base reference covering development builds, the dev workflow, debugging tools, unit testing with jest-expo, and E2E testing with Maestro. Compiled from official Expo documentation. Source URLs are listed per section.

> **SDK 56 note:** Expo Go for SDK 56 is **not** available on the Apple App Store or Google Play Store. See the [Expo Go for SDK 56](#expo-go-for-sdk-56-not-on-the-stores) section for install-via-CLI / TestFlight / `eas go` instructions.

---

## 1. Workflow Overview

Source: https://docs.expo.dev/workflow/overview/

### Development build vs Expo Go

- **Expo Go** — "the fastest way to get started with React Native, especially when combined with Snack." It is "a limited playground and not useful for building production-grade projects."
- **Development build** — recommended for serious projects. It is "a debug build of your app that contains `expo-dev-client` library" and provides "a more flexible, reliable, and complete development environment than Expo Go." Development builds can be created locally or via EAS Build in the cloud.

### The core development loop (four main activities)

1. **Write and run JavaScript code** — components, logic, npm libraries with no native changes. Reflected immediately without native interaction.
2. **Update app configuration** — modify `app.json` / `app.config.js` (name, icon, splash screen, etc.). Config plugins allow native modifications without editing native code directly.
3. **Write native code or modify native project configuration** — requires access to native directories or creating a local Expo Module.
4. **Install libraries requiring native code modifications** — may provide config plugins or need app config updates; requires a development build.

### Continuous Native Generation (CNG)

When initializing with `npx create-expo-app`, the native `android` and `ios` directories are **not** created by default. CNG generates them on-demand:

```sh
npx expo prebuild
```

The native directories are automatically added to `.gitignore` and can be regenerated anytime.

### Cloud vs local development

- **EAS Build (cloud)** — a single command; no Android Studio or Xcode install needed.
- **Local** — requires Android Studio and/or Xcode; use:

```sh
npx expo run:android   # or
npx expo run:ios
```

### Workflow stages

- **Initialize**: `create-expo-app`
- **Share / Testing**: Internal distribution or local production builds
- **Release**: EAS Submit or manual store submission
- **Monitor**: crash reporting (Sentry / BugSnag) and analytics
- **Update**: `expo-updates` and EAS Update for instant JavaScript updates

---

## 2. Set Up Your Environment

Source: https://docs.expo.dev/get-started/set-up-your-environment/

### Choosing your approach

> "Expo Go is a playground for students and learners to try Expo quickly. A development build is a build of your own app that includes Expo's developer tools."

Platform/device matrix (each supports Expo Go and/or EAS or local development build):

- **Android device** — Expo Go, EAS development build, or local development build
- **Android Emulator** — Expo Go, EAS development build, or local development build
- **iOS device** — Expo Go or EAS / local development build
- **iOS Simulator** — Expo Go or EAS / local development build

### Prerequisites & tools

**All Android setups require:**
- Android Studio
- Android 15 (VanillaIceCream) SDK with Platform 35
- Android SDK Build-Tools and Emulator
- `ANDROID_HOME` environment variable configured
- ADB (Android Debug Bridge) verification

**All iOS setups require:**
- Xcode
- Xcode Command Line Tools
- iOS Simulator components
- Watchman

**For local builds:**
- JDK 17 (Azul Zulu for macOS; OpenJDK for Windows/Linux)
- Watchman

**For EAS builds:**

```sh
npm install -g eas-cli
```

Then sign up for an Expo account and log in to EAS.

### Key commands

Development build workflow:

```sh
eas build:configure
eas build --platform [android|ios] --profile development
```

Local development:

```sh
npx expo install expo-dev-client
npx expo run:android   # or
npx expo run:ios
```

### iOS Developer Mode

For iOS device development, enable **Developer Mode** via **Settings > Privacy & Security**. This requires a device restart and confirmation.

---

## 3. Development Builds

### 3.1 Introduction

Source: https://docs.expo.dev/develop/development-builds/introduction/

A development build is a "Debug" build that includes the `expo-dev-client` library. This library extends React Native's built-in development tooling with capabilities like network request inspection and a launcher UI for switching between development servers and app deployments.

**Two components that differ between Expo Go and a dev build:**
- **Native app** — the installable app on the device. Expo Go is pre-built and unchangeable; development builds allow customizing app name, icon, and which native libraries are included.
- **JavaScript bundle** — your app's UI and business logic, reloaded live from your local machine during development via `npx expo start`.

**Why choose development builds — when you need to:**
1. **Use native libraries outside Expo Go** (e.g. `react-native-firebase` contains native code unavailable in Expo Go).
2. **Test app customizations** — icons, names, splash screens, animations like `SplashScreen.setOptions`.
3. **Implement remote push notifications** — server-sent notifications require tied push certificates, best tested in dev builds for production parity.
4. **Establish App / Universal links** — Android App Links and iOS Universal Links require native app URL associations.
5. **Support older SDKs on iOS devices** — Expo Go supports only the current SDK version; dev builds allow flexibility.

### 3.2 Create a Build

Source: https://docs.expo.dev/develop/development-builds/create-a-build/

**Prerequisites:** an Expo account; EAS CLI installed and logged in; platform-specific requirements.

```sh
npm install -g eas-cli && eas login
```

**Install the dev client:**

```sh
npx expo install expo-dev-client
```

**Build for Android:**

```sh
eas build --platform android --profile development
```

**Build for iOS Simulator** — set `simulator: true` in `eas.json` under the development profile:

```json
{
  "build": {
    "development": {
      "ios": {
        "simulator": true
      }
    }
  }
}
```

```sh
eas build --platform ios --profile development
```

> These iOS simulator builds work only on simulators, not physical devices.

**Build for iOS device** — requires a paid Apple Developer account for signing credentials:

```sh
eas build --platform ios --profile development
```

After EAS finishes, install via the CLI prompt, the expo.dev dashboard, or Expo Orbit. Then start the bundler:

```sh
npx expo start
```

It automatically detects the development client installation.

### 3.3 Use Development Builds

Source: https://docs.expo.dev/develop/development-builds/use-development-builds/

**Start the development server:**

```sh
npx expo start
```

**Open your project:**
- **Emulator/Simulator** — press `a` (Android Emulator) or `i` (iOS Simulator).
- **Physical device** — scan the QR code shown by Expo CLI.

**Launcher screen** — When launching the dev build from the home screen, you'll see a launcher. If a bundler is on your local network, or you've authenticated with Expo in both the CLI and the dev build, you can connect directly; otherwise use the QR code.

**Open the debug menu** — press **Cmd ⌘ + d** (macOS) or **Ctrl + d** (Windows/Linux) in Expo CLI, or shake the device. Lets you switch to a different version of your app.

**Rebuild when necessary** — if you add native libraries (e.g. `expo-secure-store`), rebuild the dev client; native dependencies aren't included automatically on package install.

### 3.4 Development Workflows

Source: https://docs.expo.dev/develop/development-builds/development-workflows/

**Tunnel URLs** — expose your dev server on a public URL (useful behind restrictive networks/firewalls):

```sh
npx expo start --tunnel
```

**Published updates via EAS** — bundle JS and assets into Expo-hosted updates so dev builds can load changes without checking out a specific commit:

```sh
eas update
```

**Manual update URL entry** in a dev build:

```
https://u.expo.dev/[project-id]?channel-name=[channel-name]
```

**Deep linking** — load a specific update:

```
{scheme}://expo-development-client/?url={manifestUrl}
```

For automation (CI/CD), add `disableOnboarding=1` to skip onboarding screens.

**QR codes** — generate via `https://qr.expo.dev/development-client` with `appScheme` and `url` parameters.

**Extensions:**
- **Dev Menu extension** — use the `registerDevMenuItems` API to add custom buttons to the dev menu.
- **EAS Update extension** — install and view/load published updates inside the dev client's Extensions panel:

```sh
npx expo install expo-dev-client expo-updates
```

**Configuration** — set `runtimeVersion` in your app config to enforce the API contract between JavaScript and native layers, ensuring compatibility between builds and loaded bundles.

---

## 4. Debugging

### 4.1 Debugging Runtime Issues

Source: https://docs.expo.dev/debugging/runtime-issues/

**Development errors — structured approach:**
- Search error messages on Google / Stack Overflow.
- Examine the stack trace.
- "Isolate the code that's throwing the error" — revert to a working version and reapply changes incrementally.
- Use breakpoints or `console.log` to verify execution and variable values.
- Simplify complex code (e.g. remove state libraries like Redux) to narrow down the issue.
- Create a minimal reproducible example in a blank `npx create-expo-app` project.

**Native debugging:**

Android Studio:
```sh
npx expo prebuild -p android
```
Open and debug in Android Studio. Delete the `android` directory afterward to keep Expo CLI management.

Xcode (macOS):
```sh
npx expo prebuild -p ios
```
Open via `xed ios`; use the LLDB debugger and Xcode tools.

**Viewing native logs:**
- **Android** — `adb logcat` streams logs from connected devices/emulators (WebADB is an SDK-free alternative).
- **iOS** — use Xcode's Console app (**Shift + Cmd ⌘ + 2**) after connecting a device or simulator.

**Production errors:**
> "The best first step in addressing a production error is to reproduce it locally."

Test production behavior locally:
```sh
npx expo start --no-dev --minify
```

Additional steps:
- Check platform crash reports (Google Play Console, Xcode Crashes Organizer).
- Use native log tools on a reproducing device.
- Integrate error reporting services (Sentry, BugSnag).
- Profile performance with React Native DevTools.

### 4.2 Debugging Tools

Source: https://docs.expo.dev/debugging/tools/

**Developer menu** — open it:
- Press `m` in the terminal where Expo CLI started.
- **Android device**: shake vertically.
- **Android Emulator/USB**: `Cmd ⌘ + m` or `Ctrl + m`; or `adb shell input keyevent 82`.
- **iOS device**: shake or three-finger tap.
- **iOS Simulator/USB**: `Ctrl + Cmd ⌘ + z` or `Cmd ⌘ + d`.

Menu options: Copy link, Reload, Go Home; toggle performance monitor (RAM, JS heap, view counts, FPS); toggle element inspector (inspect elements, performance overlay, network details, highlight touchables); Open DevTools (Console, Sources, Network, Memory, Components, Profiler for Hermes apps); Fast Refresh toggle.

**React Native DevTools** (modern debugger for Hermes apps, React Native 0.76+):

Open it by pressing `j` in the terminal where Expo started.

Features:
- **Console** — interactive terminal executing JavaScript in app scope.
- **Sources** — set breakpoints by clicking line numbers or adding `debugger` statements.
- **Exceptions** — pause on errors; enable "Pause on caught exceptions" for handled errors.
- **Network** — inspect fetch requests and external media (Expo only).
- **Memory** — usage and heap snapshots.
- **Components** — inspect React props and styles.
- **Profiler** — record/analyze JS performance (debug builds only; sourcemaps not yet symbolicated).

**VS Code debugging** — integrates with React Native DevTools via the inspector protocol:
1. Connect your app.
2. Open command palette (`Ctrl + Shift + p` / `Cmd ⌘ + Shift + p`).
3. Run the "Expo: Debug ..." command.

> The Radon IDE extension offers advanced features (network inspection, router integration) — paid, 30-day trial.

**React Native Debugger (legacy)** — deprecated for Expo SDK 50+; incompatible with Hermes and Remote JS debugging.
- Install (macOS): `brew install react-native-debugger`
- Startup: set port to `8081` (`Cmd ⌘ + t`), run `npx expo start`, select "Debug remote JS".
- Network inspection: right-click → "Enable Network Inspect" (limited); alternatives: Charles Proxy, Proxyman, mitmproxy, Fiddler.

---

## 5. Unit Testing with jest-expo

Source: https://docs.expo.dev/develop/unit-testing/

**Installation:**

```sh
npx expo install jest-expo jest @types/jest --dev
```

For TypeScript projects, add `"jest"` to `types` in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "types": ["jest"]
  }
}
```

**package.json configuration** — add a test script and the Jest preset:

```json
{
  "scripts": {
    "test": "jest --watchAll"
  },
  "jest": {
    "preset": "jest-expo"
  }
}
```

For projects using multiple npm packages, configure `transformIgnorePatterns`:

```json
"jest": {
  "preset": "jest-expo",
  "transformIgnorePatterns": [
    "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg)"
  ]
}
```

**Writing unit tests** — create a `__tests__` directory; test files use `-test.ts|tsx` extensions:

```tsx
import { render } from '@testing-library/react-native';
import HomeScreen, { CustomText } from '@/app/index';

describe('<HomeScreen />', () => {
  test('Text renders correctly on HomeScreen', () => {
    const { getByText } = render(<HomeScreen />);
    getByText('Welcome!');
  });
});
```

Run tests:

```sh
npm run test
```

**Snapshot testing:**

```tsx
test('CustomText renders correctly', () => {
  const tree = render(<CustomText>Some text</CustomText>).toJSON();
  expect(tree).toMatchSnapshot();
});
```

Snapshots are stored in `__tests__/__snapshots__` automatically.

**Code coverage** — enable in `package.json`:

```json
"jest": {
  "collectCoverage": true,
  "collectCoverageFrom": [
    "**/*.{ts,tsx,js,jsx}",
    "!**/coverage/**",
    "!**/node_modules/**"
  ]
}
```

View results by opening `coverage/lcov-report/index.html` in a browser.

---

## 6. E2E Testing with Maestro on EAS Workflows

Source: https://docs.expo.dev/eas/workflows/examples/e2e-tests/
(The previously documented path `https://docs.expo.dev/build-reference/e2e-tests/` now 404s; this is the current location.)

E2E tests require a built app file — `.apk` for Android or `.app` for iOS — that EAS can install and test on an emulator/simulator.

### Build profile configuration

Add an `e2e-test` build profile in `eas.json` (run `eas build:configure` first if the file doesn't exist):

```json
{
  "build": {
    "e2e-test": {
      "withoutCredentials": true,
      "ios": {
        "simulator": true
      },
      "android": {
        "buildType": "apk"
      }
    }
  }
}
```

### Maestro flow setup

Create a `.maestro` directory at the project root (same level as `eas.json`).

`.maestro/home.yml` — launches the app and verifies "Welcome!" text:

```yaml
appId: dev.expo.eastestsexample
---
- launchApp
- assertVisible: 'Welcome!'
```

`.maestro/expand_test.yml` — navigates and asserts UI elements:

```yaml
appId: dev.expo.eastestsexample
---
- launchApp
- tapOn: 'Explore.*'
- tapOn: '.*File-based routing'
- assertVisible: 'This app has two screens.*'
```

### EAS Workflow files

Create an `.eas/workflows` directory with workflow YAML files.

`.eas/workflows/e2e-test-android.yml`:

```yaml
name: e2e-test-android
on:
  pull_request:
    branches: ['*']
jobs:
  build_android_for_e2e:
    type: build
    params:
      platform: android
      profile: e2e-test
  maestro_test:
    needs: [build_android_for_e2e]
    type: maestro
    params:
      build_id: ${{ needs.build_android_for_e2e.outputs.build_id }}
      flow_path: ['.maestro/home.yml', '.maestro/expand_test.yml']
```

`.eas/workflows/e2e-test-ios.yml`:

```yaml
name: e2e-test-ios
on:
  pull_request:
    branches: ['*']
jobs:
  build_ios_for_e2e:
    type: build
    params:
      platform: ios
      profile: e2e-test
  maestro_test:
    needs: [build_ios_for_e2e]
    type: maestro
    params:
      build_id: ${{ needs.build_ios_for_e2e.outputs.build_id }}
      flow_path: ['.maestro/home.yml', '.maestro/expand_test.yml']
```

### Running tests

Manual execution:

```sh
npx eas-cli@latest workflow:run .eas/workflows/e2e-test-android.yml
```

Local testing (after installing the Maestro CLI):

```sh
maestro test .maestro/expand_test.yml
maestro test .maestro/home.yml
```

Automated runs trigger on pull requests via the `pull_request` trigger.

---

## 7. Expo Go for SDK 56 (NOT on the stores)

Sources:
- https://expo.dev/changelog/sdk-56
- https://expo.dev/changelog/expo-go-and-app-store-may-2026
- https://expo.dev/go

> **Expo Go for SDK 56 is not available on the Apple App Store or Google Play Store.** There is no current timeline for a store release; updates will be shared when available. Expo recommends migrating to **development builds** for apps beyond the learning phase.

### Install methods

**Android (via Expo CLI / expo.dev/go):**
- Install a specific version of Expo Go from [expo.dev/go](https://expo.dev/go), which supports downloads for Android devices/emulators and iOS simulators.

**iOS Simulator:**
- Download and install via Expo CLI, or from [expo.dev/go](https://expo.dev/go).
- Note: due to iOS platform restrictions, only the latest version of Expo Go is available for installation on **physical iOS devices**.

**iOS via TestFlight (external beta):**
- Join: https://testflight.apple.com/join/GZJxxfUU

**iOS via `eas go` (custom Expo Go build):**
- `eas go` is an EAS CLI command that "creates your own build of Expo Go that is then uploaded and made available to you through your Apple Developer account on TestFlight."
- Requires an [Apple Developer Program membership](https://developer.apple.com/programs/).
- Both SDK 55 and SDK 56 are supported with `eas go`.

### create-expo-app behavior in SDK 56

Starting with SDK 56, `create-expo-app` "will prompt developers to select whether they would like to initialize a project that is compatible with the App Store version of Expo Go, or to instead use the latest Expo SDK version."

You can pin a version using a template tag, e.g.:

```sh
bun create expo-app --template default@sdk-55
```

### Other SDK 56 highlights (from the changelog)

- The Jetpack Compose (Android) and SwiftUI (iOS) APIs in **Expo UI** are now stable, and Expo UI is included in the default `create-expo-app` template.
- SDK 56 ships **prebuilt XCFrameworks** for complex Expo modules on iOS to speed up iOS builds — enabled by default both locally and on EAS Build.
