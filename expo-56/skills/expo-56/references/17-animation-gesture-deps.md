# Core Native Dependencies: Animation, Gesture & Navigation (Expo SDK 56)

Reference for the native libraries Expo SDK 56 bundles versions of and that ship with most apps: animation (Reanimated), gestures (Gesture Handler), navigation primitives (Screens), safe area handling (Safe Area Context), and vector graphics (SVG).

> Note on dates: originally assembled 2026-05-22; re-verified 2026-07-25 against the `expo/expo` monorepo **release branches** `origin/sdk-56` and `origin/sdk-57`, and against published npm tarballs. Version pins are taken from `packages/expo/bundledNativeModules.json` **on the release branch** (`git show origin/sdk-57:packages/expo/bundledNativeModules.json`) — that file ships inside the `expo` package and is what `expo install` resolves against. API claims come from `docs/pages/versions/v56.0.0/sdk/*.mdx` on `origin/sdk-56` and package source on the release branches. Primary scope is SDK 56; SDK 57 differences are collected in [SDK 57 delta](#sdk-57-delta) at the end.

## What models get wrong here

1. **Reanimated 4 needs two packages.** `react-native-reanimated` alone is not enough — `react-native-worklets` must be installed too, on both SDK 56 and 57.
2. **The gesture API is `Gesture.Pan()` + `GestureDetector`.** The `usePanGesture()` / `useRotationGesture()` hooks are gesture-handler **3.x** and are in neither SDK 56 nor SDK 57.
3. **Reanimated and worklets are precompiled together on EAS Build.** If you need a source build of either, you must list **both** in `expo.autolinking.ios.buildFromSource`. Opting out of only one produces a mixed linkage that fails at runtime.
4. **Do not read version pins off the monorepo `main` branch, and do not trust the frozen docs schemas.** `main` tracks the *unversioned* (post-57) line. And `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json` are **stale**: they claim `react-native-screens` `4.25.2` and `expo-router` `~56.2.9` / `~57.0.0`, where the shipped values are `~4.26.0` and `~56.2.16` / `~57.0.8`. The only good pin source is `packages/expo/bundledNativeModules.json` on the release branch.

## SDK bundled versions

Pins from `packages/expo/bundledNativeModules.json` on `origin/sdk-56` and `origin/sdk-57`. That file ships inside the `expo` package, so it is what `expo install` resolves against.

| Library | SDK 56 | SDK 57 |
| --- | --- | --- |
| react-native-reanimated | `4.3.1` | `4.5.0` |
| react-native-worklets | `0.8.3` | `0.10.0` |
| react-native-gesture-handler | `~2.31.1` | `~2.32.0` |
| react-native-screens | `~4.26.0` | `~4.26.0` (unchanged) |
| react-native-safe-area-context | `~5.7.0` | `~5.7.0` (unchanged) |
| react-native-svg | `15.15.4` | `15.15.4` (unchanged) |
| react-native (context) | `0.85.3` | `0.86.0` |

> `react-native-screens` is `~4.26.0` on **both** lines. The frozen docs schema saying `4.25.2` is out of date; don't propagate it.

Adjacent same-domain pins from the same file: `lottie-react-native` `~7.3.4` → `~7.3.8`, `react-native-pager-view` `8.0.1` → `8.0.2`, `react-native-keyboard-controller` `1.21.6` → `1.21.9`; unchanged across both — `@shopify/react-native-skia` `2.6.2`, `react-native-webview` `13.16.1`, `@react-native-masked-view/masked-view` `0.3.2`.

Always install via `npx expo install <package>` so the SDK-pinned version is resolved rather than the latest npm release.

> **Where pins actually come from.** `npx expo install` queries the `/sdks/:sdkVersion/native-modules` endpoint first and only falls back to the local `expo/bundledNativeModules.json` when offline or when the SDK isn't published yet (`packages/@expo/cli/src/start/doctor/dependencies/bundledNativeModules.ts`, where the local reader is labelled "legacy static"). The fallback is only "legacy" in the sense that it is offline — the copy shipped in the installed `expo` package is correct for your SDK. What is *not* correct is reading that JSON off the monorepo's `main` branch: `main` is the post-57 unversioned track and at the time of writing reads reanimated `4.5.1`, worklets `0.10.1`, gesture-handler `~3.1.0`, screens `4.26.0` — neither SDK 56 nor SDK 57.

---

## EAS Build precompiled artifacts

EAS Build precompiles commonly used React Native community libraries into XCFrameworks to accelerate native builds. As of SDK 56 the third-party precompile set on iOS is **seven** packages (`packages/expo-modules-autolinking/external-configs/ios/`, all added in commit `9905ff771c8`, 2026-03-28):

- `@react-native-async-storage/async-storage`
- `@shopify/react-native-skia`
- `react-native-reanimated`
- `react-native-safe-area-context`
- `react-native-screens`
- `react-native-svg`
- `react-native-worklets`

The **membership** is identical on `origin/sdk-57` — no libraries were added or removed. The contents are not: `react-native-reanimated`'s and `react-native-worklets`' `spm.config.json` both changed (see [SDK 57 delta](#sdk-57-delta)).

> **Do not trust `external-configs/ios/README.md`.** Its table (data rows on lines 19-28) lists ten packages including `lottie-react-native`, `react-native-gesture-handler` and `react-native-keyboard-controller`. Those three have no config directory and never did — the README is aspirational. Only the seven directories above exist. Verified against the published `expo-modules-autolinking@56.0.21` and `@57.0.9` tarballs, which ship exactly those seven `external-configs/ios/*/spm.config.json` files.

Reported impact: precompiling these "cuts median iOS clean build times on EAS Build by another ~1 minute (~20%) on top of the Expo-modules precompile", itself on top of the earlier ~1 minute (~16%) from precompiling the Expo packages. No configuration change is required for the happy path.

### buildFromSource: the operational contract

**Sourcing, so you know how far to trust each rule.** Rule 1 is Expo-documented: `docs/pages/guides/prebuilt-expo-modules.mdx`, byte-identical on `origin/sdk-56` and `origin/sdk-57` (53 lines — it covers `EXPO_USE_PRECOMPILED_MODULES` and `buildFromSource`, and nothing below). Rules 2–4 are **not in Expo's docs at all** — `staticFeatureFlags`, the reanimated/worklets pairing requirement, and the `Unable to recognize flag` diagnostic return zero hits across the whole `origin/sdk-57` tree. They come from the Reanimated/Worklets packages themselves; `staticFeatureFlags` is confirmed real in `react-native-worklets@0.10.0` (`android/build.gradle.kts`, `scripts/worklets_utils.rb`, `src/featureFlags/staticFlags.json`). Treat 2–4 as field-tested operational guidance rather than vendor-documented contract, and confirm against the Reanimated docs if a build hangs on it.

Four rules, in priority order:

1. **Third-party precompiled artifacts are an EAS-Build-only path.** On EAS Build, `react-native-reanimated` and `react-native-worklets` are downloaded as precompiled XCFrameworks. Local `pod install` does not fetch them — they build from source locally and pick up `staticFeatureFlags` overrides automatically. **Flag and version mismatches therefore surface on EAS Build and not locally.** Note this rule is about *third-party* XCFrameworks only: Expo's *own* modules are precompiled by default on iOS locally too, since SDK 56 (`prebuilt-expo-modules.mdx`: "**iOS**: enabled by default in SDK 56+. In SDK 55, enabled by default only on EAS Build"). That is why rule 3's `EXPO_USE_PRECOMPILED_MODULES=0` is meaningful locally.
2. **Reanimated and worklets must be opted out together.** Reanimated links worklets at the native level. Source-building only one produces a mixed precompiled/source linkage that fails to resolve the matching framework at runtime.

   ```json package.json
   {
     "expo": {
       "autolinking": {
         "ios": {
           "buildFromSource": ["react-native-reanimated", "react-native-worklets"]
         }
       }
     }
   }
   ```

   Use `[".*"]` to opt out of every precompiled module. The same key exists under `android`.
3. **Custom feature flags require a source build.** Flag values are baked into the precompiled binary at build time, so `worklets.staticFeatureFlags` / `reanimated.staticFeatureFlags` in **package.json** are ignored by a precompiled artifact. To apply them, set `EXPO_USE_PRECOMPILED_MODULES=0`.
4. **Diagnostic symptom.** A runtime `Unable to recognize flag: <NAME>` that appears on EAS Build but not locally means the precompiled artifact's flag list doesn't match your pinned package version. Fix with `buildFromSource` for both packages.

---

## Worklets: reanimated vs react-native-worklets

Reanimated 4 split the worklets runtime out into a standalone `react-native-worklets` package. Install both:

```
npx expo install react-native-reanimated react-native-worklets
```

A worklet is a JavaScript function that a Babel plugin transforms so it can be serialized to and run on the UI thread instead of the JS thread. That is what lets animation and gesture callbacks execute synchronously with rendering.

### Babel wiring

`babel-preset-expo` configures the plugin automatically — "Reanimated Babel plugin is automatically configured in `babel-preset-expo` when you install the library" (`docs/pages/versions/v56.0.0/sdk/reanimated.mdx`). Two per-platform preset options control it, both defaulting to `true` (`packages/babel-preset-expo/src/index.ts:35-40`):

- `reanimated?: boolean`
- `worklets?: boolean`

Resolution order (`packages/babel-preset-expo/src/configs/expo.ts:157-168`):

- if **neither** option is `false`, it resolves `react-native-worklets/plugin` and pushes it;
- else if `reanimated !== false`, it falls back to `react-native-reanimated/plugin`;
- nothing is pushed if the corresponding package isn't installed.

This block is unchanged between `origin/sdk-56` and `origin/sdk-57`, down to the line numbers. You only need to touch these options to *disable* the transform.

### SDK 56 patch drift — worklets changes that are NOT a reason to upgrade to 57

These three landed on **both** release lines. If you are on SDK 56 you get them with a patch bump; do not upgrade the SDK for them.

- **`@expo/ui` dropped its `react-native-reanimated` peer dependency.** Needs **`@expo/ui` >= 56.0.19** (verified: `npm view @expo/ui@56.0.18 peerDependencies` still lists `react-native-reanimated`; `@56.0.19` does not). Practical rule on either line: `@expo/ui` worklet props require `react-native-worklets` installed *and natively linked*, but no longer require reanimated. Source: `packages/expo-ui/CHANGELOG.md` under `## 56.0.19` (#46922, #46935) — the identical entry is in `origin/sdk-57`'s changelog history.
- **Worklet UI-runtime resolution rewired.** `expo-modules-core` resolves the worklet UI runtime from the `react-native-worklets` holder instead of reanimated's `_WORKLET_RUNTIME` global; `installOnUIRuntime` takes the holder from `getUIRuntimeHolder()`. This is what allowed the reanimated peer to be dropped, and it breaks hand-rolled native modules that reached for `_WORKLET_RUNTIME`. Needs **`expo-modules-core` >= 56.0.18** — verified against the published tarballs: `getUIRuntimeHolder` is absent from `expo-modules-core@56.0.17` and present from `@56.0.18` on. (`packages/expo-modules-core/CHANGELOG.md` under `## 56.0.18` on `origin/sdk-56`, #46922, #46935.) Cross-reference: `04-expo-modules-api.md`.
- **Better failure mode for an unlinked worklets adapter.** iOS throws an actionable error when a worklet is used but `react-native-worklets`'s native adapter isn't linked, instead of the misleading "not an instance of Worklet". Needs **`expo-modules-core` >= 56.0.15** (`## 56.0.15`, #46571).

---

## react-native-reanimated

Declarative animation library running animations on the UI thread (targets 120+ fps) across iOS, Android, and web. SDK 56 and 57 both bundle the **Reanimated 4** major line (see the pin table).

### useSharedValue

A shared value is the core driver of animations; mutating `.value` can be observed on the UI thread.

```javascript
import { View, Button } from 'react-native';
import Animated, { useSharedValue } from 'react-native-reanimated';

export default function App() {
  const width = useSharedValue(100);

  const handlePress = () => {
    width.value = Math.random() * 100 + 50;
  };

  return (
    <View style={{ flex: 1, alignItems: 'center' }}>
      <Animated.View
        style={{
          width,
          height: 100,
          backgroundColor: 'violet',
        }}
      />
      <Button onPress={handlePress} title="Click me" />
    </View>
  );
}
```

### useAnimatedStyle

Derives a style object from shared values on the UI thread; pass the result into an `Animated.View` style array.

```javascript
import { StyleSheet } from 'react-native';
import Animated, { useAnimatedStyle, useSharedValue } from 'react-native-reanimated';

function App() {
  const sv = useSharedValue(0);

  const animatedStyles = useAnimatedStyle(() => {
    return {
      opacity: sv.value ? 1 : 0,
    };
  });

  return <Animated.View style={[styles.box, animatedStyles]} />;
}

const styles = StyleSheet.create({
  box: { width: 100, height: 80, backgroundColor: 'black' },
});
```

Anti-pattern — do not mutate a shared value inside the `useAnimatedStyle` callback:

```javascript
function App() {
  const sv = useSharedValue(0);
  const animatedStyles = useAnimatedStyle(() => {
    sv.value = withTiming(1); // Don't do this!
    return { opacity: sv.value };
  });
}
```

### withSpring

Animates a shared value toward a target with spring physics.

```javascript
import { View, Button } from 'react-native';
import Animated, { useSharedValue, withSpring } from 'react-native-reanimated';

export default function App() {
  const width = useSharedValue(100);

  const handlePress = () => {
    width.value = withSpring(Math.random() * 100 + 50);
  };

  return (
    <View style={{ flex: 1, alignItems: 'center' }}>
      <Animated.View
        style={{
          width,
          height: 100,
          backgroundColor: 'violet',
        }}
      />
      <Button onPress={handlePress} title="Click me" />
    </View>
  );
}
```

### withTiming

Time/easing-based modifier. Takes an optional config object with `duration` and `easing`. This is Expo's own canonical example (`docs/pages/versions/v56.0.0/sdk/reanimated.mdx`, Usage) and is identical for SDK 57:

```javascript
import { View, Button, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  withTiming,
  useAnimatedStyle,
  Easing,
} from 'react-native-reanimated';

export default function AnimatedStyleUpdateExample() {
  const randomWidth = useSharedValue(10);

  const config = {
    duration: 500,
    easing: Easing.bezier(0.5, 0.01, 0, 1),
  };

  const style = useAnimatedStyle(() => {
    return {
      width: withTiming(randomWidth.value, config),
    };
  });

  return (
    <View style={styles.container}>
      <Animated.View style={[styles.box, style]} />
      <Button
        title="toggle"
        onPress={() => {
          randomWidth.value = Math.random() * 350;
        }}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  box: { width: 100, height: 80, backgroundColor: 'black', margin: 30 },
});
```

### Layout animations

Declarative entering/exiting animations applied directly on `Animated.View`:

```javascript
<Animated.View entering={FadeIn} exiting={FadeOut} />
```

### Other notable hooks

- `useAnimatedKeyboard()` — track keyboard height/state as a shared value:

```javascript
const keyboard = useAnimatedKeyboard();
const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateY: -keyboard.height.value }],
}));
```

- `useAnimatedSensor()` — read device sensors on the UI thread.

### Debugging caveat

Reanimated uses React Native APIs incompatible with "Remote JS Debugging" for JavaScriptCore. Use Hermes plus the JavaScript Inspector for Hermes (`docs/pages/versions/v56.0.0/sdk/reanimated.mdx`).

---

## react-native-gesture-handler

Declarative, native-driven touch/gesture system. Integrates tightly with Reanimated to allow smooth gesture-based experiences up to 120 fps.

> **API version warning — applies to BOTH SDK 56 (`~2.31.1`) and SDK 57 (`~2.32.0`).** The correct API is `Gesture.<Type>()` + `GestureDetector`. The `usePanGesture()` / `useRotationGesture()` hooks that the library's live docs site now leads with are **gesture-handler 3.x** and exist in neither SDK. Gesture-handler 3.x first entered the monorepo on 2026-07-14 in commit `70d4ef85511` ("[expo] Update react-native-gesture-handler to 3.0.2"), which is not on `origin/sdk-57` — it targets SDK 58. Do not "modernise" the builder API below.

### Setup

Wrap the app root in `GestureHandlerRootView`, define a gesture, and connect it to a view via `GestureDetector`. The core pattern: "wrap your target view with the GestureDetector component, define your gesture, and pass it in."

### Available gestures (Gesture API)

`Gesture.Pan()`, `Gesture.Tap()`, `Gesture.LongPress()`, `Gesture.Fling()`, `Gesture.Pinch()`, `Gesture.Rotation()`, `Gesture.ForceTouch()`. Gestures can be composed (e.g. `Gesture.Simultaneous(...)`, `Gesture.Race(...)`, `Gesture.Exclusive(...)`).

### Gesture callbacks

`onBegin`, `onStart`, `onUpdate`, `onChange`, `onEnd`, `onFinalize`. For a pan gesture, the update event exposes `translationX`/`translationY` (relative to start) and `changeX`/`changeY` (delta since last event). Store the element's start position in a shared value because only translation relative to the gesture start is provided.

### Draggable ball (SDK 56 / SDK 57 API)

```javascript
import { StyleSheet } from 'react-native';
import {
  Gesture,
  GestureDetector,
  GestureHandlerRootView,
} from 'react-native-gesture-handler';
import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withSpring,
} from 'react-native-reanimated';

export default function Ball() {
  const isPressed = useSharedValue(false);
  const offset = useSharedValue({ x: 0, y: 0 });

  const gesture = Gesture.Pan()
    .onBegin(() => {
      isPressed.value = true;
    })
    .onUpdate(e => {
      offset.value = {
        x: offset.value.x + e.changeX,
        y: offset.value.y + e.changeY,
      };
    })
    .onFinalize(() => {
      isPressed.value = false;
    });

  const animatedStyles = useAnimatedStyle(() => {
    return {
      transform: [
        { translateX: offset.value.x },
        { translateY: offset.value.y },
        { scale: withSpring(isPressed.value ? 1.2 : 1) },
      ],
      backgroundColor: isPressed.value ? 'yellow' : 'blue',
    };
  });

  return (
    <GestureHandlerRootView>
      <GestureDetector gesture={gesture}>
        <Animated.View style={[styles.ball, animatedStyles]} />
      </GestureDetector>
    </GestureHandlerRootView>
  );
}

const styles = StyleSheet.create({
  ball: {
    width: 100,
    height: 100,
    borderRadius: 100,
    backgroundColor: 'blue',
    alignSelf: 'center',
  },
});
```

---

## react-native-screens

Exposes native navigation container components to React Native. It is a dependency for full-featured navigation libraries (React Navigation, Expo Router) rather than something you usually use directly. Provides native navigation primitives across iOS, Android, tvOS, visionOS, Windows, and Web.

### What it does for navigation performance

By backing each screen with a platform-native primitive, inactive screens can be detached/freed from the native view hierarchy, reducing memory and improving rendering efficiency versus plain RN `View`s. Modern React Navigation uses screens automatically, and supports a `react-freeze` integration that stops inactive screen subtrees from re-rendering while preserving their component state.

### enableScreens

Native screens are enabled by default. You can opt out globally:

```javascript
import { enableScreens } from 'react-native-screens';

enableScreens(false);
```

This reverts to plain React Native Views. Screens can also be disabled per-navigator via the `detachInactiveScreens` prop in React Navigation.

### How Expo Router configures screens feature flags

`enableScreens()` is not the knob Expo Router uses. `packages/expo-router/src/screensFeatureFlags.ts:1-22` sets, on init:

- `featureFlags.experiment.synchronousScreenUpdatesEnabled`
- `featureFlags.experiment.synchronousHeaderConfigUpdatesEnabled`
- `featureFlags.experiment.synchronousHeaderSubviewUpdatesEnabled`

all to `!disableSynchronousScreensUpdates`, and unconditionally sets `featureFlags.experiment.iosPreventReattachmentOfDismissedScreens = true` (a workaround for iOS bugs when several screens are dismissed in quick succession).

The user-facing switch is the expo-router config-plugin option `disableSynchronousScreensUpdates` (default `false`), read at runtime from `Constants.expoConfig?.extra?.router?.disableSynchronousScreensUpdates`. See `packages/expo-router/plugin/src/withRouter.ts:94` and `packages/expo-router/plugin/options.json:171` — those line numbers hold on **both** `origin/sdk-56` and `origin/sdk-57` (on `main` they have drifted to 103 and 202; that is the post-57 tree). Also `docs/pages/versions/v56.0.0/sdk/router/index.mdx:117`. Cross-reference: `02-expo-router.md`.

---

## react-native-safe-area-context

Flexible way to handle safe area insets in JS. Works on Android, iOS, Web, tvOS, macOS, and Windows. Expo's API page is byte-identical between the SDK 56 and SDK 57 release branches.

```javascript
import {
  SafeAreaView,
  SafeAreaProvider,
  SafeAreaInsetsContext,
  useSafeAreaInsets,
} from 'react-native-safe-area-context';
```

### SafeAreaProvider

Wrap the app root once so descendants can read insets. You may need additional providers at the root of modals and of routes when using `react-native-screens`.

```javascript
import { SafeAreaProvider } from 'react-native-safe-area-context';

function App() {
  return <SafeAreaProvider>{/*...*/}</SafeAreaProvider>;
}
```

To skip the asynchronous first-render inset measurement, pass `initialWindowMetrics`. **Do not do this if the provider can remount.**

```javascript
import { SafeAreaProvider, initialWindowMetrics } from 'react-native-safe-area-context';

function App() {
  return <SafeAreaProvider initialMetrics={initialWindowMetrics}>{/*...*/}</SafeAreaProvider>;
}
```

### SafeAreaView

A regular `View` with the safe area edges applied as padding. Padding you set yourself is *added* to the safe area padding. On web you must set up `SafeAreaProvider` for it to work. Prefer it over the hook where you can — it's implemented natively, so there's no async delay on rotation.

| Prop | Type | Default | Notes |
| --- | --- | --- | --- |
| `edges` | `Edge[]` | `["top", "right", "bottom", "left"]` | Which edges receive the insets. `Edge = 'top' \| 'right' \| 'bottom' \| 'left'`. |
| `emulateUnlessSupported` | `boolean` | `true` | On iOS 10+, emulate the safe area using status bar height and home indicator sizes. |

### useSafeAreaInsets

Hook returning `EdgeInsets` (`top`, `bottom`, `left`, `right`, all `number`) from the nearest provider. More advanced use-case; may perform worse than `SafeAreaView` when rotating the device.

```javascript
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function HookComponent() {
  const insets = useSafeAreaInsets();
  return <View style={{ paddingBottom: Math.max(insets.bottom, 16) }} />;
}
```

### SafeAreaInsetsContext

Render-prop consumer equivalent, for class components or where a hook isn't usable.

```javascript
import { SafeAreaInsetsContext } from 'react-native-safe-area-context';

function Component() {
  return (
    <SafeAreaInsetsContext.Consumer>
      {insets => <View style={{ paddingTop: insets.top }} />}
    </SafeAreaInsetsContext.Consumer>
  );
}
```

### Web SSR

When server-rendering on web, inject `initialSafeAreaInsets` (real values for the target device, or zero). Otherwise the async inset measurement breaks page content rendering.

> `useSafeAreaFrame()` (safe-area frame dimensions) is an upstream library API and is **not** on Expo's API page for SDK 56 or 57 — unverified here. Prefer `useSafeAreaInsets()` unless you specifically need frame geometry.

---

## react-native-svg

Adds SVG support to React Native on iOS, Android, macOS, Windows, tvOS, with a web compatibility layer. Supports most SVG elements and properties. Works in Expo Go.

### Install

```bash
npx expo install react-native-svg
```

### Components

`Svg` (root container), `Path`, `Circle`, `Rect`, `Line`, `Polyline`, `Polygon`, `ClipPath`, `G` (group), and more. Import the namespace with `import * as Svg from 'react-native-svg'`, or default + named as below.

```tsx
import Svg, { Circle, Rect } from 'react-native-svg';

export default function SvgComponent(props) {
  return (
    <Svg height="50%" width="50%" viewBox="0 0 100 100" {...props}>
      <Circle cx="50" cy="50" r="45" stroke="blue" strokeWidth="2.5" fill="green" />
      <Rect x="15" y="15" width="70" height="70" stroke="red" strokeWidth="2" fill="yellow" />
    </Svg>
  );
}
```

### Tips

- Optimize SVGs with SVGOMG before pasting, but keep the `viewBox` — removing it degrades rendering on Android ("Be sure not to remove the `viewbox` for best results on Android").
- Convert an SVG file to a component with SVGR (`react-svgr.com/playground/?native=true&typescript=true`).
- Detailed usage patterns live in the repo's `USAGE.md`: https://github.com/software-mansion/react-native-svg/blob/main/USAGE.md

---

## Sources

In-repo, on the **release branches** `origin/sdk-56` and `origin/sdk-57` (`git show origin/sdk-57:<path>`):

- `packages/expo/bundledNativeModules.json` — all version pins. **Not** `docs/public/static/schemas/v5*.0.0/native-modules.json`; those are stale (see the note under "SDK bundled versions").
- `docs/pages/versions/v56.0.0/sdk/{reanimated,gesture-handler,screens,safe-area-context,svg}.mdx` on `origin/sdk-56`, and `docs/pages/versions/unversioned/sdk/…` on `origin/sdk-57` (the `v57.0.0/sdk` folder is not populated on the release branch — only `ui` lives there)
- `docs/pages/guides/prebuilt-expo-modules.mdx` — precompiled artifacts, `buildFromSource`
- `packages/expo-modules-autolinking/external-configs/ios/` — the precompile set and its feature flags
- `packages/babel-preset-expo/src/index.ts`, `packages/babel-preset-expo/src/configs/expo.ts` — worklets/reanimated Babel wiring
- `packages/expo-router/src/screensFeatureFlags.ts`, `packages/expo-router/plugin/src/withRouter.ts` — screens feature flags
- `packages/@expo/cli/src/start/doctor/dependencies/bundledNativeModules.ts` — how pins are resolved

Published npm tarballs (the ultimate check for anything load-bearing):

- `npm pack expo-modules-autolinking@56.0.21` / `@57.0.9` — precompile set membership and the `*_FEATURE_FLAGS` strings
- `npm view @expo/ui@56.0.18 peerDependencies` / `@56.0.19` — where the reanimated peer was actually dropped
- `npm view expo-modules-core@56.0.22 peerDependencies` / `@57.0.7` — the worklets peer range

Upstream (not verifiable from the monorepo):

- https://docs.swmansion.com/react-native-reanimated/
- https://docs.swmansion.com/react-native-gesture-handler/
- https://github.com/software-mansion/react-native-screens
- https://appandflow.github.io/react-native-safe-area-context/
- https://github.com/software-mansion/react-native-svg

---

## SDK 57 delta

**No documented API changed in this domain.** Comparing `docs/pages/versions/v56.0.0/sdk/<pkg>.mdx` on `origin/sdk-56` against `docs/pages/versions/unversioned/sdk/<pkg>.mdx` on `origin/sdk-57`, all five pages — `reanimated`, `gesture-handler`, `screens`, `safe-area-context`, `svg` — are **identical**. So is `docs/pages/guides/prebuilt-expo-modules.mdx`. The real 56 → 57 delta in this domain is exactly two things: version pins, and the reanimated/worklets precompiled-XCFramework configs.

This section is deliberately short. Several worklet-related changes that look like 57 features were backported into the SDK 56 patch line — they are listed under [SDK 56 patch drift](#sdk-56-patch-drift--worklets-changes-that-are-not-a-reason-to-upgrade-to-57) instead, because "upgrade to 57 to get X" is the wrong advice when X is one patch bump away.

### Breaking

- **Precompiled reanimated XCFramework feature flags changed** — this is the one concrete EAS-only upgrade hazard in this domain. `REANIMATED_FEATURE_FLAGS` flipped `EXPERIMENTAL_CSS_ANIMATIONS_FOR_SVG_COMPONENTS` to `:true` and gained two new flags, `IOS_CSS_CORE_ANIMATION:false` and `USE_ANIMATION_BACKEND:false`; new header mappings (`CSS/core/transition/*.h`, `PseudoStyles/*.h`, `apple/CSS/*.h`, `apple/pseudoSelectors/*.h`) and a new `RNReanimated_view` target over `Common/NativeView` were added. Consequence: **on SDK 57, pinning a reanimated version other than the SDK-pinned `4.5.0` can produce a runtime `Unable to recognize flag: <NAME>` on EAS Build only** — the fix is `buildFromSource` for *both* reanimated and worklets.

  Genuinely 57-only, and worth stating precisely because the changelogs are misleading here. `origin/sdk-57`'s `expo-modules-autolinking/CHANGELOG.md` files #46950 under a `## 56.0.14` heading, which is a changelog-merge artifact from before the release branches forked. The published tarballs settle it: `expo-modules-autolinking@56.0.14` **and** `@56.0.21` both still carry the old flag string; only the 57 line (`@57.0.9`) has `IOS_CSS_CORE_ANIMATION` / `USE_ANIMATION_BACKEND`. Source: `git diff origin/sdk-56 origin/sdk-57 -- packages/expo-modules-autolinking/external-configs/ios/react-native-reanimated/spm.config.json`; #46950 (aligned to reanimated@4.4.1) and #47201 (aligned to reanimated@4.5.0, shipped in autolinking `57.0.0`).

### New in 57

- **Cross-runtime worklet stack traces.** `WORKLETS_FEATURE_FLAGS` in the precompiled worklets config gained `ENABLE_CROSS_RUNTIME_STACK_TRACES:true` (all three flag sites), so stack traces cross the JS/UI runtime boundary. 57-only — `#47478` appears in no `origin/sdk-56` changelog, and `expo-modules-autolinking@56.0.21`'s tarball still lacks the flag. Source: `packages/expo-modules-autolinking/external-configs/ios/react-native-worklets/spm.config.json`; `packages/expo-modules-autolinking/CHANGELOG.md` under `## 57.0.4` (#47478).
- **`expo-modules-core` worklets peer range widened**: `^0.7.4 || ^0.8.0` → `^0.7.4 || ^0.8.0 || ^0.9.0 || ^0.10.0` (still optional). This is what makes the worklets `0.8.3` → `0.10.0` jump installable without peer warnings, and it was *not* backported — `expo-modules-core@56.0.22` still publishes the narrow range. Source: `npm view expo-modules-core@56.0.22 peerDependencies` vs `@57.0.7`; CHANGELOG under `## 57.0.0` (#46950).

### Explicitly NOT in SDK 57 (easy to re-introduce by mistake)

Both of these are on the monorepo `main` branch — that is **SDK 58 in progress**, not 57. Neither is on `origin/sdk-57`.

- **`expo-build-properties` `ios.usePrecompiledModules: false` as a documented opt-out.** The `prebuilt-expo-modules.mdx` "Disabling on iOS" collapsible documents *only* `EXPO_USE_PRECOMPILED_MODULES` on both release branches; the config-plugin wording was added on `main` after the 57 cut. (The plugin *key* does exist on both branches — `packages/expo-build-properties/src/pluginConfig.ts:450` on `origin/sdk-56`, `:454` on `origin/sdk-57`, `@default true` — it is simply undocumented in the guide for 56 and 57 alike.)
- **Prebuilt React core default-on.** `prebuilt_react_active?` is `ENV['RCT_USE_PREBUILT_RNCORE'] == '1'` on **both** `origin/sdk-56` and `origin/sdk-57` (`packages/expo-modules-autolinking/scripts/ios/precompiled_modules.rb:995-997`, same line numbers on both). The flip to `!= '0'` landed after the 57 cut.

### Unchanged in 57 (do not go hunting)

- **The third-party precompile set is identical** — the same seven iOS configs on both release branches, confirmed against the published `expo-modules-autolinking@56.0.21` and `@57.0.9` tarballs. No libraries added.
- **`react-native-screens` (`~4.26.0`), `react-native-safe-area-context` (`~5.7.0`) and `react-native-svg` (`15.15.4`)**: pins identical, docs pages identical, and their `spm.config.json` files untouched between the two branches. There is no screens/safe-area/svg SDK 57 migration. Note screens is `~4.26.0` on both — the frozen docs schemas' `4.25.2` is stale.
- **The worklets/reanimated Babel wiring** in `babel-preset-expo` is unchanged (`src/configs/expo.ts:157-168` on both branches).
- **The `disableSynchronousScreensUpdates` expo-router plugin option** and `screensFeatureFlags.ts` — unchanged, same line numbers on both branches.

### Version pins (56 → 57)

See the pin table at the top of this file — it carries both columns. Changed: reanimated `4.3.1` → `4.5.0`, worklets `0.8.3` → `0.10.0`, gesture-handler `~2.31.1` → `~2.32.0`, lottie-react-native `~7.3.4` → `~7.3.8`, pager-view `8.0.1` → `8.0.2`, keyboard-controller `1.21.6` → `1.21.9` (react-native `0.85.3` → `0.86.0` for context). Unchanged: screens (`~4.26.0`), safe-area-context, svg, skia, webview, masked-view.

### Do not confuse: gesture-handler 3.x is SDK 58, not 57

SDK 57 pins gesture-handler `~2.32.0` — in both `bundledNativeModules.json` and `templates/expo-template-default/package.json` on `origin/sdk-57`. The 3.x line, and its hook API (`usePanGesture`, `useRotationGesture`, …), landed on `main` on 2026-07-14 in commit `70d4ef85511` (`~2.32.0` → `~3.0.2`), which is **not an ancestor of `origin/sdk-57`** (`git merge-base --is-ancestor 70d4ef85511 origin/sdk-57` fails); `main` is now on `~3.1.0`. On **both** SDK 56 and SDK 57 the correct API is `Gesture.Pan()` + `GestureDetector`.

### Anti-pin warning

Two bad pin sources, in the order you are likely to reach for them:

1. **The monorepo `main` branch is not SDK 57 — it is SDK 58 in progress.** `packages/expo/bundledNativeModules.json` and `templates/expo-template-default/package.json` on `main` show gesture-handler `~3.1.0`, reanimated `4.5.1`, worklets `0.10.1`, screens `4.26.0`. Likewise, `packages/*/package.json` on `main` still reads `56.0.5` for many packages, and any CHANGELOG entry under `## Unpublished` has shipped in no SDK at all.
2. **The frozen docs schemas are stale.** `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json` say `react-native-screens` `4.25.2` and `expo-router` `~56.2.9` / `~57.0.0`; the shipped values are `~4.26.0` and `~56.2.16` / `~57.0.8`.

The authoritative source is `packages/expo/bundledNativeModules.json` on the release branch: `git show origin/sdk-57:packages/expo/bundledNativeModules.json`. For anything load-bearing, confirm against the published npm tarball (`npm pack <pkg>@<version>`) — if a symbol or value is not in the tarball, it does not exist for that SDK.

> First-party `expo-*` pins are **not** flat `~57.0.0`. Spot-check before quoting: expo-router `~56.2.16` → `~57.0.8`, expo-video `~56.1.4` → `~57.0.2`, expo-dev-client `~56.0.24` → `~57.0.9`, expo-updates `~56.0.23` → `~57.0.10`, `@expo/fingerprint` `~0.19.9` → `~0.20.6`.
