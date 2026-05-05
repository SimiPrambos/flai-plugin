---
name: flai-vgv-migration
description: Migrate a VeryGood CLI Flutter project to flai standards. Use when the project was scaffolded with `very_good create` and needs refactoring into flai Clean Architecture + Riverpod stack.
allowed-tools: Read Glob Grep Bash Edit Write
model: sonnet
effort: high
---

# VGV → flai Migration

Step-by-step migration from VeryGood CLI scaffold to flai Clean Architecture + Riverpod + flai tech stack.

## Core Standards

Apply these standards throughout the migration:

- **Follow the migration steps in order** — each step depends on the previous one
- **Preserve VGV infrastructure** — keep CI/CD, test helpers, analysis_options.yaml, l10n setup
- **Replace all Bloc/Cubit usage** — Riverpod with `@riverpod` code generation only
- **Restructure into `lib/core/` + `lib/features/`** — flat VGV structure becomes feature-based Clean Architecture
- **Add the full flai package stack** — Dio, Retrofit, Freezed, fpdart, Isar, GoRouter, Talker, envied
- **Generate CLAUDE.md as the final step** — the enforcement contract for the migrated project

## Pre-Migration Checklist

Verify the project was created with `very_good create` before proceeding:

```bash
ls lib/app/view/app.dart
ls lib/counter/
grep -q "flutter_bloc" pubspec.yaml && echo "VGV project confirmed"
```

All three checks must pass. If any fails, this is not a VGV scaffold — invoke `flai-architecture` instead.

## Migration Steps

### Step 1: Update pubspec.yaml

Remove VGV Bloc packages and add the full flai stack.

**Remove from dependencies:**
```yaml
# Remove these
bloc:
flutter_bloc:
```

**Remove from dev_dependencies:**
```yaml
# Remove these
bloc_test:
mocktail:  # Keep if already present, flai uses it too
```

**Add to dependencies:**
```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  # State management
  flutter_riverpod:
  riverpod_annotation:

  # Networking
  dio:
  retrofit:

  # Serialization
  freezed_annotation:
  json_annotation:

  # Functional programming
  fpdart:

  # Storage
  shared_preferences:
  flutter_secure_storage:
  isar:

  # Navigation
  go_router:

  # Logging
  talker:
  talker_dio_logger:
  talker_riverpod_logger:
  talker_flutter:

  # Environment config
  envied:

  # Connectivity
  connectivity_plus:

  # Localization
  intl:
```

**Add to dev_dependencies:**
```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner:
  riverpod_generator:
  freezed:
  json_serializable:
  retrofit_generator:
  envied_generator:
  isar_generator:
  riverpod_lint:
  custom_lint:
  mocktail:
```

Run `flutter pub get` after updating.

### Step 2: Restructure lib/ Directory

Remove the VGV flat structure first:
```bash
rm -rf lib/app
rm -rf lib/counter
```

Create the flai directory structure:

```bash
mkdir -p lib/core/config
mkdir -p lib/core/database
mkdir -p lib/core/error
mkdir -p lib/core/extensions
mkdir -p lib/core/http
mkdir -p lib/core/l10n
mkdir -p lib/core/router
mkdir -p lib/features/counter/data/repositories
mkdir -p lib/features/counter/domain/entities
mkdir -p lib/features/counter/domain/failures
mkdir -p lib/features/counter/domain/repositories
mkdir -p lib/features/counter/presentation/notifiers
mkdir -p lib/features/counter/presentation/pages
mkdir -p lib/features/counter/presentation/widgets
mkdir -p lib/app
```

### Step 3: Implement Core Layer

#### lib/core/error/failures.dart

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'failures.freezed.dart';

@freezed
sealed class Failure with _$Failure {
  const factory Failure.network({
    required String message,
    int? statusCode,
  }) = NetworkFailure;

  const factory Failure.storage({
    required String message,
  }) = StorageFailure;

  const factory Failure.validation({
    required String message,
  }) = ValidationFailure;

  const factory Failure.unknown({
    required String message,
    Object? error,
    StackTrace? stackTrace,
  }) = UnknownFailure;
}
```

#### lib/core/config/env.dart

```dart
import 'package:envied/envied.dart';

part 'env.g.dart';

@Envied(path: '.env', obfuscate: true)
abstract class Env {
  @EnviedField(varName: 'API_BASE_URL', obfuscate: true)
  static final String apiBaseUrl = _Env.apiBaseUrl;

  @EnviedField(varName: 'SENTRY_DSN', obfuscate: true)
  static final String sentryDsn = _Env.sentryDsn;
}
```

Create `.env.example`:
```
API_BASE_URL=https://api.example.com
SENTRY_DSN=
```

#### lib/core/http/dio_provider.dart

```dart
import 'package:dio/dio.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:talker_dio_logger/talker_dio_logger.dart';

import '../config/env.dart';

part 'dio_provider.g.dart';

@Riverpod(keepAlive: true)
Dio dio(DioRef ref) {
  final dio = Dio(
    BaseOptions(
      baseUrl: Env.apiBaseUrl,
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
    ),
  );

  dio.interceptors.add(
    TalkerDioLogger(
      talker: ref.watch(talkerProvider),
    ),
  );

  return dio;
}
```

#### lib/core/providers.dart

```dart
export 'config/env.dart';
export 'error/failures.dart';
export 'http/dio_provider.dart';
```

#### lib/core/router/router.dart

```dart
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../features/counter/presentation/pages/counter_page.dart';

part 'router.g.dart';

@Riverpod(keepAlive: true)
GoRouter goRouter(GoRouterRef ref) {
  return GoRouter(
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const CounterPage(),
      ),
    ],
  );
}
```

### Step 4: Implement App Shell

#### lib/app/app.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../core/router/router.dart';

class App extends ConsumerWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(goRouterProvider);

    return MaterialApp.router(
      title: 'flai Starter',
      routerConfig: router,
    );
  }
}
```

#### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'app/app.dart';

void main() {
  runApp(
    const ProviderScope(
      child: App(),
    ),
  );
}
```

### Step 5: Implement Counter Feature

#### lib/features/counter/domain/entities/counter_state.dart

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'counter_state.freezed.dart';

@freezed
class CounterState with _$CounterState {
  const factory CounterState({
    required int count,
  }) = _CounterState;

  const CounterState._();
}
```

#### lib/features/counter/domain/failures/counter_failure.dart

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'counter_failure.freezed.dart';

@freezed
sealed class CounterFailure with _$CounterFailure {
  const factory CounterFailure.overflow({
    required String message,
  }) = CounterOverflowFailure;
}
```

#### lib/features/counter/domain/repositories/counter_repository.dart

```dart
import 'package:fpdart/fpdart.dart';

import '../../../../core/error/failures.dart';
import '../entities/counter_state.dart';

abstract interface class CounterRepository {
  Future<Either<Failure, CounterState>> increment(int current);
  Future<Either<Failure, CounterState>> decrement(int current);
  Future<Either<Failure, CounterState>> reset();
}
```

#### lib/features/counter/data/repositories/counter_repository_impl.dart

```dart
import 'package:fpdart/fpdart.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../../core/error/failures.dart';
import '../../domain/entities/counter_state.dart';
import '../../domain/repositories/counter_repository.dart';

part 'counter_repository_impl.g.dart';

class CounterRepositoryImpl implements CounterRepository {
  @override
  Future<Either<Failure, CounterState>> increment(int current) async {
    return Right(CounterState(count: current + 1));
  }

  @override
  Future<Either<Failure, CounterState>> decrement(int current) async {
    return Right(CounterState(count: current - 1));
  }

  @override
  Future<Either<Failure, CounterState>> reset() async {
    return Right(const CounterState(count: 0));
  }
}

@Riverpod(keepAlive: true)
CounterRepository counterRepository(CounterRepositoryRef ref) {
  return CounterRepositoryImpl();
}
```

#### lib/features/counter/presentation/notifiers/counter_notifier.dart

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../data/repositories/counter_repository_impl.dart';
import '../../domain/entities/counter_state.dart';

part 'counter_notifier.g.dart';

@riverpod
class CounterNotifier extends _$CounterNotifier {
  @override
  CounterState build() => const CounterState(count: 0);

  Future<void> increment() async {
    final repo = ref.read(counterRepositoryProvider);
    final result = await repo.increment(state.count);
    result.match(
      (l) => {/* handle failure */},
      (r) => state = r,
    );
  }

  Future<void> decrement() async {
    final repo = ref.read(counterRepositoryProvider);
    final result = await repo.decrement(state.count);
    result.match(
      (l) => {/* handle failure */},
      (r) => state = r,
    );
  }

  Future<void> reset() async {
    final repo = ref.read(counterRepositoryProvider);
    final result = await repo.reset();
    result.match(
      (l) => {/* handle failure */},
      (r) => state = r,
    );
  }
}
```

#### lib/features/counter/presentation/pages/counter_page.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../notifiers/counter_notifier.dart';

class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterNotifierProvider).count;

    return Scaffold(
      appBar: AppBar(title: const Text('Counter')),
      body: Center(
        child: Text(
          '$count',
          style: Theme.of(context).textTheme.displayLarge,
        ),
      ),
      floatingActionButton: Column(
        mainAxisAlignment: MainAxisAlignment.end,
        children: [
          FloatingActionButton(
            heroTag: 'increment',
            onPressed: () =>
                ref.read(counterNotifierProvider.notifier).increment(),
            child: const Icon(Icons.add),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'decrement',
            onPressed: () =>
                ref.read(counterNotifierProvider.notifier).decrement(),
            child: const Icon(Icons.remove),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'reset',
            onPressed: () =>
                ref.read(counterNotifierProvider.notifier).reset(),
            child: const Icon(Icons.refresh),
          ),
        ],
      ),
    );
  }
}
```

#### lib/features/counter/providers.dart

```dart
export 'domain/entities/counter_state.dart';
export 'domain/failures/counter_failure.dart';
export 'domain/repositories/counter_repository.dart';
export 'data/repositories/counter_repository_impl.dart';
export 'presentation/notifiers/counter_notifier.dart';
```

### Step 6: Update build.yaml

Create or update `build.yaml`:

```yaml
targets:
  $default:
    builders:
      freezed:
        generate_for:
          include:
            - lib/**/*.dart
      json_serializable:
        generate_for:
          include:
            - lib/**/*.dart
      riverpod_generator:
        generate_for:
          include:
            - lib/**/*.dart
```

### Step 7: Run Code Generation

```bash
dart run build_runner build --delete-conflicting-outputs
```

### Step 8: Adapt Test Helpers

Update `test/helpers/` to use Riverpod instead of Bloc.

Replace `pumpApp` helper:

```dart
// test/helpers/pump_app.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';

extension WidgetTesterX on WidgetTester {
  Future<void> pumpApp(
    Widget widget, {
    List<Override> overrides = const [],
  }) async {
    await pumpWidget(
      ProviderScope(
        overrides: overrides,
        child: MaterialApp(home: widget),
      ),
    );
  }
}
```

### Step 9: Generate CLAUDE.md

Create `CLAUDE.md` in the project root with the following content:

````markdown
# CLAUDE.md

## Project Standard

This project follows the **flai Flutter standard** — Clean Architecture with Riverpod, fpdart, and the flai tech stack.

## Required Plugin

Install the flai-plugin before working on this project:

```bash
claude plugin install flai-plugin
```

**If the plugin is not installed, STOP. Do not generate code without it.**

## Mandatory Skill Invocation

Before writing ANY code, you MUST invoke the relevant skill:

| Task | Required Skill |
|------|----------------|
| Creating features, organizing folders, reviewing structure | `flai-architecture` |
| State management, providers, notifiers | `flai-riverpod` |
| HTTP APIs, WebSocket, Dio, Retrofit | `flai-networking` |
| Data models, JSON, Freezed | `flai-serialization` |
| Error handling, Either, Failure classes | `flai-error-handling` |
| SharedPreferences, SecureStorage, Isar | `flai-local-storage` |
| Environment variables, secrets | `flai-environment-config` |
| Logging with Talker | `flai-logging` |
| Network connectivity checks | `flai-connectivity` |
| build_runner, code generation | `flai-code-generation` |
| GoRouter, navigation, deep linking | `flai-navigation` |
| Unit tests, widget tests, golden tests | `flai-testing` |
| i18n, l10n, ARB files | `flai-internationalization` |
| WCAG, semantics, screen readers | `flai-accessibility` |
| Material 3, theming, ColorScheme | `flai-material-theming` |
| Security, secrets, certificate pinning | `flai-static-security` |
| Reusable UI packages | `flai-ui-package` |
| Dart/Flutter SDK upgrades | `flai-sdk-upgrade` |

**When in doubt:** Invoke `flai-architecture` first. It defines the foundation all other skills assume.

## Hard Rules

These rules are NON-NEGOTIABLE:

### Architecture
- ✅ ALWAYS use `lib/core/` for global concerns
- ✅ ALWAYS use `lib/features/<name>/` with `data/`, `domain/`, `presentation/` subdirectories
- ❌ NEVER create cross-feature imports
- ❌ NEVER put Flutter imports in `domain/` layer
- ❌ NEVER create `utils/` or `common/` catch-all folders

### State Management
- ✅ ALWAYS use Riverpod with `@riverpod` code generation
- ✅ ALWAYS use `AsyncNotifier` or `Notifier` for state
- ❌ NEVER use Bloc, Cubit, Provider, or setState
- ❌ NEVER manage state in StatefulWidget

### Error Handling
- ✅ ALWAYS return `Either<Failure, T>` from repositories
- ✅ ALWAYS use sealed `Failure` subclasses
- ❌ NEVER throw exceptions from repositories
- ❌ NEVER use try-catch in domain layer

### Packages
- ✅ ALWAYS use packages listed in `pubspec.yaml`
- ❌ NEVER add packages without explicit approval
- ❌ NEVER use packages that conflict with the flai stack (e.g., GetX, Provider, http instead of Dio)

### Code Quality
- ✅ ALWAYS run `dart analyze` before considering work complete
- ✅ ALWAYS run `dart format` on modified files
- ✅ ALWAYS write tests for new features
- ❌ NEVER commit code that fails `dart analyze`

## Verification

Before claiming any task is complete:
1. Invoke the relevant skill(s)
2. Generate/modify code following skill patterns
3. Run `dart analyze` — must pass
4. Run `dart format` — must pass
5. Run tests if applicable — must pass

## Questions?

If you're unsure which skill applies or how to implement something within flai standards, ASK before generating code. Do not guess or fall back to generic Flutter patterns.
````

### Step 10: Final Validation

Run these commands and confirm all pass:

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
dart analyze
dart format --set-exit-if-changed lib/ test/
flutter test
flutter run -d macos  # or your preferred device
```

All commands must succeed before the migration is considered complete.

## What to Preserve from VGV

Do NOT remove or modify these VGV artifacts:

- `.github/workflows/` — CI/CD pipelines
- `test/helpers/` — test utility files (adapt for Riverpod, keep structure)
- `analysis_options.yaml` — VGV lint rules (supplement, do not replace)
- `l10n.yaml` and `lib/l10n/` or l10n configuration — localization setup
- `.gitignore` — VGV's comprehensive ignore file

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| Skipping pre-migration check | Applying migration to non-VGV project causes unpredictable results | Always verify VGV scaffold first |
| Keeping Bloc references | Mixed Bloc/Riverpod code confuses developers | Replace ALL Bloc/Cubit usage |
| Removing CI/CD workflows | Loses automated testing and deployment | Preserve all `.github/workflows/` |
| Empty core/ files | Placeholder files give false confidence | Only create files with real implementations |
| Skipping CLAUDE.md generation | No enforcement mechanism in the resulting project | Always generate CLAUDE.md as the final step |
