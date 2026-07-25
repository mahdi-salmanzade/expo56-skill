# Expo UI (`@expo/ui`) — SwiftUI / Jetpack Compose

> Knowledge-base reference for **Expo SDK 56**. `@expo/ui` is "a set of native input
> components that allows you to build fully native interfaces with Jetpack Compose and
> SwiftUI." As of SDK 56 it is **production-ready** and included in the default
> `create-expo-app` template and available in Expo Go.

**Primary sources**
- Changelog: https://expo.dev/changelog/sdk-56
- SDK reference index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/
- Documentation index (llms.txt): https://docs.expo.dev/llms.txt

**Version pin** (`packages/expo/bundledNativeModules.json` on `origin/sdk-56` — the file
that ships inside the `expo` package and that `expo install` actually resolves against):
SDK 56 pins `@expo/ui` at `~56.0.23`. `templates/expo-template-default/package.json` on
the same branch agrees (`"@expo/ui": "~56.0.23"`). npm `dist-tags.sdk-56` is `56.0.23`.

> Do **not** read the pin out of `docs/public/static/schemas/v56.0.0/native-modules.json`
> — that file is stale and still says `~56.0.6` (it also says `react-native-screens
> 4.25.0`, where the shipped pin is `~4.26.0`). Use `bundledNativeModules.json` on the
> release branch.

This file was first captured around `@expo/ui` **56.0.12** (2026-05-22), so it already
covered most of the `56.0.6`–`56.0.10` surface. The table below is a full per-patch map
of everything added across the `56.0.x` line, not a list of "what was missing".

_Note: `https://docs.expo.dev/versions/latest/...` now redirects to **v57**. Always use
the explicit `v56.0.0` URLs below when you want SDK 56 docs._

### Traps a model gets wrong from memory

- Universal subtrees need a `Host` root — **except** universal `BottomSheet`, which
  mounts its own `Host` (`@expo/ui` 56.0.10+).
- There are **two different `BottomSheet` APIs**: universal (declarative `isPresented`)
  and `@expo/ui/community/bottom-sheet` (imperative refs). See §2.8 vs §5.1.
- `Button` takes `label` (string) **or** `children` — `children` wins when both are set.
- Controlled `TextInput` uses `useNativeState()` / `ObservableState`, **not** `useState`.
- `TextInput` `selection` / `selectTextOnFocus` need **iOS 18.0+**.
- Modifiers live on separate subpaths: `@expo/ui/swift-ui/modifiers` and
  `@expo/ui/jetpack-compose/modifiers` — never on the package root. See §3.1 / §4.1.
- `WorkletCallback` is **not** a public export (see §1).
- Without `react-native-worklets`, `withAnimation(...)` **and** assigning a non-null
  `ObservableState.onChange` **throw**; only worklet props no-op silently. See §1.
- `@expo/ui` gained **nothing** in SDK 57 — 57.0.7 is a strict subset of 56.0.23. See the
  SDK 57 delta.

---

## 1. Overview & Status (SDK 56)

Source: https://expo.dev/changelog/sdk-56

- The **Jetpack Compose (Android)** and **SwiftUI (iOS)** APIs reached **stable** status
  in SDK 56, after three SDK cycles of refinement.
- Now included in the default `create-expo-app` template and available in **Expo Go**.
- Works on both **TV** (Android TV, Apple TV) and **mobile** (Android, iOS).
- New in SDK 56:
  - Cross-platform **Universal Components** (web APIs remain **experimental**).
  - Custom view / modifier extensions via SwiftUI and Jetpack Compose.
  - `useMaterialColors` hook for Material 3 Dynamic Colors.
  - `Icon` component paired with `@expo/material-symbols`.
  - `useNativeState` hook for native state control.
  - Synchronous, flicker-free controlled text inputs on both platforms.

> `WorkletCallback` is **not** a public export of `@expo/ui`. It is an internal native
> SharedObject class (`ExpoUI.WorkletCallback`) used to marshal worklets across the
> bridge; users never construct it. Verified absent from `src/universal/index.ts`,
> `src/swift-ui/index.tsx` and `src/jetpack-compose/index.ts` on the `sdk-56` branch.

### Additions across the 56.0.x line (all still SDK 56 — the pin is `~56.0.23`)

Source: `packages/expo-ui/CHANGELOG.md` on the `sdk-56` release branch. Everything here
is reachable on SDK 56 with no SDK upgrade; the "minimum patch" column is the version to
require if a project is pinned lower.

| Min patch | Addition |
|-----------|----------|
| 56.0.6 | `@expo/ui/community/slider`, `@expo/ui/community/menu`; universal `Collapsible`, `List`, `ListItem`, `Picker`; `snapPoints` on universal `BottomSheet`; `colorScheme` / `layoutDirection` on `Host`; Compose `combinedClickable`; `background(color, { animationSpec })`; `useNativeState` seed captured once via `useRef` |
| 56.0.8 | Universal `Host` gains `matchContents`, `layoutDirection`, `onLayoutContent`, `useViewportSizeMeasurement`, `ignoreSafeArea`; SwiftUI `Alert`, `symbolEffect` modifier |
| 56.0.9 | SwiftUI `withAnimation(animation, body)`; `onChange` listener on `useNativeState`; JS-thread writes to native state; Jetpack Compose `Snackbar`, `LoadingIndicator`, `ContainedLoadingIndicator` |
| 56.0.10 | `@expo/ui/community/pager-view`; universal `BottomSheet` no longer requires an outer `Host` |
| 56.0.16 | SwiftUI `accessibilityHidden`, `accessibilityIdentifier`, `dynamicTypeSize`, `buttonBorderShape`, `listRowSpacing`; `Label` `children`; custom SF Symbols in `Image`; `<DisclosureGroup.Label>`; universal `<Collapsible.labelStyle>`; Jetpack Compose `NavigationBar` / `NavigationBarItem`, `dropShadow` / `innerShadow` modifiers |
| 56.0.17 | **Breaking:** Android universal `TextInput` renders a Compose `BasicTextField` instead of the filled Material `TextField`. Also: `BasicTextField` component, `imageScale`, `minimumScaleFactor`, `accessibilityInputLabels`, `onGloballyPositioned`, `onGeometryChange` now reports `x`/`y`, `get()` / `set()` on `useNativeState` |
| 56.0.18 | SwiftUI `activityBackgroundTint` (Live Activity background) |
| 56.0.19 | `seedColor` on universal + SwiftUI `<Host>`; `ignoreSafeArea="container"` on the **SwiftUI** `Host`; `modifiers` prop on universal `ListItem`; SwiftUI `accessibilityElement`, `redacted` / `unredacted` / `privacySensitive` / `invalidatableContent`, `presentationSizing`; web `<Host colorScheme>` honored; web universal components revamped (design tokens, light/dark, focus styles); component props exported as **interfaces** (module augmentation); `react-native-reanimated` removed from the worklet integration |
| 56.0.20 | SwiftUI `accessibilityAddTraits` / `accessibilityRemoveTraits`, `strokeBorder` (full `StrokeStyle`) |
| 56.0.22 | Fix components crashing on Android 7 (API 24/25) where `Color` props had no type converter |

### Dependencies

`@expo/ui` peer dependencies (identical on `sdk-56` and `sdk-57`):
`expo`, `react`, `react-native` (required); `@babel/core`, `react-dom`,
`react-native-worklets` (optional). Runtime deps: `sf-symbols-typescript`, `vaul` (web).

`src/State/optionalWorklets.ts` swallows the `require` error when
`react-native-worklets` is not linked, so the fallback behaviour is **split** — do not
assume everything degrades quietly:

- **Silent no-op:** worklet *props* routed through `useWorkletProp` — it returns `null`
  when `worklets` is `undefined` (`src/State/useWorkletProp.ts`).
- **Throws:** `withAnimation(...)` (`src/swift-ui/withAnimation.ts`), **and** assigning a
  non-null `ObservableState.onChange`, which throws `"ObservableState.onChange requires
  the 'react-native-worklets' package, which couldn't be loaded. Install
  react-native-worklets and rebuild the native app, then assign onChange again."`
  (`defineOnChangeProperty` in `src/State/useNativeState.ts`). Assigning `onChange = null`
  returns before that check, so cleanup is safe either way.

`react-native-reanimated` is **not** required (it was dropped from the integration in
`@expo/ui` 56.0.19 / 57.0.0) and is not in `peerDependencies` on either line.

### Three API surfaces (import paths)

| Layer | Import path | Platforms |
|-------|-------------|-----------|
| **Universal** (single API over native toolkits) | `@expo/ui` (root) | Android, iOS, Web (experimental), Expo Go |
| **Jetpack Compose** (Android-native) | `@expo/ui/jetpack-compose` | Android |
| **SwiftUI** (iOS-native) | `@expo/ui/swift-ui` | iOS |
| **Drop-in replacements** (community-lib compatible) | `@expo/ui/community/*` | Android, iOS, Web, Expo Go |

The universal components delegate to `@expo/ui/jetpack-compose` on Android, to
`@expo/ui/swift-ui` on iOS, and to `react-dom` / `react-native-web` on web.

Source code: `https://github.com/expo/expo/tree/sdk-56/packages/expo-ui`

---

## 2. Universal Components

Sources:
- https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/
- Per-component pages linked below.

Universal components are imported directly from the package root: `import { ... } from '@expo/ui'`.
Every universal subtree must be wrapped in a **`Host`** root — with one exception:
universal `BottomSheet` renders its own internal
`<Host style={{ position: 'absolute' }} pointerEvents="none">` since 56.0.10, so an
outer `Host` is optional for it (`src/universal/BottomSheet/index.{ios,android}.tsx`).

### Component catalog

| Category | Components |
|----------|-----------|
| Container (required root) | `Host` |
| Layout | `Column`, `Row`, `Spacer`, `ScrollView` |
| Display | `Text`, `Icon` |
| Controls | `Button`, `Switch`, `Checkbox`, `Slider`, `TextInput`, `Picker` |
| Disclosure & Presentation | `BottomSheet`, `Collapsible` |
| Collections & Forms | `List` (with `ListItem`), `FieldGroup` |

Platform support: **Android, iOS, Web, Expo Go** (web is experimental).

### Minimal example

```tsx
import { Host, Column, Button, Text } from '@expo/ui';

export default function Example() {
  return (
    <Host style={{ flex: 1 }}>
      <Column spacing={12} alignment="center">
        <Text>Hello, world!</Text>
        <Button label="Press me" onPress={() => alert('Pressed')} />
      </Column>
    </Host>
  );
}
```

### Full universal page list

(All under `https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/`)
`host`, `row`, `column`, `spacer`, `scrollview`, `text`, `icon`, `button`, `switch`,
`checkbox`, `slider`, `textinput`, `picker`, `bottomsheet`, `collapsible`, `list`,
`fieldgroup`, `rnhostview`.

---

### 2.1 `Host`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/host/
Import: `import { Host } from '@expo/ui';`
Platforms: Android, iOS, Web, Expo Go.

`UniversalHostProps extends ViewProps`, so all React Native `View` props (`style`,
`pointerEvents`, `testID`, …) are accepted in addition to the table below.

| Prop | Type | Default | Platforms | Notes |
|------|------|---------|-----------|-------|
| `children` | `ReactNode` | — | A/i/W | |
| `matchContents` | `boolean \| { horizontal?: boolean, vertical?: boolean }` | `false` | A/i/W | Updates host size to match content's layout from native toolkit. **Can only be set once, on mount** |
| `style` | RN `View` styles (via `ViewProps`) | — | A/i/W | |
| `layoutDirection` | `'leftToRight' \| 'rightToLeft'` | locale (`I18nManager`) | A/i/W | |
| `colorScheme` | `ColorSchemeName` (`'light' \| 'dark'`) | — | A/i/W | Honored on web since 56.0.19 |
| `seedColor` | `ColorValue` | — | A/i/W | 56.0.19+. Android: generates a Material 3 `SchemeTonalSpot` palette, also exposed via `useMaterialColors()`. iOS: applied as the SwiftUI tint. Web: emits a primary color scale as CSS variables |
| `ignoreSafeArea` | `'all' \| 'keyboard'` | — | A/i/W | The **SwiftUI** `Host` additionally accepts `'container'` (56.0.19+); the universal one does not |
| `onLayoutContent` | `(event: { nativeEvent: { height: number, width: number } }) => void` | — | A/i/W | |
| `useViewportSizeMeasurement` | `boolean` | `false` | A/i/W | Propose the viewport size when no explicit size is set — needed by `List` |

```tsx
import { Host, Column, Text, Switch } from '@expo/ui';

export default function ThemedHostExample() {
  return (
    // matchContents sizes the RN view to the native content; seedColor themes the subtree.
    <Host matchContents seedColor="#6750A4" colorScheme="dark">
      <Column spacing={12} alignment="center">
        <Text>Tinted subtree</Text>
        <Switch value onValueChange={() => {}} />
      </Column>
    </Host>
  );
}
```

---

### 2.2 `Text`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/text/
Import: `import { Text } from '@expo/ui';`

| Prop | Type | Platforms |
|------|------|-----------|
| `children` | `string` | A/i/W |
| `disabled` | `boolean` | A/i/W |
| `hidden` | `boolean` | A/i/W |
| `numberOfLines` | `number` (truncates with ellipsis) | A/i/W |
| `onAppear` | `() => void` | A/i/W |
| `onDisappear` | `() => void` | A/i/W |
| `onPress` | `() => void` | A/i/W |
| `style` | `Pick<ViewStyle, 'padding' \| 'paddingHorizontal' \| 'paddingVertical' \| 'paddingTop' \| 'paddingBottom' \| 'paddingLeft' \| 'paddingRight' \| 'backgroundColor' \| 'borderRadius' \| 'borderWidth' \| 'borderColor' \| 'opacity' \| 'width' \| 'height'>` | A/i/W |
| `textStyle` | `{ color, fontFamily, fontSize, fontWeight: 'normal'\|'bold'\|'100'..'900', letterSpacing, lineHeight, textAlign: 'center'\|'left'\|'right' }` | A/i/W |
| `testID` | `string` | A/i/W |
| `modifiers` | `ModifierConfig[]` | Android, iOS only |

```tsx
import { Host, Text } from '@expo/ui';

export default function StyledTextExample() {
  return (
    <Host matchContents>
      <Text textStyle={{ fontSize: 24, fontWeight: '700', textAlign: 'center' }}>
        Headline
      </Text>
    </Host>
  );
}
```

---

### 2.3 `Button`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/button/
Platforms: Android, iOS, Web, Expo Go.

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `children` | `ReactNode` | — | Custom content; when provided `label` is ignored |
| `label` | `string` | — | Text label; ignored when `children` provided |
| `variant` | `ButtonVariant` (`'filled' \| 'outlined' \| 'text'`) | `'filled'` | |
| `disabled` | `boolean` | — | |
| `hidden` | `boolean` | — | |
| `onPress` | `() => void` | — | |
| `onAppear` | `() => void` | — | |
| `onDisappear` | `() => void` | — | |
| `modifiers` | `ModifierConfig[]` | — | Android/iOS escape hatch |
| `style` | `ViewStyle` subset (padding, backgroundColor, borderRadius, borderWidth, borderColor, opacity, width, height) | — | |
| `testID` | `string` | — | |

```tsx
<Button label="Press me" onPress={() => alert('Pressed!')} />

<Button variant="filled" label="Filled" onPress={() => {}} />
<Button variant="outlined" label="Outlined" onPress={() => {}} />
<Button variant="text" label="Text" onPress={() => {}} />

<Button label="Disabled" onPress={() => {}} disabled />
```

---

### 2.4 `Switch`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/switch/
Platforms: Android, iOS, Web.

| Prop | Type | Required | Notes |
|------|------|----------|-------|
| `value` | `boolean` | yes | Whether the switch is on |
| `onValueChange` | `(value: boolean) => void` | yes | |
| `label` | `string` | no | |
| `disabled` | `boolean` | no | |
| `modifiers` | `ModifierConfig[]` | no | |
| `testID` | `string` | no | |

```tsx
import { useState } from 'react';
import { Host, Switch } from '@expo/ui';

export default function SwitchExample() {
  const [enabled, setEnabled] = useState(false);
  return (
    <Host matchContents>
      <Switch value={enabled} onValueChange={setEnabled} />
    </Host>
  );
}
```

---

### 2.5 `Slider`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/slider/
Platforms: Android, iOS, Web.

| Prop | Type | Default | Required |
|------|------|---------|----------|
| `value` | `number` | — | yes |
| `onValueChange` | `(value: number) => void` | — | yes |
| `min` | `number` | `0` | no |
| `max` | `number` | `1` | no |
| `step` | `number` | — | no |
| `disabled` | `boolean` | — | no |
| `modifiers` | `ModifierConfig[]` | — | no |
| `testID` | `string` | — | no |

```tsx
import { useState } from 'react';
import { Host, Column, Slider, Text } from '@expo/ui';

export default function SteppedSliderExample() {
  const [volume, setVolume] = useState(50);
  return (
    <Host style={{ flex: 1 }}>
      <Column spacing={8}>
        <Text>Volume: {volume}</Text>
        <Slider value={volume} onValueChange={setVolume} min={0} max={100} step={10} />
      </Column>
    </Host>
  );
}
```

---

### 2.6 `Picker`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/picker/
Platforms: Android, iOS, Web. Uses `<Picker.Item label value />` children.

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `appearance` | `PickerAppearance` (`'menu' \| 'wheel'`) | `'menu'` | |
| `children` | `ReactNode` | — | `<Picker.Item>` options |
| `enabled` | `boolean` | `true` | |
| `onValueChange` | `(value: T) => void` | — | |
| `selectedValue` | `T` | — | Must match the `value` of a `<Picker.Item>` |
| `testID` | `string` | — | |

Types: `PickerAppearance = 'menu' | 'wheel'`; `PickerItemValue = string | number`.

```tsx
<Picker selectedValue={value} onValueChange={setValue}>
  {FLAVOURS.map(f => (
    <Picker.Item key={f.value} label={f.label} value={f.value} />
  ))}
</Picker>

<Picker selectedValue={value} onValueChange={setValue} appearance="wheel">
  {FLAVOURS.map(f => (
    <Picker.Item key={f.value} label={f.label} value={f.value} />
  ))}
</Picker>
```

---

### 2.7 `TextInput`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/textinput/
Import: `import { TextInput, useNativeState } from '@expo/ui';`
Native SwiftUI / Jetpack Compose backing with a React Native-compatible API.

| Prop | Type | Platforms |
|------|------|-----------|
| `autoCapitalize` | `'none' \| 'words' \| 'sentences' \| 'characters'` | A/i/W |
| `autoComplete` | `AutoComplete` | A/i/W |
| `autoCorrect` | `boolean` | A/i/W |
| `autoFocus` | `boolean` | A/i/W |
| `caretHidden` | `boolean` | A/i/W |
| `cursorColor` | `ColorValue` | A/i/W |
| `defaultValue` | `string` | A/i/W |
| `editable` | `boolean` | A/i/W |
| `enterKeyHint` | `EnterKeyHint` | A/i/W |
| `inputMode` | `InputMode` | A/i/W |
| `keyboardType` | `KeyboardTypeOptions` | A/i/W |
| `maxLength` | `number` | A/i/W |
| `modifiers` | `ModifierConfig[]` | Android, iOS |
| `multiline` | `boolean` | A/i/W |
| `numberOfLines` | `number` | A/i/W |
| `onBlur` | `() => void` | A/i/W |
| `onChangeText` | `(text: string) => void` | A/i/W |
| `onContentSizeChange` | `(size: { height: number, width: number }) => void` | A/i/W |
| `onFocus` | `() => void` | A/i/W |
| `onSelectionChange` | `(selection: { end: number, start: number }) => void` | A/i/W |
| `onSubmitEditing` | `(text: string) => void` | A/i/W |
| `placeholder` | `string` | A/i/W |
| `placeholderTextColor` | `ColorValue` | A/i/W |
| `readOnly` | `boolean` | A/i/W |
| `ref` | `Ref<TextInputRef>` | A/i/W |
| `returnKeyType` | `ReturnKeyTypeOptions` | A/i/W |
| `rows` | `number` | A/i/W |
| `secureTextEntry` | `boolean` | A/i/W |
| `selection` | `ObservableState<{ end: number, start: number }>` | iOS 18.0+, Android, Web |
| `selectionColor` | `ColorValue` | A/i/W |
| `selectionHandleColor` | `ColorValue` | Android |
| `selectTextOnFocus` | `boolean` | iOS 18.0+, Android, Web |
| `style` | `Pick<ViewStyle, ...>` | A/i/W |
| `testID` | `string` | A/i/W |
| `textAlign` | `'auto' \| 'center' \| 'left' \| 'right' \| 'justify'` | A/i/W |
| `textStyle` | `{ color, fontFamily, fontSize, fontWeight, letterSpacing, lineHeight, textAlign }` | A/i/W |
| `underlineColorAndroid` | `ColorValue` | Android |
| `value` | `ObservableState<string>` | A/i/W |

Note: controlled `value`/`selection` use `ObservableState` (from `useNativeState`),
enabling synchronous, flicker-free updates from worklets.

> **Android appearance changed in `@expo/ui` 56.0.17** (and applies to all of SDK 57):
> the universal `TextInput` now renders a Compose `BasicTextField` instead of the filled
> Material `TextField`. Visible effect: no filled container, no underline, no floating
> label. Restyle via `style` / `modifiers`, or import `TextField` explicitly from
> `@expo/ui/jetpack-compose` to get the Material look back.
> (`CHANGELOG.md` → `## 56.0.17` 🛠 Breaking changes, #46442.)

**Uncontrolled:**
```tsx
import { Button, Column, Host, TextInput, type TextInputRef } from '@expo/ui';
import { useRef } from 'react';

export default function UncontrolledTextInputExample() {
  const inputRef = useRef<TextInputRef>(null);
  return (
    <Host matchContents={{ vertical: true }}>
      <Column spacing={8}>
        <TextInput
          ref={inputRef}
          defaultValue="hello"
          placeholder="Type here"
          onChangeText={value => console.log(value)}
        />
        <Button label="Clear" onPress={() => inputRef.current?.clear()} />
      </Column>
    </Host>
  );
}
```

**Controlled (worklet):**
```tsx
import { Host, TextInput, useNativeState } from '@expo/ui';
import { useEffectEvent } from 'react';

export default function ControlledTextInputExample() {
  const text = useNativeState('');

  const handleChangeText = useEffectEvent((value: string) => {
    'worklet';
    text.value = value === 'Hello' ? 'World' : value;
  });

  return (
    <Host matchContents={{ vertical: true }}>
      <TextInput value={text} placeholder="Type here" onChangeText={handleChangeText} />
    </Host>
  );
}
```

**Worklet masking (phone number):**
```tsx
import { Host, TextInput, useNativeState } from '@expo/ui';
import { useEffectEvent } from 'react';

function formatPhone(input: string) {
  'worklet';
  const digits = input.replace(/\D/g, '').slice(0, 10);
  if (digits.length <= 3) return digits;
  if (digits.length <= 6) return `(${digits.slice(0, 3)}) ${digits.slice(3)}`;
  return `(${digits.slice(0, 3)}) ${digits.slice(3, 6)}-${digits.slice(6)}`;
}

export default function PhoneMaskExample() {
  const phone = useNativeState('');
  const selection = useNativeState({ start: 0, end: 0 });

  const handleChangeText = useEffectEvent((value: string) => {
    'worklet';
    const formatted = formatPhone(value);
    if (formatted !== value) {
      phone.value = formatted;
      selection.value = { start: formatted.length, end: formatted.length };
    }
  });

  return (
    <Host matchContents={{ vertical: true }}>
      <TextInput
        value={phone}
        selection={selection}
        keyboardType="phone-pad"
        placeholder="(555) 123-4567"
        onChangeText={handleChangeText}
      />
    </Host>
  );
}
```

---

### 2.8 `BottomSheet` (universal)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/bottomsheet/
Platforms: Android, iOS, Web.

> Note: this is the **universal** `BottomSheet` (declarative `isPresented` API). For the
> `@gorhom/bottom-sheet`-compatible imperative API, see §5.1.
>
> Since 56.0.10 this component mounts its own `Host` internally, so the outer `Host` in
> the example below is optional.

| Prop | Type | Default | Required |
|------|------|---------|----------|
| `children` | `ReactNode` | — | no |
| `isPresented` | `boolean` | — | yes |
| `onDismiss` | `() => void` | — | yes |
| `showDragIndicator` | `boolean` | `true` | no |
| `snapPoints` | `SnapPoint[]` | — | no |
| `modifiers` | `ModifierConfig[]` | — | no |
| `testID` | `string` | — | no |

```tsx
import { useState } from 'react';
import { Host, Column, Button, BottomSheet, Text } from '@expo/ui';

export default function BottomSheetExample() {
  const [isPresented, setIsPresented] = useState(false);
  return (
    <Host style={{ flex: 1 }}>
      <Button label="Open sheet" onPress={() => setIsPresented(true)} />
      <BottomSheet isPresented={isPresented} onDismiss={() => setIsPresented(false)}>
        <Column spacing={12}>
          <Text textStyle={{ fontSize: 18, fontWeight: '700' }}>Sheet contents</Text>
          <Text>Drag down or tap the overlay to dismiss.</Text>
          <Button label="Close" onPress={() => setIsPresented(false)} />
        </Column>
      </BottomSheet>
    </Host>
  );
}
```

**Hosting a React Native list inside the sheet** (the common real-world case): wrap it in
`RNHostView`. `snapPoints` sizes the sheet and the list scrolls within that height; with
`nestedScrollEnabled` the list scrolls its own content first and the remaining drag moves
the sheet. Source: `v56.0.0/sdk/ui/universal/bottomsheet.mdx`.

```tsx
import { Host, BottomSheet, Button, RNHostView } from '@expo/ui';
import { FlatList, Text } from 'react-native';

<Host matchContents>
  <Button label="Open" onPress={() => setIsPresented(true)} />
  <BottomSheet
    isPresented={isPresented}
    onDismiss={() => setIsPresented(false)}
    snapPoints={['half', 'full']}>
    <RNHostView>
      <FlatList
        nestedScrollEnabled
        style={{ flex: 1 }}
        data={DATA}
        keyExtractor={item => item}
        renderItem={({ item }) => <Text style={{ padding: 16 }}>{item}</Text>}
      />
    </RNHostView>
  </BottomSheet>
</Host>
```

> On Android, `{ fraction }` / `{ height }` snap points collapse to the nearest of
> `'half'` / `'full'` — the underlying `ModalBottomSheet` only has two resting states.

---

## 3. Jetpack Compose API (`@expo/ui/jetpack-compose`)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/jetpack-compose/
Platform: **Android only.** Every component must be wrapped in a `Host`.

```tsx
import { Host, Button } from '@expo/ui/jetpack-compose';

export function SaveButton() {
  return (
    <Host matchContents>
      <Button onClick={() => alert('Saved!')}>Save changes</Button>
    </Host>
  );
}
```

### Full Jetpack Compose component / page catalog

(All under `https://docs.expo.dev/versions/v56.0.0/sdk/ui/jetpack-compose/`)

| Page | Page | Page |
|------|------|------|
| `host` | `alertdialog` | `badge` |
| `badgedbox` | `basicalertdialog` | `box` |
| `button` | `card` | `carousel` |
| `checkbox` | `chip` | `column` |
| `datetimepicker` | `divider` | `dockedsearchbar` |
| `dropdownmenu` | `exposeddropdownmenubox` | `floatingactionbutton` |
| `flowrow` | `horizontalfloatingtoolbar` | `horizontalpager` |
| `icon` | `iconbutton` | `lazycolumn` |
| `lazyrow` | `listitem` | `colors` |
| `bottomsheet` | `modifiers` | `progress` |
| `pulltorefreshbox` | `radiobutton` | `rnhostview` |
| `row` | `searchbar` | `segmentedbutton` |
| `shape` | `slider` | `snackbar` |
| `spacer` | `surface` | `switch` |
| `text` | `textfield` | `togglebutton` |
| `tooltip` | `usenativestate` | `navigationbar` |

Notable Android-only Material 3 helpers: `colors` (incl. the `useMaterialColors` hook
for Material 3 Dynamic Colors), `modifiers`, `useNativeState`, `rnhostview` (embed RN
views inside Compose subtrees).

**Exported but genuinely undocumented in v56** (do not conclude from a missing page that
the API is absent): `LoadingIndicator` / `ContainedLoadingIndicator` (56.0.9+; a docs
page exists only in the SDK 57 docs set) and `AnimatedVisibility`. Neither string appears
anywhere under `docs/pages/versions/v56.0.0/`.

**Documented, but on a shared page rather than their own:** `BasicTextField` (56.0.17+) is
introduced in the opening paragraph of `jetpack-compose/textfield` and has its own section
there with a `DecorationBox` / `Placeholder` / `InnerTextField` example;
`SingleChoiceSegmentedButtonRow` and `MultiChoiceSegmentedButtonRow` each get a bullet and
a full example on `jetpack-compose/segmentedbutton`.

All six verified present in `build/jetpack-compose/index.d.ts` of the published
`@expo/ui@56.0.23` tarball.

### 3.1 Jetpack Compose modifiers (`@expo/ui/jetpack-compose/modifiers`)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/jetpack-compose/modifiers/

Modifiers are plain functions imported from the `/modifiers` **subpath** (a real
`exports` entry in `package.json`, not the package root) and passed as an array to a
component's `modifiers` prop. **Order matters** — Compose applies them left to right, so
`padding` before `background` differs from the reverse.

```tsx
import { Button, Host } from '@expo/ui/jetpack-compose';
import {
  paddingAll,
  fillMaxWidth,
  background,
  border,
  shadow,
} from '@expo/ui/jetpack-compose/modifiers';

<Host style={{ flex: 1 }}>
  <Button modifiers={[paddingAll(16), fillMaxWidth(), background('#FF6B6B')]}>
    Full-width padded button
  </Button>
  <Button modifiers={[paddingAll(12), background('#4ECDC4'), border(2, '#2C3E50'), shadow(4)]}>
    Styled with border and shadow
  </Button>
</Host>
```

Notable additions in later 56 patches: `combinedClickable` (56.0.6),
`background(color, { animationSpec })` (56.0.6), `dropShadow(shape, config)` /
`innerShadow(shape, config)` (56.0.16 — `innerShadow` requires a `background(...)`
applied *before* it or nothing renders), `onGloballyPositioned` (56.0.17).

---

## 4. SwiftUI API (`@expo/ui/swift-ui`)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/swift-ui/
Platform: **iOS only.** Every component must be wrapped in a `Host`.

```tsx
import { Host, Button } from '@expo/ui/swift-ui';

export function SaveButton() {
  return (
    <Host style={{ flex: 1 }}>
      <Button label="Save changes" />
    </Host>
  );
}
```

### Full SwiftUI component / page catalog

(All under `https://docs.expo.dev/versions/v56.0.0/sdk/ui/swift-ui/`)

| Page | Page | Page |
|------|------|------|
| `host` | `accessorywidgetbackground` | `alert` |
| `bottomsheet` | `button` | `colorpicker` |
| `confirmationdialog` | `contextmenu` | `controlgroup` |
| `datepicker` | `disclosuregroup` | `divider` |
| `form` | `gauge` | `group` |
| `hstack` | `image` | `label` |
| `lazyhstack` | `lazyvstack` | `link` |
| `list` | `menu` | `modifiers` |
| `namespace` | `overlay` | `picker` |
| `popover` | `progressview` | `rnhostview` |
| `scrollview` | `section` | `securefield` |
| `slider` | `spacer` | `swipeactions` |
| `tabview` | `text` | `textfield` |
| `toggle` | `usenativestate` | `vstack` |
| `zstack` | | |

Notable SwiftUI-only constructs: stack layouts (`hstack`/`vstack`/`zstack`/lazy
variants), `form`/`section`, `swipeactions`, `contextmenu`, `popover`, `gauge`,
`namespace`, `accessorywidgetbackground` (widget support), `rnhostview` (embed RN views
inside SwiftUI), `modifiers`, `usenativestate`.

**Exported but without a dedicated v56.0.0 docs page**: `Chart`, `ContentUnavailableView`,
`LabeledContent`, `ShareLink`, `Stepper`, `Grid`, `Mask`, `GlassEffectContainer`,
`SyncToggle`, `Shapes`, and `withAnimation`. All eleven are present in
`build/swift-ui/index.d.ts` of the published `@expo/ui@56.0.23` tarball. (`Mask`,
`Shapes`, `GlassEffectContainer` and `SyncToggle` are *mentioned* on other v56 pages;
`Chart`, `ContentUnavailableView`, `LabeledContent`, `ShareLink`, `Stepper`, `Grid` and
`withAnimation` appear nowhere in the v56 docs tree at all.)

### 4.1 SwiftUI modifiers (`@expo/ui/swift-ui/modifiers`)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/swift-ui/modifiers/

Same shape as the Compose side: functions imported from the `/modifiers` subpath and
passed as an array to a component's `modifiers` prop.

```tsx
import { Host, Text, VStack } from '@expo/ui/swift-ui';
import {
  background,
  cornerRadius,
  padding,
  shadow,
  foregroundColor,
  onTapGesture,
} from '@expo/ui/swift-ui/modifiers';

<Host style={{ flex: 1 }}>
  <VStack spacing={20}>
    <Text
      modifiers={[
        background('#FF6B6B'),
        cornerRadius(12),
        padding({ all: 16 }),
        foregroundColor('#FFFFFF'),
      ]}>
      Basic styled text
    </Text>
    <Text
      modifiers={[
        padding({ horizontal: 20, vertical: 12 }),
        shadow({ radius: 4, x: 0, y: 2, color: '#4ECDC440' }),
        onTapGesture(() => console.log('Tapped!')),
      ]}>
      Shadow + tap gesture
    </Text>
  </VStack>
</Host>
```

The published `@expo/ui@56.0.23` tarball exports **154** symbols from
`@expo/ui/swift-ui/modifiers`, of which **~148 are modifiers** — the other six are
helpers (`createModifier`, `createModifierWithEventListener`,
`createViewModifierEventListener`, the `Animation` factory, `shapes`, `VALUE_SYMBOL`).
Count against the tarball, not the branch: `menuOrder` exists in
`src/swift-ui/modifiers/menuOrder.ts` on `origin/sdk-56` but sits under `## Unpublished`
in the changelog and is **absent from every published `56.0.x`**.

Modifiers added across the `56.0.x` line that a model is unlikely to know (all verified
in the published `56.0.23` build):

| Modifier | Signature | Min `@expo/ui` patch |
|----------|-----------|----------------------|
| `symbolEffect` | `symbolEffect(effect, options?)` | 56.0.8 |
| `font` `textStyle` option | `font({ textStyle: … })` for Dynamic Type | 56.0.10 |
| `accessibilityHidden` | `(hidden?: boolean)` — defaults `true` | 56.0.16 |
| `accessibilityIdentifier` | `(identifier: string)` | 56.0.16 |
| `dynamicTypeSize` | `(size)` or `({ min, max })`; cascades from `<Host>` | 56.0.16 |
| `buttonBorderShape` | reshapes a styled (e.g. `glass`) button | 56.0.16 |
| `listRowSpacing` | `(spacing?: number)` | 56.0.16 |
| `imageScale` | `('small' \| 'medium' \| 'large')` | 56.0.17 |
| `minimumScaleFactor` | `(factor: number)` | 56.0.17 |
| `accessibilityInputLabels` | `(inputLabels: string[])` | 56.0.17 |
| `onGeometryChange` | now reports global `x`/`y` alongside size | 56.0.17 |
| `activityBackgroundTint` | `(color: Color \| null)` — Live Activity | 56.0.18 |
| `accessibilityElement` | `('ignore' \| 'combine' \| 'contain')`, default `'ignore'` | 56.0.19 |
| `redacted` / `unredacted` / `privacySensitive` / `invalidatableContent` | redaction family (skeleton loading, sensitive content) | 56.0.19 |
| `presentationSizing` | `('automatic' \| 'fitted' \| 'form' \| 'page')` | 56.0.19 — **56-line only**, absent from `57.0.7` |
| `accessibilityAddTraits` / `accessibilityRemoveTraits` | `(traits: AccessibilityTrait[])` | 56.0.20 |
| `strokeBorder` | `({ color?, style?: StrokeStyle, antialiased?, shape?, cornerRadius? })` | 56.0.20 |

### SwiftUI guides
- https://docs.expo.dev/guides/expo-ui-swift-ui — using Expo UI to integrate SwiftUI.
- https://docs.expo.dev/guides/expo-ui-swift-ui/extending — custom SwiftUI components & modifiers.
- https://docs.expo.dev/guides/expo-ui-jetpack-compose/extending — custom Jetpack Compose components & modifiers.

---

## 5. Drop-in Replacements (`@expo/ui/community/*`)

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/drop-in-replacements/

API-compatible replacements for popular React Native community libraries, backed by
`@expo/ui` native (Jetpack Compose / SwiftUI) implementations. Platforms: Android, iOS,
Web, Expo Go.

| Replacement component | Replaces | Detail page |
|-----------------------|----------|-------------|
| `BottomSheet` | `@gorhom/bottom-sheet` | `/drop-in-replacements/bottomsheet` |
| `DateTimePicker` | `@react-native-community/datetimepicker` | `/drop-in-replacements/datetimepicker` |
| `MaskedView` | `@react-native-masked-view/masked-view` | `/drop-in-replacements/maskedview` |
| `Menu` | `@react-native-menu/menu` | `/drop-in-replacements/menu` |
| `PagerView` | `react-native-pager-view` | `/drop-in-replacements/pagerview` |
| `Picker` | `@react-native-picker/picker` | `/drop-in-replacements/picker` |
| `SegmentedControl` | `@react-native-segmented-control/segmented-control` | `/drop-in-replacements/segmentedcontrol` |
| `Slider` | `@react-native-community/slider` | `/drop-in-replacements/slider` |

Generic migration pattern (from the changelog):
```tsx
// Old:
import DateTimePicker from '@react-native-community/datetimepicker';

// New:
import DateTimePicker from '@expo/ui/community/datetime-picker';
```

---

### 5.1 `BottomSheet` — replaces `@gorhom/bottom-sheet`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/drop-in-replacements/bottomsheet/
Platforms: Android, iOS, Web, Expo Go.

Imports:
```tsx
import BottomSheet, { BottomSheetView } from '@expo/ui/community/bottom-sheet';
import { BottomSheetModal, BottomSheetView } from '@expo/ui/community/bottom-sheet';
```

Exports (`src/community/bottom-sheet/index.tsx`): `BottomSheet` (default + named),
`BottomSheetModal`, `BottomSheetModalProvider`, `BottomSheetView`,
`BottomSheetScrollView`, `BottomSheetFlatList`, `BottomSheetSectionList`,
`BottomSheetTextInput` (the last four are straight re-exports of the RN components —
native sheets coordinate scrolling themselves), and the `useBottomSheet()` hook.
Types: `BottomSheetProps`, `BottomSheetMethods`, `BottomSheetViewProps`,
`BottomSheetHandleProps`, `BottomSheetBackdropProps`, `BottomSheetBackgroundProps`,
`BottomSheetFooterProps`.

`BottomSheet` / `BottomSheetModal` use TypeScript declaration merging, so
`useRef<BottomSheet>(null)` resolves to `BottomSheetMethods` without importing it.

**`BottomSheetProps` / `BottomSheetViewProps`:**
- `children: React.ReactNode`
- `enableDynamicSizing?: boolean` (default `true`) — only effective when `snapPoints` is
  omitted; when both are set `snapPoints` wins (unlike `@gorhom/bottom-sheet`, which
  prepends a content-sized point)
- `enablePanDownToClose?: boolean` (default `false`) — on iOS this enables **both**
  swipe-to-dismiss and backdrop-tap dismiss; SwiftUI cannot separate them
- `index?: number` (default `0`; `-1` starts closed). `BottomSheetModal` always starts
  closed and uses `index` only to pick the snap point `present()` opens to
- `onChange?: (index: number) => void`
- `onClose?: () => void`
- `onDismiss?: () => void`
- `snapPoints?: (string | number)[]` — bottom-to-top; **Android supports only 2 states**,
  so with >2 points only the first and last are effective
- `style?: StyleProp<ViewStyle>`
- `backgroundStyle?: StyleProp<ViewStyle>` — only `backgroundColor` is honored on native
  (Android `containerColor`, iOS `.presentationBackground(_:)`)

**`BottomSheetMethods` — all 8 imperative ref methods** (`src/community/bottom-sheet/types.ts`):

| Method | Signature | Notes |
|--------|-----------|-------|
| `snapToIndex` | `(index: number) => void` | Android maps indices to its two states (partial / expanded) |
| `snapToPosition` | `(position: string \| number) => void` | Mapped to nearest detent on native; arbitrary positions only on web |
| `expand` | `() => void` | Highest snap point (full screen on Android) |
| `collapse` | `() => void` | Lowest snap point (~50% on Android) |
| `close` | `() => void` | |
| `forceClose` | `() => void` | Identical to `close()` here — native sheets can't be interrupted mid-close |
| `present` | `() => void` | On `BottomSheetModal`, opens at `index`; on `BottomSheet`, `snapToIndex(0)` |
| `dismiss` | `() => void` | Alias of `close()` |

`snapToIndex(0)` and `present()` are therefore interchangeable for opening a plain
`<BottomSheet index={-1}>`; both appear in the examples below and neither is a typo.

Many `@gorhom/bottom-sheet` props are accepted for API compatibility but have **no
effect** on native: `animateOnMount`, `overDragResistanceFactor`, `enableOverDrag`,
`enableContentPanningGesture`, `enableHandlePanningGesture`, `keyboardBehavior`,
`keyboardBlurBehavior`, `backgroundComponent`, `footerComponent`, `handleStyle`,
`handleIndicatorStyle`, `containerStyle`. `handleComponent={null}` hides the drag
indicator (any non-null value just shows the default native one).

**Basic usage:**
```tsx
import { useRef } from 'react';
import { Button, Text, View } from 'react-native';
import BottomSheet, { BottomSheetView } from '@expo/ui/community/bottom-sheet';

export default function BottomSheetExample() {
  const sheetRef = useRef<BottomSheet>(null);

  return (
    <View style={{ flex: 1 }}>
      <Button title="Open" onPress={() => sheetRef.current?.snapToIndex(0)} />
      <BottomSheet
        ref={sheetRef}
        snapPoints={['25%', '50%', '90%']}
        index={-1}
        onChange={index => console.log('onChange', index)}
        onClose={() => console.log('closed')}
        enablePanDownToClose>
        <BottomSheetView style={{ flex: 1, padding: 24, alignItems: 'center' }}>
          <Text>Sheet content</Text>
        </BottomSheetView>
      </BottomSheet>
    </View>
  );
}
```

**`BottomSheetModal`:**
```tsx
import { useRef } from 'react';
import { Button, Text, View } from 'react-native';
import { BottomSheetModal, BottomSheetView } from '@expo/ui/community/bottom-sheet';

export default function BottomSheetModalExample() {
  const modalRef = useRef<BottomSheetModal>(null);

  return (
    <View style={{ flex: 1 }}>
      <Button title="Present" onPress={() => modalRef.current?.present()} />
      <BottomSheetModal ref={modalRef} snapPoints={['50%', '90%']} enablePanDownToClose>
        <BottomSheetView style={{ padding: 24 }}>
          <Text>Modal content</Text>
          <Button title="Dismiss" onPress={() => modalRef.current?.dismiss()} />
        </BottomSheetView>
      </BottomSheetModal>
    </View>
  );
}
```

**Dynamic sizing:**
```tsx
import { useRef } from 'react';
import { Button, Text, View } from 'react-native';
import BottomSheet, { BottomSheetView } from '@expo/ui/community/bottom-sheet';

export default function DynamicBottomSheetExample() {
  const sheetRef = useRef<BottomSheet>(null);

  return (
    <View style={{ flex: 1 }}>
      <Button title="Open" onPress={() => sheetRef.current?.present()} />
      <BottomSheet ref={sheetRef} index={-1} enablePanDownToClose>
        <BottomSheetView style={{ padding: 24 }}>
          <Text>This sheet sizes itself to its content.</Text>
        </BottomSheetView>
      </BottomSheet>
    </View>
  );
}
```

---

### 5.2 `DateTimePicker` — replaces `@react-native-community/datetimepicker`

Source: https://docs.expo.dev/versions/v56.0.0/sdk/ui/drop-in-replacements/datetimepicker/
Import: `import DateTimePicker from '@expo/ui/community/datetime-picker';`

| Prop | Type | Platforms |
|------|------|-----------|
| `accentColor` | `string` | Android, iOS |
| `disabled` | `boolean` | iOS |
| `display` | `'default' \| 'spinner' \| 'compact' \| 'inline' \| 'calendar' \| 'clock'` | Android, iOS |
| `is24Hour` | `boolean` | Android |
| `locale` | `string` | iOS |
| `maximumDate` | `Date` | Android, iOS |
| `minimumDate` | `Date` | Android, iOS |
| `mode` | `'date' \| 'time' \| 'datetime'` | Android, iOS |
| `negativeButton` | `{ label: string }` | Android |
| `onChange` | `(event, date?) => void` | Android, iOS |
| `onDismiss` | `() => void` | Android |
| `onValueChange` | `(event, date) => void` | Android, iOS |
| `positiveButton` | `{ label: string }` | Android |
| `presentation` | `'inline' \| 'dialog'` | Android |
| `testID` | `string` | Android, iOS |
| `themeVariant` | `'dark' \| 'light'` | iOS |
| `timeZoneName` | `string` | iOS |
| `value` | `Date` | Android, iOS |
| `style` | ViewProps style | Android, iOS |

**Basic (date):**
```tsx
import { useState } from 'react';
import DateTimePicker from '@expo/ui/community/datetime-picker';

export default function DateTimePickerExample() {
  const [date, setDate] = useState(new Date());

  return (
    <DateTimePicker
      value={date}
      onValueChange={(event, selectedDate) => {
        setDate(selectedDate);
      }}
      mode="date"
    />
  );
}
```

**Time:**
```tsx
import { useState } from 'react';
import DateTimePicker from '@expo/ui/community/datetime-picker';

export default function TimePickerExample() {
  const [date, setDate] = useState(new Date());
  return (
    <DateTimePicker
      value={date}
      onValueChange={(event, selectedDate) => {
        setDate(selectedDate);
      }}
      mode="time"
    />
  );
}
```

**With constraints:**
```tsx
import { useState } from 'react';
import DateTimePicker from '@expo/ui/community/datetime-picker';

const today = new Date();
const thirtyDaysFromNow = new Date(today.getTime() + 30 * 24 * 60 * 60 * 1000);

export default function ConstrainedDatePickerExample() {
  const [date, setDate] = useState(new Date());
  return (
    <DateTimePicker
      value={date}
      onValueChange={(event, selectedDate) => {
        setDate(selectedDate);
      }}
      mode="date"
      minimumDate={today}
      maximumDate={thirtyDaysFromNow}
    />
  );
}
```

**Android dialog presentation:**
```tsx
import { useState } from 'react';
import { Button, View } from 'react-native';
import DateTimePicker from '@expo/ui/community/datetime-picker';

export default function AndroidDialogExample() {
  const [date, setDate] = useState(new Date());
  const [show, setShow] = useState(false);

  return (
    <View>
      <Button title="Pick a date" onPress={() => setShow(true)} />
      {show && (
        <DateTimePicker
          value={date}
          onValueChange={(event, selectedDate) => {
            setShow(false);
            setDate(selectedDate);
          }}
          onDismiss={() => {
            setShow(false);
          }}
          mode="date"
          presentation="dialog"
        />
      )}
    </View>
  );
}
```

---

## 6. Hooks & State

### `useNativeState`

Sources:
- https://docs.expo.dev/versions/v56.0.0/sdk/ui/jetpack-compose/usenativestate/
- (mirror: `/swift-ui/usenativestate/`)

A React hook that creates **observable state shared between JavaScript and native views**.
Enables synchronous updates to native UI from worklets without triggering React's render
cycle (flicker-free controlled inputs).

Signature:
```tsx
useNativeState(initialValue: T): ObservableState<T>
```

`initialValue` is captured **once**, on the first render (via `useRef`, fixed in 56.0.6);
changing it later does not recreate the state.

Returns `ObservableState<T>`. The full native type (from `@expo/ui/swift-ui` or
`@expo/ui/jetpack-compose`, `src/State/useNativeState.ts`):

```tsx
type ObservableState<T> = SharedObject & {
  value: T;
  get(): T;                                 // 56.0.17+, React Compiler-friendly read
  set(value: T): void;                      // 56.0.17+, React Compiler-friendly write
  onChange: ((value: T) => void) | null;    // 56.0.9+
};
```

> **Platform caveat (type surface vs runtime — they differ):** the root `@expo/ui` export
> narrows the *type* to `{ value: T }` on all three platforms
> (`src/universal/State.{ts,ios.ts,android.ts}`). On native that is only a type narrowing:
> `State.ios.ts` is literally `export { useNativeState } from '@expo/ui/swift-ui';` and
> `State.android.ts` re-exports from `@expo/ui/jetpack-compose`, so `get()` / `set()` /
> `onChange` **do exist at runtime** on the object the root import returns — TypeScript
> just cannot see them. Use the platform-specific import (or a cast) to type them.
> On **web** they genuinely do not exist: `State.ts` is a plain React-state polyfill with
> only a `value` getter/setter.

**Thread semantics** (current doc comment, superseding the older "reads are safe from any
thread" wording): writes from a UI worklet are synchronous and immediately readable;
writes from the JS thread are scheduled to the UI thread **asynchronously** and the new
value is *not* readable until the update has been applied. JS-thread writes were only
allowed from 56.0.9 onward. Prefer writing from a worklet when you need synchrony.

**`onChange`** (56.0.9+): a single worklet listener invoked on the native UI runtime
after iOS `didSet` / Android's setter. Assigning replaces the previous listener; assign
`null` to clear. The initial value does not fire it. Attach in `useEffect`, clear in the
cleanup.

> The setter **throws**, it does not no-op, if `react-native-worklets` is missing, or if
> the assigned function lacks the `'worklet'` directive. Only clearing (`= null`) is
> unconditionally safe. See §1 → Dependencies.

```tsx
const state = useNativeState(0);

useEffect(() => {
  state.onChange = value => {
    'worklet';
    console.log('changed to', value);
  };
  return () => {
    state.onChange = null;
  };
}, []);
```

```tsx
const maskedPhone = useNativeState('');
const selection = useNativeState({ start: 0, end: 0 });

const handleValueChange = useEffectEvent((v: string) => {
  'worklet';
  // formatting logic...
  maskedPhone.value = formatted;
  selection.value = { start: formatted.length, end: formatted.length };
});
```

### `withAnimation` (SwiftUI, 56.0.9+)

```tsx
import { withAnimation } from '@expo/ui/swift-ui';
import { Animation } from '@expo/ui/swift-ui/modifiers';

withAnimation(
  animation: ChainableAnimationType | null,   // built with the `Animation` factory; null = no animation
  body: () => void,                           // MUST be a worklet; mutates useNativeState values
  completion?: () => void,                    // MUST be a worklet; iOS/tvOS 17+, silently skipped below
  completionCriteria?: 'logicallyComplete' | 'removed'  // default 'logicallyComplete'
): void
```

Mirrors SwiftUI's `withAnimation(_:_:)`. Any `useNativeState` value mutated inside `body`
animates to its new value. Throws if `react-native-worklets` is missing, and throws if
`body` (or `completion`) is not a worklet. Source: `src/swift-ui/withAnimation.ts`.

### Other hooks / utilities (SDK 56)
- `useMaterialColors` — Material 3 Dynamic Colors (Android; see `jetpack-compose/colors`).
  Also picks up a `seedColor` set on an ancestor `<Host>` (56.0.19+).
- `useBottomSheet()` — context hook exported from `@expo/ui/community/bottom-sheet`.
- `Icon` — platform-native icon: SF Symbol on iOS, Material Symbol XML vector on Android.
  Optionally install `@expo/material-symbols` (`npx expo install @expo/material-symbols`)
  for the bundled Android assets. Use `Icon.select({ ios: 'star.fill', android:
  import('@expo/material-symbols/star.xml') })` for cross-platform names; the
  `@expo/ui/babel-plugin` (auto-loaded by `babel-preset-expo`) lets Metro tree-shake the
  unused side. **`Icon` does not render on web.**

---

## 7. Stability Notes & Known Limitations

- **Stable:** Jetpack Compose (Android) and SwiftUI (iOS) native APIs are production-ready
  in SDK 56.
- **Experimental:** **Web** support for universal components and community drop-ins is
  experimental. On web, universal components fall back to `react-dom` /
  `react-native-web`, so native-only behaviors may differ.
- **Platform-specific props:** Many drop-in props are Android-only or iOS-only (see the
  per-prop tables, e.g. `DateTimePicker` `is24Hour`/`presentation` = Android;
  `locale`/`themeVariant`/`timeZoneName` = iOS).
- **Version-gated props:** `TextInput` `selection` and `selectTextOnFocus` require
  **iOS 18.0+** (work on Android/Web regardless).
- **`Host` requirement:** Every `@expo/ui/jetpack-compose`, `@expo/ui/swift-ui`, and
  universal subtree must be wrapped in a `Host` root; sizing usually needs `style={{ flex: 1 }}`
  or `matchContents`. **Exception:** the universal `BottomSheet` mounts its own `Host`
  (56.0.10+), so an outer one is optional there.
- **Worklet peer dep:** worklet features need `react-native-worklets` (an *optional* peer
  dep). Worklet *props* silently no-op without it (`useWorkletProp` returns `null`), but
  **two APIs throw**: `withAnimation(...)` and assigning a non-null
  `ObservableState.onChange`. `react-native-reanimated` is not needed (removed from the
  integration in 56.0.19) and is not a peer dep on either SDK line.
- **Two BottomSheet APIs:** universal `BottomSheet` is declarative (`isPresented`/`onDismiss`);
  the `@expo/ui/community/bottom-sheet` drop-in is imperative
  (refs + `snapToIndex`/`present`/`dismiss`, `snapPoints`) to match `@gorhom/bottom-sheet`.
- **Worklets:** controlled `TextInput` and synchronous native state rely on the
  `'worklet'` directive and `ObservableState.value` writes from the UI thread.

---

## Appendix — Source URL index

- Changelog: https://expo.dev/changelog/sdk-56
- SDK ref index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/
- Universal index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/universal/
- Universal sub-pages: `.../universal/{host,row,column,spacer,scrollview,text,icon,button,switch,checkbox,slider,textinput,picker,bottomsheet,collapsible,list,fieldgroup,rnhostview}/`
- Jetpack Compose index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/jetpack-compose/
- SwiftUI index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/swift-ui/
- Drop-in replacements index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/drop-in-replacements/
- Drop-in sub-pages: `.../drop-in-replacements/{bottomsheet,datetimepicker,maskedview,menu,pagerview,picker,segmentedcontrol,slider}/`
- Guides: https://docs.expo.dev/guides/expo-ui-swift-ui/ , `/guides/expo-ui-swift-ui/extending/` , `/guides/expo-ui-jetpack-compose/extending/`
- Docs index: https://docs.expo.dev/llms.txt

---

## SDK 57 delta

**`@expo/ui` is effectively unchanged between SDK 56 and SDK 57.** Everything Expo's
SDK 57 release notes list under `@expo/ui` was also shipped into the SDK 56 line as patch
releases `56.0.16`–`56.0.20`, and since SDK 56 pins `~56.0.23` a fresh SDK 56 install
already has all of it. Diffing the **published tarballs** (`npm pack @expo/ui@56.0.23`
vs `@expo/ui@57.0.7` — the versions `bundledNativeModules.json` pins on each release
branch) confirms it. Exactly three TypeScript sources differ:

```
src/community/bottom-sheet/BottomSheet.ios.tsx
src/community/datetime-picker/DateTimePicker.tsx
src/swift-ui/modifiers/presentationModifiers.ts
```

plus, on the native side, `ios/Modifiers/PresentationModifiers.swift`,
`ios/Modifiers/ViewModifierRegistry.swift` (one `register("presentationSizing")` block)
and `android/build.gradle` (version string only). In every case **57 is the one that is
behind**. `peerDependencies` / `peerDependenciesMeta` / `dependencies` and the `exports`
map are identical in both tarballs; every other file under `build/`, `src/`, `ios/`,
`android/`, `swift-ui/` and `jetpack-compose/` is byte-identical.

### New in 57

**Nothing.** There is no `@expo/ui` API that requires SDK 57. Do not budget upgrade work
for this package.

Everything the SDK 57 notes attribute to `@expo/ui` was **backported to the SDK 56 patch
line** and is reachable by bumping the patch, not the SDK. Confirmed by checking
`packages/expo-ui/CHANGELOG.md` on both `origin/sdk-56` and `origin/sdk-57`:

| Commonly mis-attributed to 57 | Minimum `@expo/ui` patch on the 56 line |
|-------------------------------|-----------------------------------------|
| SwiftUI `accessibilityHidden`, `accessibilityIdentifier`, `dynamicTypeSize`, `buttonBorderShape`, `listRowSpacing`; `Label` `children`; custom SF Symbols in `Image`; `<DisclosureGroup.Label>`; `<Collapsible.labelStyle>`; Compose `NavigationBar` / `NavigationBarItem`, `dropShadow` / `innerShadow` | **56.0.16** |
| Compose `BasicTextField` + the universal-`TextInput` restyle, `onGloballyPositioned`, `useNativeState` `get()` / `set()`, SwiftUI `imageScale`, `minimumScaleFactor`, `accessibilityInputLabels`, `onGeometryChange` `x`/`y` | **56.0.17** |
| SwiftUI `activityBackgroundTint` | **56.0.18** |
| `seedColor` on `<Host>`, SwiftUI `ignoreSafeArea="container"`, `ListItem` `modifiers`, `accessibilityElement`, the `redacted` / `unredacted` / `privacySensitive` / `invalidatableContent` family, web `<Host colorScheme>`, the web universal-component revamp, props exported as interfaces, the `react-native-reanimated` removal | **56.0.19** |
| SwiftUI `accessibilityAddTraits` / `accessibilityRemoveTraits`, `strokeBorder` | **56.0.20** |

See the per-patch table in §1 for the exact patch each landed in. Since SDK 56 pins
`~56.0.23`, a fresh SDK 56 install already satisfies all of these.

### The one behaviour change to plan for — and it is a 56 patch, not a 57 upgrade

- **[universal][android] `TextInput` renders a Compose `BasicTextField`** instead of the
  filled Material `TextField` (#46442). No filled container, no underline, no floating
  label. Restyle via `style` / `modifiers`, or import `TextField` from
  `@expo/ui/jetpack-compose` to get the Material look back.
- It is listed under `## 57.0.0` in the SDK 57 changelog, but it shipped first in
  **`@expo/ui` 56.0.17** and is in `origin/sdk-56`'s changelog too. **Minimum patch:
  56.0.17.** If you are pinned ≤ 56.0.16, this lands the moment you bump the patch —
  moving to SDK 57 changes nothing extra here.

### Two things SDK 57 does *not* have that SDK 56 does

Both were on `sdk-56` at `56.0.23` and were **absent** from `57.0.7` (verified in the
published tarballs, not the branches):

- SwiftUI `presentationSizing('automatic' | 'fitted' | 'form' | 'page')` (56.0.19,
  #47050) and its use in `community/bottom-sheet` dynamic sizing, which makes the sheet
  hug its content on iPad instead of opening near full-screen. Absent from
  `build/swift-ui/modifiers/presentationModifiers.d.ts` and from the iOS
  `ViewModifierRegistry` in `57.0.7`.
- The `community/datetime-picker` `compact`/`automatic` fix that keeps the field from
  collapsing to zero width under a non-stretch parent such as `alignItems: 'center'`
  (#47033).

If your app depends on either, upgrading to `@expo/ui@57.0.7` is a **regression**.

Docs-tree note: `jetpack-compose/loadingindicator` is the only new `@expo/ui` page in the
SDK 57 docs set — but `LoadingIndicator` / `ContainedLoadingIndicator` shipped in
`@expo/ui` **56.0.9**, so a missing v56 docs page is not evidence of a missing API. The
universal (18), SwiftUI (43) and drop-in (8) page sets are otherwise identical between
the two SDK docs sets. (On `origin/sdk-57` the SDK 57 pages still live under
`docs/pages/versions/unversioned/`, not `v57.0.0/` — the `v57.0.0/` directory on that
branch is nearly empty. `git ls-tree` there will mislead you; diff against
`unversioned/`.)

> Do **not** trust `docs/public/static/data/v56.0.0/expo-ui/**` to decide what shipped in
> SDK 56: that directory is not frozen, it is regenerated from whatever branch you are on.
> On `origin/sdk-57`, `data/v56.0.0/expo-ui/swift-ui/modifiers.json` is byte-identical to
> the `unversioned` (SDK 57) copy — it lists `menuOrder`, which has **never** shipped in
> any published `@expo/ui`, and omits `presentationSizing`, which **has** shipped since
> 56.0.19. It is wrong in both directions. Use `packages/expo-ui/CHANGELOG.md` on
> `origin/sdk-56` / `origin/sdk-57`, or the published tarball, instead.

### Version pins (56 → 57)

| Package | SDK 56 | SDK 57 | Moved? |
|---------|--------|--------|--------|
| `@expo/ui` | `~56.0.23` | `~57.0.7` | major bump only — `57.0.7` is a strict **subset** of `56.0.23`, no new API |
| `react-native-worklets` (optional peer) | `0.8.3` | `0.10.0` | yes |
| `react-native-reanimated` (not a peer dep of `@expo/ui` on either line) | `4.3.1` | `4.5.0` | yes |
| `react-native` | `0.85.3` | `0.86.0` | yes |

Source: `packages/expo/bundledNativeModules.json` on `origin/sdk-56` and `origin/sdk-57`.

> Do **not** source these from `docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json`
> — those are stale. The v56 schema still lists `@expo/ui` at `~56.0.6` and
> `react-native-screens` at `4.25.0` (the shipped screens pin is `~4.26.0`, and it is
> **unchanged** between 56 and 57).

Also update the `sourceCodeUrl` base when reading source for 57:
`https://github.com/expo/expo/tree/sdk-57/packages/expo-ui`.
