# Expo UI (`@expo/ui`) — SwiftUI / Jetpack Compose

> Knowledge-base reference for **Expo SDK 56**. `@expo/ui` is "a set of native input
> components that allows you to build fully native interfaces with Jetpack Compose and
> SwiftUI." As of SDK 56 it is **production-ready** and included in the default
> `create-expo-app` template and available in Expo Go.

**Primary sources**
- Changelog: https://expo.dev/changelog/sdk-56
- SDK reference index: https://docs.expo.dev/versions/v56.0.0/sdk/ui/
- Documentation index (llms.txt): https://docs.expo.dev/llms.txt

_Captured 2026-05-22. Note: Expo serves the `latest` alias and `v56.0.0` for SDK 56;
both are referenced below._

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
  - `WorkletCallback` for synchronous UI worklet callbacks.
  - Synchronous, flicker-free controlled text inputs on both platforms.

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
Every universal subtree must be wrapped in a **`Host`** root.

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

| Prop | Type | Default | Platforms | Notes |
|------|------|---------|-----------|-------|
| `children` | `ReactNode` | — | A/i/W | |
| `matchContents` | `boolean \| { horizontal: boolean, vertical: boolean }` | `false` | A/i/W | Updates host size to match content's layout from native toolkit |
| `style` | `ViewProps` (RN View styles) | — | A/i/W | |
| `layoutDirection` | `'leftToRight' \| 'rightToLeft'` | — | A/i/W | |
| `colorScheme` | `ColorSchemeName` (`'light' \| 'dark'`) | — | Android, iOS | |
| `ignoreSafeArea` | `'all' \| 'keyboard'` | — | A/i/W | |
| `onLayoutContent` | `(event: { nativeEvent: { height: number, width: number } }) => void` | — | A/i/W | |
| `useViewportSizeMeasurement` | `boolean` | `false` | A/i/W | |

```tsx
import { Host, Column, Text, Button } from '@expo/ui';

export default function HostExample() {
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
| `tooltip` | `usenativestate` | |

Notable Android-only Material 3 helpers: `colors` (incl. the `useMaterialColors` hook
for Material 3 Dynamic Colors), `modifiers`, `useNativeState`, `rnhostview` (embed RN
views inside Compose subtrees).

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

Exports: `BottomSheet` (default), `BottomSheetModal`, `BottomSheetModalProvider`, `BottomSheetView`.

**`BottomSheetProps` / `BottomSheetViewProps`:**
- `children: React.ReactNode`
- `enableDynamicSizing?: boolean` (default `true`)
- `enablePanDownToClose?: boolean` (default `false`)
- `index?: number` (default `0`)
- `onChange?: (index: number) => void`
- `onClose?: () => void`
- `onDismiss?: () => void`
- `snapPoints?: (string | number)[]`

Imperative ref methods seen in examples: `snapToIndex(i)`, `present()`, `dismiss()`.

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

Returns `ObservableState<T>` — a SharedObject with a `.value` property of type `T`.
"Reads are safe from any thread; prefer writing from a worklet so the update runs on the
native UI thread."

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

### Other hooks / utilities (SDK 56)
- `useMaterialColors` — Material 3 Dynamic Colors (Android; see `jetpack-compose/colors`).
- `WorkletCallback` — synchronous UI worklet callbacks.
- `Icon` component pairs with the `@expo/material-symbols` package.

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
  or `matchContents`.
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
