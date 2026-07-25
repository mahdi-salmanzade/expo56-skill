# expo-56

An offline, **version-pinned** reference for **Expo SDK 56 and SDK 57**, packaged as a [Claude Code](https://code.claude.com) skill and distributed as a plugin marketplace.

It exists for one reason: **both SDKs postdate current model training**, so when an AI assistant writes Expo code from memory it produces plausible-but-wrong APIs — inventing `expo-av`, missing that `expo-file-system` `copy()`/`move()` are async, or that `expo-router` dropped React Navigation. This skill grounds the model in the actual SDK surface, for **building on either release** and for the **55→56** and **56→57** migrations.

| | SDK 57 (current) | SDK 56 (previous, supported) |
|---|---|---|
| Released | 2026-07-08 | 2026-06-01 |
| Latest patch | 57.0.8 | 56.0.17 |
| React Native | 0.86.0 | 0.85.3 |
| Node.js | ≥ 22.13 | ≥ 20.19.4 |

> ⚠️ **Snapshot, not a live mirror.** Audited against the `expo/expo` monorepo at commit `e84a707b28f` and npm dist-tags on **2026-07-25**. Package versions and APIs move; re-verify load-bearing details against the live [Expo docs](https://docs.expo.dev) and `npx expo install --fix`. This is an **unofficial** project, not affiliated with or endorsed by Expo.

---

## Install (Claude Code)

This repo is a plugin marketplace. Add it once, then install the plugin:

```
/plugin marketplace add mahdi-salmanzade/expo56-skill
/plugin install expo-56
```

The **`expo-56`** skill then auto-triggers whenever you work in an Expo project. To update later:

```
/plugin marketplace update expo56-skill
```

> **The plugin ID stays `expo-56`** even though it now covers 57, so existing installs keep working.

> **It's one skill, not ten.** A single skill (`expo-56`) backed by **21 bundled reference documents**. `SKILL.md` is a lightweight router that opens only the reference relevant to your task.

---

## How it works

`SKILL.md` makes the model do one thing first: **establish which SDK the project is actually on** (`package.json` → `expo`). A correct answer for the wrong SDK is still a wrong answer, and users routinely misname their version.

From there it routes to one of 21 references. Each reference body documents **SDK 56**; each ends with a **`## SDK 57 delta`** section covering what changed. Where SDK 57 changed nothing for a domain, the section says so explicitly — that's a useful signal, not a gap, because it tells the model it can trust the body as-is.

The alternative — duplicating all 20 references for SDK 57 — would double maintenance and rot within one patch cycle.

---

## What it covers

| Area | Topics |
|------|--------|
| Core / setup | versions, tooling minimums, New Architecture, project creation, **55→56 upgrade** + codemods |
| Routing | `expo-router` (Stack/Tabs/Link, typed routes, API routes, **React Navigation removal**, data loaders, SSR) |
| UI | `@expo/ui` (SwiftUI / Jetpack Compose), datetimepicker & bottom-sheet drop-ins |
| Native | Expo Modules API, Module DSL, inline modules, `expo-type-information`, config plugins |
| Tooling | CLI, Metro, on-demand filesystem, tree-shaking, env vars, `prebuild` semantics |
| Data / IO | `expo-file-system`, `expo/fetch`, `expo-sqlite`, `expo-updates` / EAS Update, runtime-version policy |
| Device / media | redesigned Calendar/Contacts/MediaLibrary, camera, image, `expo-video`/`expo-audio`, location, sensors, notifications |
| Auth | `expo-secure-store`, `expo-auth-session`, `expo-local-authentication`, Router `Stack.Protected` |
| System UI | StatusBar, NavigationBar, iOS Widgets, vector-icons migration |
| Build / ship | EAS Build/Submit/Update, Workflows, brownfield, dev builds, testing (jest-expo, Maestro) |
| Misc | `expo-maps`, web output modes, utility packages |
| **Migration** | **`21-sdk-57-migration.md`** — full 56→57 breaking-change table, ordered steps, rollback |

### Key breaking changes it captures

**SDK 56:** `expo/fetch` as the default `globalThis.fetch` · `expo-file-system` rewritten (async `copy`/`move`, `File`/`Directory`/`Paths`) · `expo-router` off React Navigation · `expo-av` removed → `expo-video` + `expo-audio` · Calendar/Contacts/MediaLibrary redesigned · `@expo/vector-icons` → `@react-native-vector-icons/*`

**SDK 57:** Node ≥22.13 (CI on Node 20 fails at install) · iOS **UIKit scene lifecycle** (`SceneDelegate.swift`; source-patching AppDelegate plugins break) · `expo prebuild` **cleans by default** · runtime-version/fingerprint change that **silently breaks OTA matching** for projects using `ios.version`/`android.version` · reanimated 4.5.0 / worklets 0.10.0 · Android R8 optimization on by default · silent default flips in `expo-camera` (`pictureSize`) and `expo-video` (`audioMixingMode`)

---

## Relationship to Expo's official skills

Expo publishes ~21 [official skills](https://docs.expo.dev/skills/) (`/plugin install expo@claude-plugins-official`). **They're task playbooks for the newest SDK; this is a version-pinned API reference.** They compose — `SKILL.md` names where to defer to them (UI construction, native modules, brownfield, store submission, EAS workflows, NativeWind, DOM components) and where this skill is the only coverage (location, sensors, calendar, contacts, maps, notifications, secure-store, auth-session, the `app.json` schema, config-plugin authoring, Metro config, jest-expo, Maestro).

It also documents, with source citations, four points where the official skills are currently stale: Hermes v1 is the **default** since SDK 56 (not opt-in); `@expo/vector-icons` migrates to `@react-native-vector-icons/*` (not `expo-symbols`); `expo-linear-gradient` is **not** deprecated; and Expo Go shouldn't be the default dev target.

Where the official skills were right and this one was wrong, it was corrected — Expo Router data loaders are **SDK 55+ and alpha**, not SDK 56.

---

## Verification status (2026-07-25 audit)

Honesty matters more than coverage. Every version pin traces to the frozen versioned schemas (`docs/public/static/schemas/v56.0.0|v57.0.0/native-modules.json`) — the only authoritative source for what an SDK pins.

- ✅ **`expo-type-information` confirmed real** (previously flagged unverified): package at `packages/expo-type-information`, CLI binary `expo-type-information`, commands `module-interface` / `inline-modules-interface` / `short-module-interface`.
- ✅ **`android.usePrecompiledHeaders` confirmed real** — in `expo-build-properties`' `pluginConfig.ts` and implemented in `android.ts`, merely absent from the auto-generated docs page.
- ⚠️ **`packages/*/package.json` in the monorepo is not authoritative** for either SDK — `main` mixes versions (`packages/expo/package.json` still reads `56.0.5` while templates pin RN 0.86). Only the frozen schemas are.
- ⚠️ **The v57 versioned docs are a beta cut-off (2026-06-30)** while v56 docs kept receiving backports through July. In places the v56 page is *newer* than its v57 counterpart, so a v56↔v57 docs diff is **not** evidence of an API change.

Every SDK 57 claim was written from the changelog state **at the release cut-off commit**, not from `main` — several `@expo/fingerprint` features on `main` (presets, `package` source type) landed *after* the cut and are **not** in SDK 57.

---

## How it was built

References were compiled from Expo's docs and changelogs, then audited against a local `expo/expo` clone: a research pass over the monorepo, per-file updates, and an adversarial fact-check of every changed line against package source, `CHANGELOG.md`, and the frozen schemas.

It was benchmarked with-skill vs. a no-skill baseline. Headline finding: on **mainstream** topics the base model already does well; on **long-tail** APIs the baseline confidently hallucinates (it denied `expo-widgets` exists). That's where this earns its keep.

> **Methodology caveat (see `evals/evals.json`).** Baseline runs **must disable web tools** — otherwise the baseline just reads live docs and the comparison measures "offline convenience," not anti-hallucination. Small 3-run samples showed variance too high (±~46%) for firm conclusions; ≥5 iterations per config with web disabled is the right bar. Treat published deltas as directional.

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
│           ├── SKILL.md           # SDK detection + router + quick facts + migrations
│           ├── references/        # 21 topic reference docs
│           └── evals/             # benchmark prompts, assertions, methodology
├── LICENSE
└── README.md
```

You can browse the references directly on GitHub without installing anything.

---

## Attribution

The reference documents are **derived summaries** of Expo's official documentation (<https://docs.expo.dev>) and the SDK 56/57 changelogs (© Expo / 650 Industries). Expo's documentation is itself MIT-licensed. This project re-organizes that material for AI-assistant consumption; for anything authoritative or current, defer to the official docs.

## License

[MIT](./LICENSE) © 2026 Mahdi Salmanzade
