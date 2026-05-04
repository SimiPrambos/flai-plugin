# flai-plugin Design Spec

**Date:** 2026-05-04
**Status:** Approved

## Overview

`flai-plugin` is a Claude Code plugin that enforces project codebase standards for Flutter development using a Clean Architecture + Riverpod tech stack. It is a documentation-only repository (no Dart/Flutter source code) — all value lives in markdown skill files and shell hook scripts.

It is based on the structure of [vgv-ai-flutter-plugin](https://github.com/VeryGoodOpenSource/vgv-ai-flutter-plugin) with VGV-brand-specific skills removed, the BLoC/layered-architecture skills replaced, and new skills added for the preferred tech stack.

---

## Tech Stack

| Concern | Package(s) |
|---|---|
| Architecture | Clean Architecture — Feature Based |
| State Management & DI | riverpod, riverpod_generator, riverpod_annotation |
| Navigation | go_router, go_router_builder (TypedGoRoute) |
| Networking | dio, retrofit, socket_io_client |
| Serialization & Models | freezed, json_serializable |
| Error Handling | fpdart |
| Local Storage | isar, shared_preferences, flutter_secure_storage |
| Environment Config | envied |
| Logging | talker, talker_flutter, talker_dio_logger, talker_riverpod_logger, talker_flutter_route_observer |
| Connectivity | connectivity_plus |
| Internationalization | intl (ARB files) |
| Code Generation | build_runner |
| Testing | mocktail, riverpod_test |

---

## Plugin Structure

```
flai-plugin/
├── .claude-plugin/
│   └── plugin.json              # name: flai-plugin, version: 0.1.0
├── .github/
│   └── workflows/
│       ├── ci.yaml              # markdown lint, spell check, skill validation
│       └── release_please.yaml  # automated versioning
├── config/
│   └── cspell.json              # custom word list for stack terminology
├── hooks/
│   ├── hooks.json               # PostToolUse: analyze + format
│   └── scripts/
│       ├── analyze.sh           # dart analyze on edited .dart files (blocking)
│       └── format.sh            # dart format on edited .dart files (non-blocking)
├── skills/
│   ├── architecture/            # NEW
│   ├── riverpod/                # NEW
│   ├── networking/              # NEW
│   ├── serialization/           # NEW
│   ├── error-handling/          # NEW
│   ├── local-storage/           # NEW
│   ├── environment-config/      # NEW
│   ├── logging/                 # NEW
│   ├── connectivity/            # NEW
│   ├── code-generation/         # NEW
│   ├── sdk-upgrade/             # ADAPTED from VGV (generic, VGV CI refs removed)
│   ├── navigation/              # ADAPTED from VGV (add TalkerRouteObserver)
│   ├── testing/                 # ADAPTED from VGV (bloc_test → riverpod_test)
│   ├── internationalization/    # ADAPTED from VGV (rebrand only)
│   ├── accessibility/           # ADAPTED from VGV (rebrand only)
│   ├── material-theming/        # ADAPTED from VGV (rebrand only)
│   ├── static-security/         # ADAPTED from VGV (adapt storage section)
│   ├── ui-package/              # ADAPTED from VGV (rebrand only)
│   └── license-compliance/      # ADAPTED from VGV (remove VGV CLI tool)
├── .mcp.json                    # dart MCP server only
├── CLAUDE.md
├── README.md
└── CHANGELOG.md
```

---

## Skill Inventory (19 skills)

### New Skills

#### `flai-architecture`
- **Trigger:** Feature folder structure, layer imports, project scaffolding
- **Key standards:**
  - `lib/core/` holds global providers, dio config, isar init, failure base classes, extensions, constants
  - `lib/features/<name>/data/`, `domain/`, `presentation/` per feature
  - No cross-feature imports except through the domain layer (repositories)
  - Domain layer is pure Dart — no Flutter imports
  - Feature-level `providers.dart` for discoverability
  - Scales to monorepo: `features/` and `core/` become separate packages, same internal structure
  - No `utils/` or `common/` catch-all folders — be specific

#### `flai-riverpod`
- **Trigger:** Adding/modifying providers, state classes, `ref.watch`, `ref.read`
- **Key standards:**
  - `@riverpod` code generation mandatory — no manual `Provider()`/`StateNotifierProvider()`
  - `AsyncNotifier<T>` for async state, `Notifier<T>` for sync state — no `StateNotifier`
  - Providers are `autoDispose` by default via code generation
  - `ref.watch()` in `build()` methods only; `ref.read()` in callbacks/event handlers only
  - Keep providers small and single-purpose
  - Expose `providers.dart` at feature level

#### `flai-networking`
- **Trigger:** Dio setup, Retrofit API files, socket_io usage
- **Key standards:**
  - Single `dioProvider` (singleton) in `core/http/`
  - `TalkerDioLogger` attached to Dio at init
  - Connectivity-aware retry interceptor on Dio (fail-fast when offline, retry on timeout)
  - Retrofit services return raw `Future<T>` — repositories wrap with `Either`
  - Socket.IO via a service class with `StreamController.broadcast()`; exposed through `StreamProvider`
  - Explicit websocket transport: `OptionBuilder().setTransports(['websocket'])`
  - Socket service disposes stream and disconnects on provider teardown

#### `flai-serialization`
- **Trigger:** Data models, JSON parsing, Freezed annotations
- **Key standards:**
  - `@freezed` for all data models and domain entities
  - Never apply `@freezed` to `Notifier`/`AsyncNotifier` state classes — use plain Dart classes or primitive types
  - Union types (sealed classes) for state variants via Freezed
  - `@JsonSerializable` via Freezed's `fromJson`/`toJson` in data layer models
  - Disable `invalid_annotation_target` in `analysis_options.yaml`
  - Pin `analyzer` version to avoid `go_router_builder` + `freezed` conflict (document in `build.yaml`)

#### `flai-error-handling`
- **Trigger:** Failure classes, Either usage, repository return types
- **Key standards:**
  - `Either<Failure, T>` propagates through data and domain layers only
  - Presentation layer converts Either to `AsyncValue` or state objects via `.fold()`
  - `TaskEither` for complex async chains that need composition
  - Sealed `Failure` class hierarchy — base `Failure` in `core/error/`, feature-specific subclasses in `features/<name>/domain/failures/`
  - Never throw exceptions across layer boundaries — wrap in `Left(Failure)`

#### `flai-local-storage`
- **Trigger:** SharedPreferences, SecureStorage, Isar usage
- **Key standards:**
  - Three-tier storage: SharedPreferences → simple non-sensitive prefs; `flutter_secure_storage` → tokens/secrets; Isar → complex entities with queries
  - Never store auth tokens or API keys in SharedPreferences
  - Single `isarProvider` (FutureProvider) in `core/database/`; service providers per entity group
  - Use `StreamProvider` wrapping Isar `.watch()` for reactive queries
  - Encryption key for SharedPreferences stored in SecureStorage when extra security needed

#### `flai-environment-config`
- **Trigger:** `.env` files, envied, API base URLs, secrets
- **Key standards:**
  - `.env` in `.gitignore` — never commit
  - `obfuscate: true` on all secret fields in `@Envied`
  - envied for local development; `--dart-define` for CI/CD release builds
  - All envied fields in `lib/core/config/env.dart`
  - Provide `.env.example` with placeholder values committed to git

#### `flai-logging`
- **Trigger:** Talker setup, logging calls, `print()` usage
- **Key standards:**
  - Single `talkerProvider` in `core/`
  - `TalkerDioLogger` on every Dio instance
  - `TalkerRiverpodObserver` in `ProviderScope` (configure to hide dispose noise)
  - `TalkerRouteObserver` on go_router
  - No `print()`, `debugPrint()`, or `log()` calls — all logging via talker
  - Use `talker.debug()` / `talker.info()` / `talker.error()` with appropriate levels

#### `flai-connectivity`
- **Trigger:** Connectivity checks, network guards, offline handling
- **Key standards:**
  - `connectivityProvider` as `StreamProvider<List<ConnectivityResult>>`
  - Never gate UI directly on raw `Connectivity` — always use the provider
  - Connectivity-aware Dio interceptor handles fail-fast (not the UI layer)
  - Distinguish between "no connection" (fail fast) and "timeout" (retry with backoff)

#### `flai-code-generation`
- **Trigger:** `build_runner`, `build.yaml`, generated files, code gen errors
- **Key standards:**
  - `build.yaml` restricts generators to relevant paths (freezed → models; riverpod_generator → notifiers)
  - Generated files (`.g.dart`, `.freezed.dart`) committed to git — never regenerate on CI
  - Never manually edit generated files
  - Pin `analyzer` version in `pubspec.yaml` to avoid `go_router_builder` + `freezed` conflict
  - Run `dart run build_runner build --delete-conflicting-outputs` after model changes
  - Watch mode for development: `dart run build_runner watch`

---

### Adapted from VGV

#### `flai-navigation` (from `vgv-navigation`)
- Keep all `@TypedGoRoute` standards and `go()` over `push()` rules
- Add: attach `TalkerRouteObserver` to go_router
- Remove: VGV branding

#### `flai-testing` (from `vgv-testing`)
- Replace `bloc_test` → `riverpod_test` + `ProviderContainer`
- Replace BLoC mocking → mock repositories via `ProviderContainer` overrides
- Keep `mocktail` for all mocking
- Add: Either mocking patterns (`Right(...)` / `Left(...)`)
- Add: note on `riverpod_lint` incompatibility with `flutter_test`
- Remove: VGV branding

#### `flai-internationalization` (from `vgv-internationalization`)
- Keep ARB + `context.l10n` extension pattern unchanged
- Remove: VGV branding

#### `flai-accessibility` (from `vgv-accessibility`)
- No content changes
- Remove: VGV branding

#### `flai-material-theming` (from `vgv-material-theming`)
- No content changes
- Remove: VGV branding

#### `flai-static-security` (from `vgv-static-security`)
- Adapt storage section to reference `flutter_secure_storage` (already in stack) and three-tier storage pattern from `flai-local-storage`
- Remove: VGV branding

#### `flai-ui-package` (from `vgv-ui-package`)
- Remove: VGV CLI MCP tool references; use `flutter create --template=package` instead
- Remove: VGV branding

#### `flai-license-compliance` (from `vgv-license-compliance`)
- Replace `mcp__very-good-cli__packages_check_licenses` with `dart pub deps --json` + manual audit
- Remove: VGV branding

#### `flai-sdk-upgrade` (from `vgv-dart-flutter-sdk-upgrade`)
- Keep SDK versioning guidance (CI `^MAJOR.MINOR.x`, pubspec `^MAJOR.MINOR.PATCH`)
- Remove: VGV CI workflow references; generalize for standard GitHub Actions Flutter setup
- Remove: VGV branding

---

### Removed from VGV

| Skill | Reason |
|---|---|
| `vgv-bloc` | Replaced by `flai-riverpod` |
| `vgv-layered-architecture` | Replaced by `flai-architecture` |
| `vgv-create-project` | Very Good CLI specific — no equivalent needed |
| `vgv-very-good-analysis-upgrade` | VGV brand specific — no equivalent needed |

---

## Hooks

Two PostToolUse hooks — identical in behavior to VGV's quality hooks:

```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "matcher": "Edit|Write",
      "script": "hooks/scripts/analyze.sh"
    },
    {
      "event": "PostToolUse",
      "matcher": "Edit|Write",
      "script": "hooks/scripts/format.sh"
    }
  ]
}
```

- `analyze.sh` — runs `dart analyze <file>` on modified `.dart` files; exits 2 on failure (blocking)
- `format.sh` — runs `dart format <file>` on modified `.dart` files; always exits 0 (non-blocking)
- Both skip gracefully if `jq` is not installed
- Both skip non-`.dart` files silently

**Removed from VGV:** `warn-missing-mcp.sh`, `check-vgv-cli.sh`, `block-cli-workarounds.sh`

---

## MCP Integration

Dart MCP server only:

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}
```

**Removed from VGV:** `very-good-cli` MCP server.

---

## Plugin Manifest

```json
{
  "name": "flai-plugin",
  "version": "0.1.0",
  "keywords": [
    "accessibility", "analyze", "architecture", "clean-architecture",
    "code-generation", "connectivity", "dart", "dio", "environment-config",
    "error-handling", "feature-based", "flutter", "format", "fpdart",
    "freezed", "go-router", "hooks", "i18n", "internationalization",
    "isar", "json-serializable", "l10n", "license-compliance",
    "local-storage", "logging", "material-3", "mobile-security",
    "navigation", "networking", "retrofit", "riverpod", "sdk-upgrade",
    "secure-storage", "serialization", "shared-preferences",
    "socket-io", "talker", "testing", "theming", "typed-go-route",
    "ui-package", "wcag", "widget-testing"
  ]
}
```

---

## CI/CD

### `ci.yaml` jobs
1. **Markdown lint** — `markdownlint-cli2` on all `.md` files except CHANGELOG
2. **Spell check** — `cspell` with `config/cspell.json` (custom words: riverpod, isar, envied, fpdart, talker, freezed, retrofit, dio, etc.)
3. **Detect changed skills** — git diff to identify modified skill directories
4. **Validate skills** — `Flash-Brew-Digital/validate-skill` on changed skills
5. **Plugin validation** — `claude plugin validate .`

### `release_please.yaml`
Automated versioning via `googleapis/release-please-action@v5`.

---

## Skill File Format

Every `SKILL.md` uses this frontmatter:

```yaml
---
name: flai-<skill-name>
description: <when to invoke this skill>
allowed-tools: Read Glob Grep [additional tools]
model: haiku|sonnet
effort: low|medium|high
---
```

Standards are written as directives (never "consider" or "you might") with code examples, anti-patterns, and references subdirectories for deep-dive content.

---

## Known Issues / Caveats

- **go_router_builder + freezed analyzer conflict** — pin `analyzer` version in `pubspec.yaml`; document in `flai-code-generation` and `flai-serialization` skills
- **riverpod_lint + flutter_test incompatibility** — document in `flai-testing` skill; recommend using analyzer-based checks instead
- **Isar v3 vs v4 API differences** — `flai-local-storage` should specify which Isar version its examples target
