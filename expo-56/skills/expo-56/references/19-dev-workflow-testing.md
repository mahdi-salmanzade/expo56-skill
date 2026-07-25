# Development Workflow, Debugging & Testing — Expo SDK 56

A knowledge-base reference covering development builds, the dev workflow, debugging tools, unit testing with jest-expo, and E2E testing with Maestro. Compiled from official Expo documentation. Source URLs are listed per section.

> **Doc versioning note:** every page this file draws from is **unversioned** in the Expo docs (`/develop/`, `/debugging/`, `/eas/`, `/eas-insights/`, `/get-started/`, `/workflow/`). There is no `/versions/v56.0.0/…` counterpart for them — do not invent one. The only genuinely SDK-scoped facts here are the package pins (`jest-expo`, `expo-dev-client`), the `expo prebuild` default, and which SDK the store build of Expo Go targets.

### Gotchas a model usually gets wrong

- **Expo Go on the App Store / Play Store targets SDK 54.** Neither SDK 56 nor SDK 57 has a store build. Error text: `"Project is incompatible with this version of Expo Go"`. See [§7](#7-expo-go-for-sdk-56-not-on-the-stores).
- **`npx expo-go download android latest`** is the supported way to fetch a matching Expo Go binary (Android device/Emulator + iOS Simulator only).
- **React Native Testing Library's `render` is async.** `const { getByText } = await render(<X />)` inside an `async` test. `react-test-renderer` is deprecated (no React 19 support) — remove it.
- **`jest-expo/universal` is the recommended preset**, not the bare `jest-expo`.
- **SDK 57: `npx expo prebuild` cleans by default** — it wipes and regenerates **android**/**ios**. Pass `--no-clean` for the old additive behaviour.
- **Node floor is 20.19.4 on both SDK 56 and SDK 57**, and it is a *warning*, not a hard failure. `@expo/cli` prints a red "Node.js (vX) is outdated and unsupported" line to stderr and keeps going. Neither `expo` nor `@expo/cli` ships an `engines` field on the 56 or 57 release lines, so there is no install-time `EBADENGINE` either. Do not claim SDK 57 raised the floor to 22.13 — that is SDK 58 work.

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
- **Release**: EAS Submit (`/deploy/submit-to-app-stores/`) or manual store submission
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
- **Android 16 (`Baklava`) SDK** — in **Settings > Languages & Frameworks > Android SDK > SDK Platforms**, select **Android SDK Platform 36** and **Sources for Android 36**
- Android SDK Build-Tools and Emulator (**SDK Tools** tab)
- `ANDROID_HOME` environment variable configured
- ADB (Android Debug Bridge) verification

**All iOS setups require:**
- Xcode
- Xcode Command Line Tools
- iOS Simulator components
- Watchman

**For local builds:**
- JDK 17 (Azul Zulu for macOS — `brew install --cask zulu@17`, then `JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home`; OpenJDK for Windows/Linux)
- Watchman

**For EAS builds:**

```sh
npm install -g eas-cli
```

Then sign up for an Expo account and log in to EAS.

### Key commands

Development build workflow:

```sh
eas build --platform [android|ios] --profile development
```

If your project doesn't have an **eas.json** yet, the EAS CLI prompts you to create one with a default `development` profile — `eas build:configure` is no longer presented as a required step here (it still is when hand-authoring an `e2e-test` profile, see §6).

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

Source: https://docs.expo.dev/develop/development-builds/introduction/?buildenv=build-with-eas#create-a-development-build-with-eas

> The old path `https://docs.expo.dev/develop/development-builds/create-a-build/` was deleted and now **301-redirects** to the URL above. The introduction page hosts all three build methods behind a `?buildenv=` selector.

**Three first-class build methods:**

| Method | Command | Notes |
| --- | --- | --- |
| **Build locally** | `npx expo run:android` / `npx expo run:ios` | Expo CLI + Android Studio/Xcode. No Expo account. The **only** way to install a dev build on an iPhone without a paid Apple Developer account. |
| **Build with EAS** | `eas build --platform … --profile development` | Cloud. No native toolchain needed; can build iOS from Windows/Linux. |
| **Build locally with EAS CLI** | `eas build --platform … --profile development --local` | Same EAS Build recipe on your machine; EAS still manages signing and profiles. |

Host-platform support for **Build locally**:

| | Android | iOS Simulator | iPhone device |
| --- | --- | --- | --- |
| **macOS** | yes | yes | yes |
| **Windows** | yes | no | no |
| **Linux** | yes | no | no |

#### Build with EAS

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

> **Gotcha:** on a hand-written EAS profile you must set `"developmentClient": true` on the build profile. Without it, EAS Build produces a standalone build with no development tools. (The default `development` profile the CLI generates already has it.) Source: https://docs.expo.dev/eas/json/ (`docs/pages/eas/json.mdx`) — the versioned `/versions/vNN.0.0/sdk/dev-client/` page does **not** mention `developmentClient`.

#### Build locally

```sh
npx expo install expo-dev-client
npx expo run:android          # installs on a running Android Emulator
npx expo run:android --device # physical device: plug in via USB, allow USB debugging
npx expo run:ios              # installs on the iOS Simulator
npx expo run:ios --device     # iPhone: enable iOS developer mode, needs a unique `ios.bundleIdentifier`
```

`expo run:*` runs prebuild to generate **android**/**ios** if they don't exist, compiles, installs, and starts the dev server. It **reuses** those directories on later runs.

> **SDK 56 patch drift — Xcode 27 / DeviceHub.** Xcode 27 replaces Simulator.app with **DeviceHub**. `npx expo run:ios` (and pressing `i`) falls back to `open devices://device/open?id=<udid>` when `open -a Simulator` fails, and the simulator prerequisite check accepts bundle id `com.apple.dt.Devices` alongside `com.apple.iphonesimulator` / `com.apple.CoreSimulator.SimulatorTrampoline`. This landed in `@expo/cli` **56.1.16**, i.e. **`expo` >= 56.0.12** — you do *not* need SDK 57 for it. Source: `@expo/cli` CHANGELOG 56.1.16 and 57.0.0+ (#46757).

> It is possible to use development builds *without* the `expo-dev-client` library. In that case start the dev server with `npx expo start --dev-client` so it targets your development build instead of Expo Go.

**Regenerate the native directories** only when you:
1. Install or update a library containing native code
2. Change your app config (**app.json**)
3. Upgrade the Expo SDK version

```sh
npx expo prebuild --clean   # then rebuild with expo run:android|ios
```

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

> **SDK 57:** `npx expo prebuild` now **clears and regenerates** the native folders by default. If you have hand-edited **android**/**ios**, use `npx expo prebuild -p android --no-clean`. See [SDK 57 delta](#sdk-57-delta).

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

**App crashes on certain (older) devices** — usually a performance issue rather than a logic bug. Profile the app (React Native DevTools Profiler, or React Native's profiling docs) to find what is getting the process killed.

### 4.2 Debugging Tools

Source: https://docs.expo.dev/debugging/tools/

**Developer menu** — open it:
- Press `m` in the terminal where Expo CLI started.
- **Android device**: shake vertically.
- **Android Emulator/USB**: `Cmd ⌘ + m` or `Ctrl + m`; or `adb shell input keyevent 82`.
- **iOS device**: shake or three-finger tap.
- **iOS Simulator/USB**: `Ctrl + Cmd ⌘ + z` or `Cmd ⌘ + d`.

Menu options: Copy link, Reload, Go Home; toggle performance monitor (RAM, JS heap, view counts, FPS); toggle element inspector (inspect elements, performance overlay, network details, highlight touchables); Open DevTools (Console, Sources, Network, Memory, Components, Profiler for Hermes apps); Fast Refresh toggle.

> **SDK 56 patch drift — two Android dev-menu Tools entries were broken and are fixed on the 56 line.** "Open React Native dev menu" (#47047) and the Fast Refresh toggle (#47136) were restored in `expo-dev-menu` **56.0.18**, which ships with `expo-dev-client` **>= 56.0.21**. Both are also in the 57 line; upgrading the SDK is not required, bumping `expo-dev-client` is.

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

> To profile the *native* runtime, use the tools in Android Studio or Xcode instead.

**Rozenite** (https://www.rozenite.dev/) — a React Native DevTools *plugin framework*. Plug-and-play integrations are auto-discovered and appear as extra panels inside React Native DevTools; you can also author your own plugin.

**Expo DevTools plugins** (source: https://docs.expo.dev/debugging/devtools-plugins/) — JS-only packages that open a two-way channel between your app and a browser panel. Install the package, call its hook from the root component, then `npx expo start` and press **Shift + M** to pick a plugin (it opens in a new Chrome window).

```sh
npx expo install @dev-plugins/react-navigation   # useReactNavigationDevTools
npx expo install @dev-plugins/apollo-client      # useApolloClientDevTools
npx expo install @dev-plugins/react-query        # useReactQueryDevTools
npx expo install redux-devtools-expo-dev-plugin  # default export: devToolsEnhancer
npx expo install @dev-plugins/tinybase
```

> Compatibility: because plugins are pure JavaScript they generally work in **both** Expo Go and development builds without a rebuild. The exception is when the underlying package being inspected has native code that isn't in Expo Go (e.g. React Native Firebase) — then you need a development build for both the library and its plugin.

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

**Installation** (`jest-expo` is pinned at `~56.0.5` in SDK 56 — `packages/expo/bundledNativeModules.json` on `sdk-56`; earlier 56 patches resolved `~56.0.4`):

```sh
npx expo install jest-expo jest @types/jest --dev
npx expo install @testing-library/react-native --dev
```

On Windows the extra flags need an escape hatch: `npx expo install jest-expo jest @types/jest "--" --dev`.

> **Deprecated:** `@testing-library/react-native` replaces `react-test-renderer`, which does **not** support React 19+. Remove `react-test-renderer` from your project if it is still there.

> **Peer dependency:** `jest-expo` peer-depends on `@react-native/jest-preset` (SDK 56 range `^0.85.0`) and optionally on `react-server-dom-webpack` (`~19.0.4 || ~19.1.5 || ~19.2.4`). If Jest fails to resolve the React Native preset, install `@react-native/jest-preset` matching your React Native version. (`jest-expo` switched from `react-native/jest-preset` to `@react-native/jest-preset` in **56.0.1**.)

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

**Presets** — `jest-expo` (bare) runs tests in the standard React Native (iOS) environment for legacy reasons. The **recommended** preset is `jest-expo/universal`, which runs every test against iOS, Android, web, and Node (SSR) runners. Press **X** in watch mode to open the platform-selection dialog.

| Preset | Environment |
| --- | --- |
| `jest-expo/universal` | iOS + Android + web + Node — recommended |
| `jest-expo/ios` | Mock native environment (iOS) |
| `jest-expo/android` | Mock native environment (Android) |
| `jest-expo/web` | JSDOM, for Expo web |
| `jest-expo/node` | Node, for SSR compliance |
| `jest-expo/rsc` | React Server Components |

Under `universal`, snapshots are written per platform: `__snapshots__/View-test.tsx.snap.ios`, `.snap.android`, `.snap.web`, `.snap.node`. Platform-specific test extensions: `-test.ios.*` / `-test.native.*`, `-test.android.*` / `-test.native.*`, `-test.web.*`, `-test.node.*` / `-test.web.*`.

Mix runners with Jest `projects` (single-runner presets only):

```json
"jest": {
  "projects": [
    { "preset": "jest-expo/ios" },
    { "preset": "jest-expo/android" }
  ]
}
```

To build a custom preset, use `jest-expo/config`: `getWatchPlugins(jestConfig)`, `withWatchPlugins(jestConfig)`, `getWebPreset()`, `getIOSPreset()`, `getAndroidPreset()`, `getNodePreset()`.

For projects using multiple npm packages, configure `transformIgnorePatterns`. **The regex differs per package manager** — pnpm and Bun need an extra alternative and different parenthesis nesting:

```json
// npm / Yarn
"transformIgnorePatterns": [
  "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg)"
]

// pnpm
"transformIgnorePatterns": [
  "node_modules/(?!(.pnpm|(jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg))"
]

// Bun
"transformIgnorePatterns": [
  "node_modules/(?!(.bun|(jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg))"
]
```

**Writing unit tests** — create a `__tests__` directory; test files use `-test.ts|tsx` extensions:

`render` from React Native Testing Library is **async** — `await` it inside an `async` test:

```tsx
import { render } from '@testing-library/react-native';
import HomeScreen, { CustomText } from '@/app/index';

describe('<HomeScreen />', () => {
  test('Text renders correctly on HomeScreen', async () => {
    const { getByText } = await render(<HomeScreen />);
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
test('CustomText renders correctly', async () => {
  const tree = (await render(<CustomText>Some text</CustomText>)).toJSON();
  expect(tree).toMatchSnapshot();
});
```

Snapshots are stored in `__tests__/__snapshots__` automatically.

> For UI testing, Expo recommends E2E tests over snapshot unit tests — see §6.

**Code coverage** — enable in `package.json`:

```json
"jest": {
  "collectCoverage": true,
  "collectCoverageFrom": [
    "**/*.{ts,tsx,js,jsx}",
    "!**/coverage/**",
    "!**/node_modules/**",
    "!**/babel.config.js",
    "!**/expo-env.d.ts",
    "!**/.expo/**"
  ]
}
```

View results by opening `coverage/lcov-report/index.html` in a browser. Add `coverage/**/*` to **.gitignore**.

**Jest flows (optional)** — documented script set:

```json
"scripts": {
  "test": "jest --watch --coverage=false --changedSince=origin/main",
  "testDebug": "jest -o --watch --coverage=false",
  "testFinal": "jest",
  "updateSnapshots": "jest -u --coverage=false"
}
```

**Testing Expo Router route trees** — for integration tests that render a whole route tree rather than a bare component, use `renderRouter` and the custom Jest matchers from `expo-router/testing-library`. Source: https://docs.expo.dev/router/reference/testing/

---

## 6. E2E Testing with Maestro on EAS Workflows

Sources:
- https://docs.expo.dev/eas/workflows/examples/e2e-tests/ (the older path `https://docs.expo.dev/build-reference/e2e-tests/` **301-redirects** here — it does not 404)
- https://docs.expo.dev/eas/workflows/pre-packaged-jobs/#maestro
- https://docs.expo.dev/eas-insights/maestro/

> **Maestro tests on EAS Workflows are in alpha.**

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

### The `maestro` job in full

The two-key form above (`build_id`, `flow_path`) is the minimum. The full surface:

```yaml
jobs:
  run_maestro_tests:
    type: maestro
    environment: production | preview | development  # optional — defaults to preview
    image: string                                   # optional — see EAS build infrastructure
    runs_on: string                                 # optional — Android Emulator tests need a
                                                    #   linux-*-nested-virtualization worker
    params:
      build_id: string                    # required
      flow_path: string | string[]        # required — flow file(s) or a directory
      shards: number                      # optional, experimental — default 1
      retries: number                     # optional — default 0
      retry_failed_only: boolean          # optional — default true
      record_screen: boolean              # optional — default false (costs emulator perf)
      include_tags: string | string[]     # optional — Maestro --include-tags
      exclude_tags: string | string[]     # optional — Maestro --exclude-tags
      maestro_version: string             # optional — defaults to latest
      android_system_image_package: string # optional — e.g. system-images;android-36;google_apis;x86_64
      device_identifier: string | { android?: string, ios?: string }  # e.g. pixel_6 / "iPhone 16 Plus"
      output_format: string               # optional — default junit; Maestro --format
      skip_build_check: boolean           # optional — default false
    hooks:
      after_checkout: step[]
      before_maestro_tests: step[]
      after_maestro_tests: step[]
```

Notes:
- `runs_on` matters: Android Emulator runs require a **nested-virtualization** Linux worker.
- `device_identifier` accepts a single-value expression, e.g. `${{ needs.build.outputs.platform == "android" ? "pixel_6" : "iPhone 16 Plus" }}`. Available iOS devices differ per runner image and are listed in the job logs.
- `skip_build_check` bypasses the validation that an iOS build is a simulator build.

### Maestro insights

The **Maestro** tab in the EAS dashboard (`/eas-insights/maestro/`) tracks per-flow **Runs, Pass rate, Fails, Flake rate, P90** duration and last run, plus per-flow drill-down. **Production and Enterprise plans only.**

> **Gotcha:** insights are collected from the JUnit report the `maestro` job emits by default (`output_format: junit`). Setting `output_format` to anything else (e.g. `html`) silently means no results are reported.

### Running tests

Manual execution:

```sh
eas workflow:run .eas/workflows/e2e-test-android.yml
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
- https://docs.expo.dev/develop/tools/#expo-go-cli
- https://docs.expo.dev/develop/development-builds/faq/
- https://expo.dev/go

> **The App Store / Play Store version of Expo Go targets SDK 54.** Expo Go supports exactly one SDK version at a time, and only that version is installable from the stores. **Neither SDK 56 nor SDK 57 has a store build.** Opening a newer project in the store build gives:
>
> ```text
> "Project is incompatible with this version of Expo Go"
> ```
>
> Fix by upgrading the project to the store SDK, or by fetching a matching Expo Go with the `expo-go` CLI. Expo recommends migrating to **development builds** for anything beyond the learning phase.

### Install methods

**`expo-go` CLI** (the documented way to pin an Expo Go version):

```sh
npx expo-go download android latest   # downloads the binary into the CWD, caches under ~/.expo
npx expo-go url ios latest            # prints the download URL instead of downloading
```

Pass an SDK version in place of `latest` to pin a specific release; omit it for the latest. (Also available as `yarn dlx expo-go …`, `pnpm dlx expo-go …`, `bunx expo-go …`.)

> **Warning:** the `expo-go` CLI works for **Android device, Android Emulator, and iOS Simulator only**. Apple does not permit side-loading older app versions, so an iPhone device is not supported.

**expo.dev/go** — the same binaries, downloadable from the browser (Android device/emulator and iOS Simulator).

**iOS device — build your own Expo Go and ship it via your own TestFlight:**
1. Hold an active [Apple Developer Program](https://developer.apple.com/programs/) subscription.
2. Build Expo Go: `npx eas-cli@latest go`
3. Install the [TestFlight app](https://apps.apple.com/us/app/testflight/id899247664).
4. In [App Store Connect](https://appstoreconnect.apple.com), select the Expo Go app > **TestFlight** tab > add your Apple ID email as an **internal tester**, then accept the emailed invite.

> This is *your own* App Store Connect record, not a public beta. There is no public `testflight.apple.com/join/…` link for Expo Go in the docs — treat any such URL as unverified.

### create-expo-app SDK selection

`create-expo-app` prompts `Select an Expo SDK version:` with the choices:

| Choice | Description | Shown when |
| --- | --- | --- |
| `Latest (SDK N)` | "Recommended for most projects" | always |
| `For learning with Expo Go (SDK M)` | "Compatible with Expo Go on App Store and Play Store" | only when the store-compatible SDK differs from latest |
| `Other SDK version…` | picks from the last 4 released majors | always |

The prompt is **skipped** when: `EXPO_BETA` is set; the `--template` value already carries an `@sdk-*` tag; the template is not a known Expo template; or the run is non-interactive (`--yes`, `CI`, or no TTY). In the non-interactive + default-template case the template is pinned to `@sdk-<latest>`. Source: `packages/create-expo/src/promptSdkVersion.ts`.

> The docs note at `/get-started/create-a-project/` says that `create-expo-app@latest` *without* `--template` creates an SDK 54 project during the transition. That conflicts with the non-interactive code path above (which pins to latest) — pass `--template` explicitly rather than relying on either.

Pin a version with a template tag (note the package name differs per manager):

```sh
npx create-expo-app@latest --template default@sdk-56
yarn create expo-app --template default@sdk-56
pnpm create expo-app --template default@sdk-56
bun create expo --template default@sdk-56          # `bun create expo`, NOT `bun create expo-app`
```

### Other SDK 56 highlights (from the changelog)

- The Jetpack Compose (Android) and SwiftUI (iOS) APIs in **Expo UI** are now stable, and Expo UI is included in the default `create-expo-app` template.
- SDK 56 ships **prebuilt XCFrameworks** for complex Expo modules on iOS to speed up iOS builds — enabled by default both locally and on EAS Build.

---

## SDK 57 delta

SDK 57 (`expo@57.0.0` published 2026-06-30; `57.0.8` is the current patch) changes this domain in two ways that matter: one destructive `expo prebuild` default, and a new DevTools-plugin server side. Everything else in this file — Maestro job syntax, React Native DevTools, Jest authoring, `adb logcat`, VS Code debugging, the Node floor, the Expo Go store situation — is unchanged.

> Reminder: the pages this file draws from are unversioned, so there is no `v56.0.0` vs `v57.0.0` docs diff to consult. Verify SDK-scoped claims against the **release branches** (`git show origin/sdk-57:<path>`), `packages/expo/bundledNativeModules.json` on those branches, and published npm tarballs. Do **not** use `main` (that is SDK 58 in progress), `## Unpublished` changelog sections, or `docs/public/static/schemas/v5{6,7}.0.0/native-modules.json` (stale).

### Breaking

| Change | Detail |
| --- | --- |
| **`npx expo prebuild` cleans by default** | It now **clears and regenerates** the native folders. Pass `--no-clean` to apply changes to the existing folders instead (the SDK 56 behaviour). Directly affects §4.1's "prebuild, open in Android Studio, delete the folder afterward" recipe — on 57 it will wipe hand-edited native code. Landed in `@expo/cli` **57.0.0**, so it applies to every 57 patch. Source: `git show origin/sdk-57:packages/@expo/cli/CHANGELOG.md` → 57.0.0 🛠 Breaking changes (#47209); `packages/@expo/cli/src/prebuild/index.ts` (`clean: !args['--no-clean']`). |

That is the entire breaking-change surface for this domain. In particular:

- **The Node floor did NOT move.** `NODE_MIN` is `[20, 19, 4]` in the published `@expo/cli@56.1.21` *and* `@expo/cli@57.0.10` bundles, and it only drives a stderr warning (`"Node.js (vX) is outdated and unsupported"`), never a non-zero exit. `git show origin/sdk-57:packages/expo/package.json` has **no `engines` field**, so there is no `EBADENGINE` on install either, and `create-expo` on `sdk-57` still declares `"node": ">=18.13.0"`. The `^22.13.0 || ^24.3.0 || …` engines bump exists only on `main` — it is post-57 (SDK 58) and must not be presented as part of a 56 → 57 upgrade.
- **There is no `jest-expo` Babel breaking change in 57.** `git show origin/sdk-57:packages/jest-expo/CHANGELOG.md` records 57.0.0 as *"This version does not introduce any user-facing changes."* Snapshots do not need re-recording because of a 57 Babel realignment. (The real `jest-expo` Babel-config alignment, #45968, shipped in **56.0.4** — on the SDK 56 line.)

### New in 57

- **DevTools plugins gained a server side** (not yet in `docs/pages/debugging/create-devtools-plugins.mdx`). A plugin can declare `serverEntryPoint` — HTTP + WebSocket handlers that run inside the Expo CLI process — and `bannerTitle` (boolean or string, shown in the CLI startup banner). Shipped in `@expo/cli` **57.0.5**, i.e. **`expo` >= 57.0.3**; not present in 57.0.0–57.0.2. Verified in the published `@expo/cli@57.0.10` tarball (`build/src/start/server/DevToolsPlugin.schema.js`, `…/DevToolsPluginServerHelpers.js`). Source: `git show origin/sdk-57:packages/@expo/cli/CHANGELOG.md` → 57.0.5 (#46764).

  ```ts
  type DevToolsPluginRequestHandler = (
    request: Request
  ) => Response | null | undefined | Promise<Response | null | undefined>;

  type DevToolsPluginWebSocketHandler = (
    socket: WebSocket,
    request: Request,
    server: WebSocketServer
  ) => void;
  ```

  The request handler is the module's **default export**; returning `null`/`undefined` falls through to static `webpageRoot` serving. WebSocket handlers are exported as a `webSocketHandlers` record keyed by route, and receive a fetch API `Request` (built from the Node `IncomingMessage` via `convertUpgradeRequest`) — read `request.url` / `request.headers` (a `Headers` object), not Node properties. Both modules are loaded via `loadModule` from the `@expo/require-utils` package. The `Request` shape and `loadModule` landed in the *same* release as the feature itself (`@expo/cli` 57.0.5, #47410 / #47139), so there is no migration for plugin authors — only the one API to learn.

- **Launcher changes visible in §3.3** (`expo-dev-launcher` **57.0.1**, so `expo-dev-client` >= 57.0.1):
  - Settings toggle to auto-launch the most recent app instead of always showing the launcher, backed by a `tryToLaunchLastBundle` preference (#47131). Includes the Android fix for actually auto-launching the most recently opened project on startup.
  - iOS dev-server list made reliable: discovery keeps running across tab switches, discovered servers are periodically re-verified, pull-to-refresh on the Home tab, and a "searching" state instead of "No development servers found" (#46811).
  - iOS Settings gained the build's expiration date (#47190); failed project loads now explain why instead of a generic "Failed to connect" (#46866).

  Source: `git show origin/sdk-57:packages/expo-dev-launcher/CHANGELOG.md` → 57.0.1. All of these are absent from `origin/sdk-56`'s changelog, so they are genuine 57 deltas.

> **Do NOT attribute to SDK 57** (all verified absent from every `CHANGELOG.md` on `origin/sdk-57` — they live only in `## Unpublished` on `main`, i.e. SDK 58):
> - Android dev-launcher forwarding the launching intent's extras so Maestro/Detox/`adb am start -e` launch arguments reach the app (#47352). On SDK 57 a cold launch through the launcher still drops them.
> - The dev-menu **Components** section for swapping the active `AppRegistry` component (#46613).
> - Android packager discovery across all connected networks on Android 33+ (#46487).
> - The `expo-dev-menu` / `expo-dev-launcher` legacy-architecture (bridge) removal and the RN 0.86 public-API migration (#47637–#47640).

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 |
| --- | --- | --- |
| `jest-expo` | `~56.0.5` | `~57.0.2` |
| `expo-dev-client` | `~56.0.24` | `~57.0.8` (`origin/sdk-57` HEAD already reads `~57.0.9`) |
| `react-native` | `0.85.3` | `0.86.0` |
| Node (minimum) | `20.19.4` (warning only) | `20.19.4` (warning only) — **unchanged** |

Source: `bundledNativeModules.json` shipped inside the published `expo@56.0.17` and `expo@57.0.8` tarballs, cross-checked against `git show origin/sdk-5{6,7}:packages/expo/bundledNativeModules.json`. That file is what `expo install` resolves against. Do **not** read pins out of `docs/public/static/schemas/v5{6,7}.0.0/native-modules.json`; those schemas are stale (they still say `react-native-screens` 4.25.2 when both SDK 56 and 57 actually pin `~4.26.0`). Node floor from `NODE_MIN` in the published `@expo/cli` bundles.

Note that the first-party `expo-*` pins are *not* a flat `~57.0.0` — each package has its own patch cadence, so always resolve through `npx expo install` rather than hand-writing versions.

**`jest-expo` peer range (resolved):** `jest-expo@57.0.2` peer-depends on `@react-native/jest-preset` `^0.86.0` (56.0.5 requires `^0.85.0`). The range was wrong at 57.0.0 and fixed in `jest-expo` **57.0.1** (#47440), so on 57 use `jest-expo` >= 57.0.1. Verified with `npm view jest-expo@57.0.2 peerDependencies`.

**Checked and unchanged in 57:** the Maestro EAS Workflows job (params, hooks, `runs_on`), Maestro insights, React Native DevTools, the dev menu shortcuts, `adb logcat` / Console.app native logging, Jest test authoring and `transformIgnorePatterns`, the `jest-expo` preset list, the `expo-go` CLI, the Node floor, and the Expo Go store situation (still SDK 54; no store build for 56 or 57).

**Already available on the SDK 56 line — do not upgrade for these:** Xcode 27 / DeviceHub support (`@expo/cli` 56.1.16 → `expo` >= 56.0.12, see §3.2), and the restored Android dev-menu Tools entries (`expo-dev-menu` 56.0.18 → `expo-dev-client` >= 56.0.21, see §4.2). SDK 56 is still actively patched (`expo@56.0.17` at time of writing), so check `origin/sdk-56` before treating any fix as a 57-only feature.
