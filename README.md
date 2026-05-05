# flai-plugin

A [Claude Code][claude_code_link] plugin that enforces Flutter project standards with Clean Architecture, Riverpod state management, and the flai standard tech stack.

[claude_code_link]: https://claude.ai/code

## Overview

flai-plugin is a collection of contextual best-practices skills that Claude uses when helping you write Flutter and Dart code. Each skill provides opinionated guidance covering architecture patterns, naming conventions, folder structures, code examples, testing strategies, and anti-patterns to avoid.

## Installation

```bash
claude plugin install /path/to/flai-plugin
```

## Skills

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
| [**License Compliance**](skills/license-compliance/SKILL.md) | Dependency license auditing — permissive, copyleft, and unknown license categorization |
| [**SDK Upgrade**](skills/sdk-upgrade/SKILL.md) | Dart and Flutter SDK version bumps — CI wildcard pinning, pubspec exact pinning, and upgrade workflow |

## Hooks

| Hook | Behavior |
| ---- | -------- |
| **Analyze** | Runs `dart analyze` on modified `.dart` files after every Edit/Write; blocking on failure |
| **Format** | Runs `dart format` on modified `.dart` files after every Edit/Write; non-blocking |

### Prerequisites

- **Dart SDK** on your `PATH`
- **jq** for hook payload parsing (hooks skip gracefully if missing)
