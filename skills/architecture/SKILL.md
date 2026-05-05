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

## SOLID Principles

Clean Architecture defines where files go; SOLID defines how to design classes and functions within those layers.

### Single Responsibility Principle (SRP)

Each class has one reason to change.

**In flai architecture:**
- One notifier manages one piece of state — `AuthNotifier` handles auth, `ProfileNotifier` handles profile
- One repository handles one domain entity — `UserRepository` for users, `PostRepository` for posts
- One widget does one thing — extract into smaller widgets when a widget grows beyond 100 lines

**Example:**
```dart
// ❌ Violates SRP — notifier does API calls directly
class SongsNotifier extends _$SongsNotifier {
  @override
  Future<List<Song>> build() async {
    final dio = ref.watch(dioProvider);
    final response = await dio.get('/songs');  // WRONG: notifier doing I/O
    return (response.data as List).map((e) => Song.fromJson(e)).toList();
  }
}

// ✅ Follows SRP — notifier delegates to repository
class SongsNotifier extends _$SongsNotifier {
  @override
  Future<List<Song>> build() async {
    final repo = ref.watch(songRepositoryProvider);
    return (await repo.getSongs()).fold(
      (failure) => throw failure,
      (songs) => songs,
    );
  }
}
```

### Open/Closed Principle (OCP)

Open for extension, closed for modification.

**In flai architecture:**
- Use sealed classes for failures and states — add new variants without touching existing code
- Use `@freezed` for immutable data — extend via `copyWith`, not mutation
- Use strategy pattern via abstract interfaces when behavior varies

**Example:**
```dart
// ✅ Open for extension — add new failure types without modifying existing code
@freezed
sealed class Failure with _$Failure {
  const factory Failure.network({required String message}) = NetworkFailure;
  const factory Failure.storage({required String message}) = StorageFailure;
  // Add new failure types here without touching existing code
}

// ✅ Extend state via copyWith, not mutation
final updatedUser = currentUser.copyWith(name: 'New Name');
```

### Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without breaking behavior.

**In flai architecture:**
- Repository implementations must honor the domain interface contract
- Never throw unexpected exception types from implementations
- Return types must match exactly — if interface returns `Either<Failure, User>`, implementation cannot return `Either<Failure, dynamic>`

**Example:**
```dart
// ✅ Implementation honors the contract
abstract interface class UserRepository {
  Future<Either<Failure, User>> getUser(String id);
}

class UserRepositoryImpl implements UserRepository {
  @override
  Future<Either<Failure, User>> getUser(String id) async {
    // Always returns Either<Failure, User>, never throws
    try {
      final response = await _api.getUser(id);
      return Right(response);
    } catch (e) {
      return Left(Failure.network(message: e.toString()));
    }
  }
}

// ❌ Violates LSP — throws instead of returning Either
class BadUserRepositoryImpl implements UserRepository {
  @override
  Future<Either<Failure, User>> getUser(String id) async {
    final response = await _api.getUser(id);  // WRONG: can throw
    return Right(response);
  }
}
```

### Interface Segregation Principle (ISP)

Don't force clients to depend on interfaces they don't use.

**In flai architecture:**
- Keep repository interfaces small — one per feature, not one giant repository
- Don't add methods "just in case" — add them when actually needed
- Split large interfaces into smaller, focused ones

**Example:**
```dart
// ❌ Violates ISP — forces all clients to depend on methods they don't use
abstract interface class UserRepository {
  Future<Either<Failure, User>> getUser(String id);
  Future<Either<Failure, User>> updateUser(User user);
  Future<Either<Failure, void>> deleteUser(String id);
  Future<Either<Failure, List<Post>>> getUserPosts(String id);  // WRONG: posts belong in PostRepository
  Future<Either<Failure, List<Comment>>> getUserComments(String id);  // WRONG: comments belong in CommentRepository
}

// ✅ Follows ISP — small, focused interfaces
abstract interface class UserRepository {
  Future<Either<Failure, User>> getUser(String id);
  Future<Either<Failure, User>> updateUser(User user);
  Future<Either<Failure, void>> deleteUser(String id);
}

abstract interface class PostRepository {
  Future<Either<Failure, List<Post>>> getPostsByUser(String userId);
}

abstract interface class CommentRepository {
  Future<Either<Failure, List<Comment>>> getCommentsByUser(String userId);
}
```

### Dependency Inversion Principle (DIP)

Depend on abstractions, not concretions.

**In flai architecture:**
- Domain layer defines abstract interfaces (repositories)
- Data layer implements them
- Presentation depends on domain abstractions, never data layer directly
- Use Riverpod providers to inject implementations

**Example:**
```dart
// ✅ Domain defines abstraction
// features/songs/domain/repositories/song_repository.dart
abstract interface class SongRepository {
  Future<Either<Failure, List<Song>>> getSongs();
}

// ✅ Data implements abstraction
// features/songs/data/repositories/song_repository_impl.dart
class SongRepositoryImpl implements SongRepository {
  final SongRemoteDataSource _remote;
  SongRepositoryImpl(this._remote);

  @override
  Future<Either<Failure, List<Song>>> getSongs() async {
    // implementation
  }
}

// ✅ Presentation depends on abstraction via provider
// features/songs/presentation/notifiers/songs_notifier.dart
@riverpod
class SongsNotifier extends _$SongsNotifier {
  @override
  Future<List<Song>> build() async {
    final repo = ref.watch(songRepositoryProvider);  // depends on abstraction
    return (await repo.getSongs()).fold(
      (failure) => throw failure,
      (songs) => songs,
    );
  }
}

// ❌ Violates DIP — presentation depends on concrete implementation
@riverpod
class BadSongsNotifier extends _$BadSongsNotifier {
  @override
  Future<List<Song>> build() async {
    final impl = SongRepositoryImpl(SongRemoteDataSource());  // WRONG: depends on concrete class
    return (await impl.getSongs()).fold(
      (failure) => throw failure,
      (songs) => songs,
    );
  }
}
```

## How SOLID Reinforces Clean Architecture

| SOLID Principle | Clean Architecture Benefit |
| --- | --- |
| SRP | Each layer has one responsibility; each file within a layer has one job |
| OCP | Add new features without modifying existing layers |
| LSP | Repository implementations are interchangeable (mock in tests, real in prod) |
| ISP | Small, focused repository interfaces per feature |
| DIP | Domain layer is independent; data and presentation depend on it |

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `features/auth/` importing from `features/profile/` | Creates circular dependencies; breaks feature isolation | Move shared data to a domain entity and have both features depend on the same repository abstraction |
| Flutter imports in `domain/` | Domain becomes tied to the Flutter SDK; can't test without Flutter | Domain uses only Dart SDK and `fpdart` |
| Single `repositories/` folder at root | Loses feature boundaries; hard to navigate as the codebase grows | Each feature owns its own repository implementation |
| `utils/` folder at root | Becomes a catch-all; obscures responsibility | Name folders by specific purpose: `extensions/`, `constants/`, `formatters/` |
| Business logic in presentation notifiers | Duplicates logic; hard to test without UI | Move logic to domain use cases or repository methods |
