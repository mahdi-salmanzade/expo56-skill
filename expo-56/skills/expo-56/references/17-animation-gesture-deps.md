# Core Native Dependencies: Animation, Gesture & Navigation (Expo SDK 56)

Reference for the native libraries Expo SDK 56 bundles versions of and that ship with most apps: animation (Reanimated), gestures (Gesture Handler), navigation primitives (Screens), safe area handling (Safe Area Context), and vector graphics (SVG).

> Note on dates: this reference was assembled 2026-05-22 against the live documentation sites. Library "current versions" below reflect the latest published release at fetch time; the **SDK 56-bundled version** is what `npx expo install` resolves and is the authoritative one for SDK 56 projects.

## SDK 56 bundled versions

These are the exact version specifiers from Expo's `bundledNativeModules.json` on the `sdk-56` branch.

| Library | SDK 56 bundled version | Latest published (2026-05) |
| --- | --- | --- |
| react-native-reanimated | `4.3.1` | 4.x |
| react-native-gesture-handler | `~2.31.1` | 2.x |
| react-native-screens | `4.25.1` | 4.25.2 |
| react-native-safe-area-context | `~5.7.0` | 5.8.0 |
| react-native-svg | `15.15.4` | 15.15.5 |

Source: https://raw.githubusercontent.com/expo/expo/sdk-56/packages/expo/bundledNativeModules.json

Always install via `npx expo install <package>` so the SDK-pinned version is resolved rather than the latest npm release.

---

## EAS Build precompiled artifacts (SDK 56)

EAS Build now precompiles some of the most commonly used React Native community libraries to accelerate native builds. In SDK 56 this was extended beyond the Expo modules to include:

- `react-native-reanimated`
- `react-native-screens`

Reported impact: precompiling these "cuts median iOS clean build times on EAS Build by another ~1 minute (~20%) on top of the Expo-modules precompile." This is on top of the earlier ~1 minute (~16%) saved by precompiling the Expo packages themselves. The feature requires no configuration changes for projects using these libraries. The changelog does not pin specific library versions in this note.

Source: https://expo.dev/changelog/sdk-56

---

## react-native-reanimated (SDK 56: 4.3.1)

Declarative animation library running animations on the UI thread (targets 120+ fps) across iOS, Android, and web. SDK 56 bundles the **Reanimated 4** major line. Reanimated 4 splits the worklets runtime into a separate `react-native-worklets` package; install both:

```
npx expo install react-native-reanimated react-native-worklets
```

### Worklets

A worklet is a special JavaScript function that the Worklets Babel plugin automatically transforms so it can be passed to and run on the UI thread, instead of the JS thread. This is what allows animations and gesture callbacks to execute synchronously with rendering. The plugin handles the conversion automatically.

### useSharedValue

A shared value is the core driver of animations; mutating `.value` can be observed on the UI thread.

```javascript
import { useSharedValue } from 'react-native-reanimated';

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
import { useAnimatedStyle } from 'react-native-reanimated';

function App() {
  const animatedStyles = useAnimatedStyle(() => {
    return {
      opacity: sv.value ? 1 : 0,
    };
  });

  return <Animated.View style={[styles.box, animatedStyles]} />;
}
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

Time/easing-based animation modifier, used the same way as `withSpring`: assign `sv.value = withTiming(target)`. (No standalone full-screen example was published on the API page beyond the anti-pattern snippet above.)

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

Sources:
- https://docs.swmansion.com/react-native-reanimated/
- https://docs.swmansion.com/react-native-reanimated/docs/fundamentals/getting-started
- https://docs.swmansion.com/react-native-reanimated/docs/fundamentals/your-first-animation
- https://docs.swmansion.com/react-native-reanimated/docs/core/useAnimatedStyle
- https://docs.expo.dev/versions/latest/sdk/reanimated/

---

## react-native-gesture-handler (SDK 56: ~2.31.1)

Declarative, native-driven touch/gesture system. Integrates tightly with Reanimated to allow smooth gesture-based experiences up to 120 fps.

> Doc-site versioning note: the live docs landing/quickstart pages display a "3.x" docs version label and use a `usePanGesture()` hook in their newest examples. SDK 56, however, pins gesture-handler to the **2.31.x** line, whose canonical API is the `Gesture.Pan()` builder shown below. Treat `Gesture.<Type>()` + `GestureDetector` as the API surface for SDK 56; the `usePanGesture()` hook in the verbatim quickstart example is from the newer 3.x docs and may not exist on 2.31.x.

### Setup

Wrap the app root in `GestureHandlerRootView`, define a gesture, and connect it to a view via `GestureDetector`. The core pattern: "wrap your target view with the GestureDetector component, define your gesture, and pass it in."

### Available gestures (Gesture API)

`Gesture.Pan()`, `Gesture.Tap()`, `Gesture.LongPress()`, `Gesture.Fling()`, `Gesture.Pinch()`, `Gesture.Rotation()`, `Gesture.ForceTouch()`. Gestures can be composed (e.g. `Gesture.Simultaneous(...)`, `Gesture.Race(...)`, `Gesture.Exclusive(...)`).

### Gesture callbacks

`onBegin`, `onStart`, `onUpdate`, `onChange`, `onEnd`, `onFinalize`. For a pan gesture, the update event exposes `translationX`/`translationY` (relative to start) and `changeX`/`changeY` (delta since last event). Store the element's start position in a shared value because only translation relative to the gesture start is provided.

### Quickstart example (verbatim, from 3.x docs)

Draggable ball wired to Reanimated. Note this uses the newer `usePanGesture()` hook; on SDK 56's 2.31.x build the equivalent is `const gesture = Gesture.Pan().onBegin(...).onUpdate(...).onFinalize(...)`.

```javascript
import { StyleSheet } from 'react-native';
import {
  GestureDetector,
  GestureHandlerRootView,
  usePanGesture,
} from 'react-native-gesture-handler';
import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withSpring,
} from 'react-native-reanimated';

export default function Ball() {
  const isPressed = useSharedValue(false);
  const offset = useSharedValue({ x: 0, y: 0 });

  const gesture = usePanGesture({
    onBegin: () => {
      isPressed.value = true;
    },
    onUpdate: (e) => {
      offset.value = {
        x: offset.value.x + e.changeX,
        y: offset.value.y + e.changeY,
      };
    },
    onFinalize: () => {
      isPressed.value = false;
    },
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

### Equivalent 2.31.x (SDK 56) gesture definition

```javascript
const gesture = Gesture.Pan()
  .onBegin(() => {
    isPressed.value = true;
  })
  .onUpdate((e) => {
    offset.value = {
      x: offset.value.x + e.changeX,
      y: offset.value.y + e.changeY,
    };
  })
  .onFinalize(() => {
    isPressed.value = false;
  });
```

Sources:
- https://docs.swmansion.com/react-native-gesture-handler/
- https://docs.swmansion.com/react-native-gesture-handler/docs/guides/quickstart/
- https://docs.swmansion.com/react-native-gesture-handler/docs/gestures/pan-gesture/
- https://docs.swmansion.com/react-native-gesture-handler/docs/fundamentals/gesture-composition/

---

## react-native-screens (SDK 56: 4.25.1)

Exposes native navigation container components to React Native. It is a dependency for full-featured navigation libraries (React Navigation, Expo Router) rather than something you usually use directly. Provides native navigation primitives across iOS, Android, tvOS, visionOS, Windows, and Web.

### What it does for navigation performance

By backing each screen with a platform-native primitive, inactive screens can be detached/freed from the native view hierarchy, reducing memory and improving rendering efficiency versus plain RN `View`s. React Navigation (v2.14.0+) automatically uses screens. v3.9.0+ supports an experimental `react-freeze` integration that stops inactive screen subtrees from re-rendering while preserving their component state.

### enableScreens

Native screens are enabled by default. You can opt out globally:

```javascript
import { enableScreens } from 'react-native-screens';

enableScreens(false);
```

This reverts to plain React Native Views. Screens can also be disabled per-navigator via the `detachInactiveScreens` prop in React Navigation.

### Version notes

- 4.25.x requires React Native 0.82.0+ with the Fabric (new) architecture.
- 4.25.0+ dropped support for the legacy Paper architecture.

Source: https://github.com/software-mansion/react-native-screens

---

## react-native-safe-area-context (SDK 56: ~5.7.0)

Flexible way to handle safe area insets in JS. Works on Android, iOS, Web, macOS, and Windows.

### SafeAreaProvider

Wrap the app root once so descendants can read insets.

```javascript
import { SafeAreaProvider } from 'react-native-safe-area-context';

function App() {
  return <SafeAreaProvider>{/*...*/}</SafeAreaProvider>;
}
```

### useSafeAreaInsets

Hook returning the safe area insets (`top`, `bottom`, `left`, `right`) of the nearest provider.

```javascript
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function HookComponent() {
  const insets = useSafeAreaInsets();
  return <View style={{ paddingBottom: Math.max(insets.bottom, 16) }} />;
}
```

### SafeAreaView

Component that applies the safe area insets as padding to its children, simplifying layout around notches and system UI.

### useSafeAreaFrame

Hook returning the dimensions of the safe area frame, useful for computing available screen space.

Sources:
- https://github.com/AppAndFlow/react-native-safe-area-context
- https://appandflow.github.io/react-native-safe-area-context/api/safe-area-provider
- https://appandflow.github.io/react-native-safe-area-context/api/use-safe-area-insets
- https://docs.expo.dev/versions/latest/sdk/safe-area-context/

---

## react-native-svg (SDK 56: 15.15.4)

Adds SVG support to React Native on iOS, Android, macOS, Windows, with a web compatibility layer. Supports most SVG elements and properties.

### Components

`Svg` (root container), `Path`, `Circle`, `Rect`, `Line`, `Polyline`, `Polygon`, `G` (group), and more.

### Install

```bash
npm install react-native-svg
```

(In an Expo project use `npx expo install react-native-svg`.) For bare iOS: `cd ios && pod install`.

### Compatibility notes

- Requires React Native 0.70.0+.
- Fabric (new architecture) support added in v13.0.0+.
- Detailed usage patterns live in the repo's `USAGE.md`: https://github.com/software-mansion/react-native-svg/blob/main/USAGE.md

Source: https://github.com/software-mansion/react-native-svg
