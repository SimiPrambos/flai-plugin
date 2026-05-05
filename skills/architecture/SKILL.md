---
name: flai-architecture
description: Clean Architecture (Feature-Based) structure for Flutter. Use when creating new features, setting up project structure, organizing imports between layers, reviewing folder layout, or scaffolding a new Flutter project.
allowed-tools: Read Glob Grep
model: sonnet
effort: high
---

# Clean Architecture (Feature-Based)

Feature-based Clean Architecture for Flutter — each feature owns its data, domain, and presentation layers. A shared `core/` layer holds global configuration and cross-cutting concerns.

## Core Standards

Apply these standards to ALL architectural work:

- **`lib/core/` for global concerns** — dio config, isar init, base failure classes, extensions, constants, and global providers live here; nothing feature-specific
- **`lib/features/<name>/` for each feature** — every feature has its own `data/`, `domain/`, and `presentation/` subdirectories
- **No cross-feature imports** — features never import directly from other features; all shared data flows through the domain layer (repository abstractions)
- **Domain layer is pure Dart** — no Flutter or package imports in `domain/`; only Dart SDK and fpdart
- **Feature-level `providers.dart`** — each feature exposes its Riverpod providers from a single file for discoverability
- **No `utils/` or `common/` catch-all folders** — every file belongs to a specific layer in a specific feature or to `core/`
- **Dependency direction is inward** — `presentation` depends on `domain`; `data` depends on `domain`; `domain` depends on nothing

## Directory Structure

```text
lib/
├── core/
│   ├── config/
│   │   └── env.dart              # envied environment config
│   ├── database/
│   │   └── isar_provider.dart    # single Isar instance provider
│   ├── error/
│   │   └── failures.dart         # base Failure sealed class
│   ├── extensions/               # Dart/Flutter extension methods
│   ├── http/
│   │   └── dio_provider.dart     # singleton Dio provider
│   ├── l10n/                     # generated localization files
│   ├── router/
│   │   └── router.dart           # go_router configuration
│   └── providers.dart            # global providers (talker, connectivity)
│
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/
│   │   │   │   ├── auth_remote_datasource.dart
│   │   │   │   └── auth_local_datasource.dart
│   │   │   ├── models/
│   │   │   │   └── user_model.dart          # @freezed + @JsonSerializable
│   │   │   └── repositories/
│   │   │       └── auth_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   └── user.dart                # @freezed, no JSON
│   │   │   ├── failures/
│   │   │   │   └── auth_failure.dart        # sealed AuthFailure extends Failure
│   │   │   └── repositories/
│   │   │       └── auth_repository.dart     # abstract interface
│   │   ├── presentation/
│   │   │   ├── notifiers/
│   │   │   │   └── auth_notifier.dart       # @riverpod AsyncNotifier
│   │   │   ├── pages/
│   │   │   │   └── login_page.dart
│   │   │   └── widgets/
│   │   └── providers.dart                   # feature provider exports
│   │
│   └── <other_features>/
│
└── main.dart
```

## Layer Responsibilities

### Data Layer

Owns all I/O. Communicates with APIs, databases, and device storage. Converts raw data (JSON, Isar objects) into domain entities.

- `datasources/` — remote (Retrofit/Dio) and local (Isar/SharedPreferences/SecureStorage) data sources
- `models/` — `@freezed` classes with `@JsonSerializable` for JSON serialization
- `repositories/` — implements the domain repository abstraction; wraps data source calls in `Either<Failure, T>`

### Domain Layer

Pure business logic. No Flutter or external package dependencies (except `fpdart`).

- `entities/` — `@freezed` classes without JSON annotations
- `failures/` — sealed failure subclasses specific to this feature
- `repositories/` — abstract repository interfaces returning `Future<Either<Failure, T>>`

### Presentation Layer

UI and state. Consumes domain repositories via Riverpod.

- `notifiers/` — `@riverpod` `AsyncNotifier<T>` or `Notifier<T>` classes; convert `Either` results to `AsyncValue`
- `pages/` — full-screen widgets registered as GoRouter routes
- `widgets/` — reusable widgets scoped to this feature

## core/ Contents

| File/Directory | Purpose |
| --- | --- |
| `core/config/env.dart` | envied `Env` class with all environment variables |
| `core/database/isar_provider.dart` | Single `FutureProvider<Isar>` initialized once |
| `core/error/failures.dart` | Sealed `Failure` base class |
| `core/http/dio_provider.dart` | Singleton `dioProvider` with all interceptors attached |
| `core/router/router.dart` | GoRouter instance with all top-level `@TypedGoRoute` definitions |
| `core/providers.dart` | `talkerProvider`, `connectivityProvider`, and other global providers |

## Monorepo Scaling

When the project grows beyond a single app, move `core/` and each `features/<name>/` into separate Dart packages under a `packages/` directory. The internal `data/domain/presentation` structure stays identical — only the package boundaries change.

```text
packages/
├── core/                    # shared utilities and config
├── feature_auth/            # auth feature package
│   ├── lib/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── pubspec.yaml
└── feature_home/
apps/
└── flutter_app/             # thin shell, wires packages together
```

Use path dependencies during development:

```yaml
dependencies:
  feature_auth:
    path: ../../packages/feature_auth
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `features/auth/` importing from `features/profile/` | Creates circular dependencies; breaks feature isolation | Move shared data to a domain entity and have both features depend on the same repository abstraction |
| Flutter imports in `domain/` | Domain becomes tied to the Flutter SDK; can't test without Flutter | Domain uses only Dart SDK and `fpdart` |
| Single `repositories/` folder at root | Loses feature boundaries; hard to navigate as the codebase grows | Each feature owns its own repository implementation |
| `utils/` folder at root | Becomes a catch-all; obscures responsibility | Name folders by specific purpose: `extensions/`, `constants/`, `formatters/` |
| Business logic in presentation notifiers | Duplicates logic; hard to test without UI | Move logic to domain use cases or repository methods |
