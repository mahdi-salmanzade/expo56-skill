# expo-56

An offline, **point-in-time** reference for **Expo SDK 56** (React Native 0.85, React 19.2, Hermes v1), packaged as a [Claude Code](https://code.claude.com) skill and distributed as a plugin marketplace.

It exists for one reason: **SDK 56 postdates current model training**, so when an AI assistant writes Expo code from memory it tends to produce plausible-but-wrong APIs (e.g. inventing `expo-av`, missing that `expo-file-system` `copy()`/`move()` are now async, or that `expo-router` dropped React Navigation). This skill grounds the model in the actual SDK 56 surface — for both **building new apps** and **migrating from SDK 55**.

> ⚠️ **Snapshot, not a live mirror.** These references were compiled on **2026-05-22, when SDK 56 was in beta.** Package versions and a few APIs may shift before stable. Always re-verify load-bearing details against the live [Expo docs](https://docs.expo.dev) and `npx expo install --fix` output. This is an **unofficial** project and is not affiliated with or endorsed by Expo.

---

## Install (Claude Code)

This repo is a plugin marketplace. Add it once, then install the plugin:

```
/plugin marketplace add mahdi-salmanzade/expo56-skill
/plugin install expo-56
```

After that, the **`expo-56`** skill is available and auto-triggers whenever you work in an Expo project. To update later (after new commits are pushed):

```
/plugin marketplace update expo56-skill
```

> **It's one skill, not ten.** The plugin contains a single skill (`expo-56`) backed by **20 bundled reference documents** (~11,400 lines). SKILL.md is a lightweight router that opens only the reference file relevant to your task.

---

## What it covers

A routing table in `SKILL.md` maps each topic to one of 20 reference files:

| Area | Topics |
|------|--------|
| Core / setup | versions, tooling minimums, New Architecture, project creation, **SDK 55→56 upgrade** + codemods |
| Routing | `expo-router` (Stack/Tabs/Link, typed routes, API routes, **React Navigation removal**, data loaders, SSR) |
| UI | `@expo/ui` (SwiftUI / Jetpack Compose), datetimepicker & bottom-sheet drop-ins |
| Native | Expo Modules API, Module DSL, inline modules, config plugins |
| Tooling | CLI, Metro, on-demand filesystem, tree-shaking, env vars |
| Data / IO | `expo-file-system` (async copy/move, File/Directory/Paths, upload/download tasks), `expo/fetch`, `expo-sqlite`, `expo-updates` / EAS Update |
| Device / media | redesigned Calendar/Contacts/MediaLibrary, camera, image, **`expo-video`/`expo-audio` (expo-av removed)**, location, sensors, notifications |
| Auth | `expo-secure-store`, `expo-auth-session`, `expo-local-authentication`, Router `Stack.Protected` pattern |
| System UI | StatusBar, NavigationBar, iOS Widgets (`expo-widgets`), vector-icons migration |
| Build / ship | EAS Build/Submit/Update, Workflows, brownfield, dev builds, testing (jest-expo, Maestro) |
| Misc | `expo-maps`, web output modes, and utility packages |

### Key SDK 56 breaking changes it captures
- `expo/fetch` is now the default `globalThis.fetch`
- `expo-file-system` rewritten — `copy()`/`move()` are now **async**; object-oriented `File`/`Directory`/`Paths`
- `expo-router` **no longer depends on React Navigation** (codemod available)
- `expo-av` **removed** → `expo-video` + `expo-audio`
- Calendar / Contacts / MediaLibrary **redesigned** (old APIs only via `/legacy`)
- `@expo/vector-icons` deprecated → scoped `@react-native-vector-icons/*`

---

## Verification status (as of 2026-05-22 spot-check)

Honesty matters more than coverage here. The skill carries an inline verification block:

- ✅ **Re-confirmed against live docs:** core versions (RN 0.85.2, React 19.2.3, Hermes v1, Node ≥20.19.4), `expo-contacts` (`Contact.getAllDetails` / `ContactField` / `ContactsSortOrder`), `expo-file-system` (async `copy`/`move` + `*Sync` + `File`/`Directory`/`Paths`), and `expo-widgets` (`createWidget` / `updateTimeline`).
- ⚠️ **Not yet confirmed:** the `expo-type-information` package name and its `module-interface` / `inline-modules-interface` / `short-module-interface` command names. The *capability* (CLI TypeScript generation for inline modules, watch mode) is documented, but these exact names weren't found on npm or the docs index. The skill flags them as unverified and tells the model to check `npx <pkg> --help` before relying on them.

Other known doc-vs-changelog discrepancies (e.g. `android.usePrecompiledHeaders`, the gesture-handler 2.31 `Gesture.Pan()` builder vs the 3.x `usePanGesture()` docs) are documented in the skill's **Known discrepancies** section rather than papered over.

---

## How it was built & evaluated

Docs were gathered from the [SDK 56 changelog](https://expo.dev/changelog/sdk-56) and the live Expo docs, then organized into 20 topic references with a routing `SKILL.md`.

It was benchmarked with-skill vs. a no-skill baseline. The headline finding: on **mainstream** topics the base model already does well, but on **long-tail** APIs the baseline confidently hallucinates (e.g. it denied `expo-widgets` exists), which is exactly where this skill earns its keep.

> **Methodology caveat (see `evals/evals.json`).** For the benchmark to mean anything, baseline runs **must disable web tools** — otherwise the baseline just reads live docs and the comparison measures "offline convenience," not anti-hallucination. The small 3-run samples in development showed variance too high (±~46%) to draw firm conclusions; ≥5 iterations per config with web disabled is the right bar. Treat the published deltas as directional, not definitive.

---

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog (this repo)
├── expo-56/                       # the plugin
│   ├── .claude-plugin/
│   │   └── plugin.json            # plugin manifest
│   └── skills/
│       └── expo-56/
│           ├── SKILL.md           # router + quick facts + migration checklist
│           ├── references/        # 20 topic reference docs (~11.4k lines)
│           └── evals/             # benchmark prompts, assertions, methodology
├── LICENSE
└── README.md
```

You can also browse the references directly on GitHub without installing anything.

---

## Attribution

The reference documents are **derived summaries** of Expo's official documentation (<https://docs.expo.dev>) and the SDK 56 changelog (© Expo / 650 Industries). Expo's documentation is itself MIT-licensed. This project re-organizes that material for AI-assistant consumption; for anything authoritative or current, defer to the official docs.

## License

[MIT](./LICENSE) © 2026 Mahdi Salmanzade
