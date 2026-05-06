# flai-plugin

> Stop reviewing AI-generated Flutter code that ignores your team's architecture. flai-plugin teaches Claude Code your stack — and enforces it on every edit.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Version](https://img.shields.io/badge/version-0.1.0-brightgreen.svg)](https://github.com/SimiPrambos/flai-plugin/releases)
[![Flutter](https://img.shields.io/badge/Flutter-Standards-02569B?logo=flutter&logoColor=white)](https://flutter.dev)
[![Dart](https://img.shields.io/badge/Dart-Plugin-0175C2?logo=dart&logoColor=white)](https://dart.dev)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-D97757)](https://claude.ai/code)

**flai-plugin** ships a curated Flutter stack — Clean Architecture, Riverpod, fpdart, Freezed, Talker, GoRouter — to [Claude Code](https://claude.ai/code) as 19 contextual skills, then enforces it with `dart analyze` and `dart format` hooks that run on every edit. AI-generated Flutter that lands on-stack the first time.

## 🤔 The Problem

Claude Code writes great Flutter. But out of the box, it doesn't write *your team's* Flutter. Without project-specific guidance, every prompt produces the same drift:

- Business logic leaking into widgets instead of use-cases or notifiers
- Ad-hoc state management instead of your chosen Riverpod patterns
- API calls without your error-handling, logging, or retry conventions
- Generated files that pass `flutter run` but fail `dart analyze` or `dart format`
- Five engineers, five subtly different solutions to the same problem

Rules files and prompt snippets help, but they're advisory — the AI ignores them when convenient, and bad edits still land in your branch.

## ✨ Why flai-plugin?

flai-plugin ships a curated Flutter stack to Claude Code as **enforceable** rules:

1. **Architecture by default** — every feature scaffolded as Clean Architecture (`data/`, `domain/`, `presentation/`) with the right layer boundaries
2. **One stack, agreed on** — Riverpod, Dio, Freezed, GoRouter, Talker, Isar — documented as 19 single-purpose skills
3. **Enforced, not suggested** — `dart analyze` runs after every edit and **blocks** on failure; `dart format` normalizes style automatically
4. **Team alignment** — every developer's Claude Code produces the same shape of code for the same prompt
5. **Scoped, low token cost** — skills load only when relevant; you don't pay tokens for accessibility rules while writing networking code
6. **Migration-friendly** — refactor an existing `very_good create` scaffold to flai standards in one prompt

## ⚙️ How It Works

- 🧠 **Skills** teach Claude *what* the conventions are — Markdown files loaded on demand based on the task
- 🛡️ **Hooks** make the conventions *stick* — `analyze.sh` exits with code 2 to **block** bad edits, `format.sh` normalizes style after every save
- 🪶 **Zero runtime cost** — pure Markdown and shell, nothing added to your `pubspec.yaml`
- ✅ **Result** — AI-generated Flutter that passes code review on the first pass

## 🧱 The flai Stack

| Layer        | Choice                                                                                |
| ------------ | ------------------------------------------------------------------------------------- |
| Architecture | Clean Architecture (Feature-Based) — [`architecture`](skills/architecture/SKILL.md)   |
| State & DI   | Riverpod with `@riverpod` codegen — [`riverpod`](skills/riverpod/SKILL.md)            |
| Networking   | Dio · Retrofit · Socket.IO — [`networking`](skills/networking/SKILL.md)               |
| Data Models  | Freezed · json_serializable — [`serialization`](skills/serialization/SKILL.md)        |
| Errors       | fpdart `Either` + sealed `Failure` — [`error-handling`](skills/error-handling/SKILL.md) |
| Storage      | SharedPreferences · flutter_secure_storage · Isar — [`local-storage`](skills/local-storage/SKILL.md) |
| Routing      | GoRouter with `@TypedGoRoute` — [`navigation`](skills/navigation/SKILL.md)            |
| Logging      | Talker — [`logging`](skills/logging/SKILL.md)                                         |
| Testing      | flutter_test · mocktail · golden — [`testing`](skills/testing/SKILL.md)               |

## 🚀 Quick Start

Install from the flai marketplace:

```bash
/plugin marketplace add SimiPrambos/flai-plugin
/plugin install flai-plugin@flai-marketplace
```

Then open a Flutter project and ask Claude — relevant skills auto-engage.

**Try these prompts:**

```
Scaffold a login feature with Riverpod and a fake repository
Migrate this VGV project to the flai stack
Add an authenticated Dio client with retry on 401
Write golden tests for the OnboardingPage
```

## 📚 Skills

Each skill is loaded contextually based on the task you ask Claude to perform.

| Skill | Description |
| ----- | ----------- |
| [**Architecture**](skills/architecture/SKILL.md) | Clean Architecture (Feature-Based) — `core/` and `features/` structure, layer boundaries, dependency rules, and monorepo scaling |
| [**Riverpod**](skills/riverpod/SKILL.md) | State management and DI with Riverpod — `@riverpod` code generation, `AsyncNotifier`/`Notifier` patterns, and provider organization |
| [**Networking**](skills/networking/SKILL.md) | HTTP with Dio and Retrofit, real-time with Socket.IO — singleton Dio setup, connectivity-aware interceptors, and StreamProvider for WebSocket |
| [**Serialization**](skills/serialization/SKILL.md) | Data models with Freezed and json_serializable — immutable models, union types, JSON parsing, and analyzer version pinning |
| [**Error Handling**](skills/error-handling/SKILL.md) | Functional error handling with fpdart — `Either` propagation through layers, sealed `Failure` hierarchy, and `TaskEither` for complex chains |
| [**Local Storage**](skills/local-storage/SKILL.md) | Three-tier storage — SharedPreferences for prefs, FlutterSecureStorage for secrets, Isar for complex entities with reactive queries |
| [**Environment Config**](skills/environment-config/SKILL.md) | Secrets and configuration with envied — `obfuscate: true`, `.env.example`, and `--dart-define` for CI/CD releases |
| [**Logging**](skills/logging/SKILL.md) | Structured logging with Talker — Dio logger, Riverpod observer, route observer, and log level conventions |
| [**Connectivity**](skills/connectivity/SKILL.md) | Network status with connectivity_plus — `StreamProvider`, fail-fast Dio interceptor, and offline handling patterns |
| [**Code Generation**](skills/code-generation/SKILL.md) | build_runner — `build.yaml` path restriction, analyzer version pinning, generated files policy, and common conflict fixes |
| [**Navigation**](skills/navigation/SKILL.md) | GoRouter routing — `@TypedGoRoute` type-safe routes, deep linking, redirects, and `TalkerRouteObserver` |
| [**Testing**](skills/testing/SKILL.md) | Unit, widget, and golden tests — `ProviderContainer` overrides, `mocktail` mocking, Either test patterns, and `pumpApp` helpers |
| [**Internationalization**](skills/internationalization/SKILL.md) | i18n/l10n — ARB files, `context.l10n` patterns, pluralization, and RTL/directional layout |
| [**Accessibility**](skills/accessibility/SKILL.md) | WCAG 2.1 AA — semantics, screen reader support, touch targets, focus management, color contrast, and motion sensitivity |
| [**Material Theming**](skills/material-theming/SKILL.md) | Material 3 theming — `ColorScheme`, `TextTheme`, component themes, spacing systems, and light/dark mode |
| [**Security**](skills/static-security/SKILL.md) | Flutter-specific static security — secrets management, `flutter_secure_storage`, certificate pinning, and OWASP Mobile Top 10 |
| [**UI Package**](skills/ui-package/SKILL.md) | Reusable Flutter UI packages — custom widget libraries with `ThemeExtension` theming, barrel exports, and widget tests |
| [**SDK Upgrade**](skills/sdk-upgrade/SKILL.md) | Dart and Flutter SDK version bumps — CI wildcard pinning, pubspec exact pinning, and upgrade workflow |
| [**VGV Migration**](skills/vgv-migration/SKILL.md) | Migrate a VeryGood CLI scaffold to flai standards — refactors Bloc→Riverpod, flat→Clean Architecture, generates CLAUDE.md enforcement contract |

## 🪝 Hooks

Hooks run automatically after every `Edit` or `Write` on a `.dart` file.

| Hook | Behavior |
| ---- | -------- |
| **Analyze** | Runs `dart analyze` on the modified file; **blocking** on failure |
| **Format**  | Runs `dart format` on the modified file; non-blocking |

## 🛠️ Requirements

- [Claude Code](https://claude.ai/code) CLI
- **Dart SDK** on your `PATH`
- **jq** — used by hooks for payload parsing; hooks skip gracefully if missing

## 🤝 Contributing

Contributions are welcome. See [`CLAUDE.md`](CLAUDE.md) for the `SKILL.md` format and the "Adding a New Skill" checklist. All commits use [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, …).

## 📄 License

[MIT](https://opensource.org/licenses/MIT)