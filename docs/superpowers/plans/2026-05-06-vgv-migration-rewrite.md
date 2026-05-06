# VGV Migration Rewrite — Battery-Included Orchestrator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `flai-vgv-migration` from a 700-line monolithic guide into a thin orchestrator that delegates to existing flai skills, installs packages via `dart pub add` (no version pinning, always latest), and produces a battery-included starter project with a working `posts` feature, full logger wiring, theme, l10n, and tests — passing `dart analyze` with `very_good_analysis` out of the box.

**Architecture:** Three-part change. (1) Fix bugs and lint compliance in subordinate skills so delegated code is correct from the start. (2) Add a complete `main.dart` wiring template to `flai-logging` so vgv-migration has one place to delegate app bootstrap. (3) Rewrite `skills/vgv-migration/SKILL.md` as an orchestrator that invokes other flai skills phase-by-phase, scaffolds a `posts` feature consuming `https://jsonplaceholder.typicode.com`, wires Material 3 theme, l10n, accessibility, and verifies with `dart analyze` + `flutter test` before considering migration complete.

**Tech Stack:** Markdown skill files, YAML frontmatter, Claude Code plugin system. Documentation-only repository — no Dart/Flutter code in this plugin. The Dart code shown in tasks is templates that the AI writes into target Flutter projects when invoking the skill.

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `skills/networking/SKILL.md` | Modify | Fix `connectivityProvider.notifier` bug; apply `cascade_invocations` |
| `skills/connectivity/SKILL.md` | Modify | Add `connectivityInstanceProvider` for direct `Connectivity` injection |
| `skills/material-theming/SKILL.md` | Modify | Add explicit `Color` types to `AppColors` (very_good_analysis `always_specify_types`) |
| `skills/logging/SKILL.md` | Modify | Replace partial main.dart hint with complete wiring template (Riverpod observer + Route observer + TalkerScreen route) |
| `skills/vgv-migration/SKILL.md` | Replace | Rewrite as orchestrator; from ~700 lines to ~250 lines |
| `README.md` | Modify | Update Quick Start prompts; mention battery-included starter |
| `.claude-plugin/plugin.json` | Modify | Bump version to 0.2.0 |

---

### Task 1: Fix `connectivityProvider.notifier` bug in `flai-networking`

**Background:** `connectivityProvider` is defined in `flai-connectivity` as a function-based `StreamProvider<List<ConnectivityResult>>`. Function-based providers do **not** expose `.notifier`. The current `dio_provider` example references `connectivityProvider.notifier` and would fail to compile.

**Files:**
- Modify: `skills/networking/SKILL.md` (Connectivity-Aware Interceptor section, ~line 130)

- [ ] **Step 1: Replace the `ConnectivityInterceptor` provider definition**

In `skills/networking/SKILL.md`, locate the `Connectivity-Aware Interceptor` section (around line 128-167). Replace the entire code block with:

````markdown
```dart
// core/http/connectivity_interceptor.dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:dio/dio.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../providers/connectivity_provider.dart';

part 'connectivity_interceptor.g.dart';

@Riverpod(keepAlive: true)
ConnectivityInterceptor connectivityInterceptor(
  ConnectivityInterceptorRef ref,
) {
  return ConnectivityInterceptor(ref.watch(connectivityInstanceProvider));
}

class ConnectivityInterceptor extends Interceptor {
  ConnectivityInterceptor(this._connectivity);

  final Connectivity _connectivity;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final result = await _connectivity.checkConnectivity();
    if (result.contains(ConnectivityResult.none)) {
      handler.reject(
        DioException(
          requestOptions: options,
          type: DioExceptionType.unknown,
          error: 'No internet connection',
        ),
      );
      return;
    }
    handler.next(options);
  }
}
```
````

The change: `ref.watch(connectivityProvider.notifier)` → `ref.watch(connectivityInstanceProvider)`. The `connectivityInstanceProvider` is added in Task 2.

- [ ] **Step 2: Verify the change reads correctly**

Run:
```bash
grep -n "connectivityInstanceProvider" /Users/silmi/workspace/devtools/flai-plugin/skills/networking/SKILL.md
```

Expected: 1 match in the `connectivity_interceptor.dart` block.

- [ ] **Step 3: Commit**

```bash
git add skills/networking/SKILL.md
git commit -m "$(cat <<'EOF'
fix(networking): replace invalid connectivityProvider.notifier with connectivityInstanceProvider

Function-based StreamProviders do not expose `.notifier`. Inject
Connectivity directly via a new `connectivityInstanceProvider` (added in
flai-connectivity skill).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Add `connectivityInstanceProvider` to `flai-connectivity`

**Files:**
- Modify: `skills/connectivity/SKILL.md` (Connectivity Provider section)

- [ ] **Step 1: Replace the `Connectivity Provider` section**

Locate the section starting `## Connectivity Provider` (around line 22). Replace the entire code block with:

````markdown
```dart
// core/providers/connectivity_provider.dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'connectivity_provider.g.dart';

/// Singleton `Connectivity` instance for direct calls (e.g. interceptors,
/// one-off checks). Use this when you need to call `checkConnectivity()`
/// imperatively rather than subscribe to changes.
@Riverpod(keepAlive: true)
Connectivity connectivityInstance(ConnectivityInstanceRef ref) =>
    Connectivity();

/// Reactive stream of connectivity changes. Subscribe in widgets that
/// need to rebuild when the network status changes.
@Riverpod(keepAlive: true)
Stream<List<ConnectivityResult>> connectivity(ConnectivityRef ref) =>
    ref.watch(connectivityInstanceProvider).onConnectivityChanged;

bool isOnline(List<ConnectivityResult> result) =>
    result.isNotEmpty && !result.contains(ConnectivityResult.none);
```
````

- [ ] **Step 2: Verify the additions**

Run:
```bash
grep -n "connectivityInstanceProvider\|connectivityInstance " /Users/silmi/workspace/devtools/flai-plugin/skills/connectivity/SKILL.md
```

Expected: at least 2 matches (definition + reference inside the stream provider).

- [ ] **Step 3: Commit**

```bash
git add skills/connectivity/SKILL.md
git commit -m "$(cat <<'EOF'
feat(connectivity): add connectivityInstanceProvider for direct Connectivity injection

Allows interceptors and one-off connectivity checks to inject the
Connectivity instance via Riverpod without subscribing to the stream.
The stream provider now derives its source from this instance provider
to keep a single instance.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Apply `cascade_invocations` in `flai-networking` dio_provider

**Background:** `very_good_analysis` enforces `cascade_invocations` — when calling multiple methods on the same target, prefer `..` cascade. The current `dio_provider` example assigns `dio` then mutates `dio.interceptors` — should be cascaded.

**Files:**
- Modify: `skills/networking/SKILL.md` (Dio Setup section, ~line 26)

- [ ] **Step 1: Replace the Dio Setup code block**

Locate the section starting `## Dio Setup` (around line 25). Replace the entire code block with:

````markdown
```dart
// core/http/dio_provider.dart
import 'package:dio/dio.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:talker_dio_logger/talker_dio_logger.dart';

import '../config/env.dart';
import '../logging/talker_provider.dart';
import 'connectivity_interceptor.dart';

part 'dio_provider.g.dart';

@Riverpod(keepAlive: true)
Dio dio(DioRef ref) {
  final talker = ref.watch(talkerProvider);

  return Dio(
    BaseOptions(
      baseUrl: Env.apiBaseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
    ),
  )..interceptors.addAll([
      TalkerDioLogger(
        talker: talker,
        settings: const TalkerDioLoggerSettings(
          printRequestHeaders: false,
          printResponseMessage: true,
        ),
      ),
      ref.watch(connectivityInterceptorProvider),
    ]);
}
```
````

The change: extracted `talker` first, used cascade `..interceptors.addAll(...)` directly on `Dio(...)`, and added imports that the example previously omitted.

- [ ] **Step 2: Verify**

Run:
```bash
grep -n "..interceptors.addAll" /Users/silmi/workspace/devtools/flai-plugin/skills/networking/SKILL.md
```

Expected: 1 match.

- [ ] **Step 3: Commit**

```bash
git add skills/networking/SKILL.md
git commit -m "$(cat <<'EOF'
fix(networking): use cascade for dio.interceptors and add missing imports

very_good_analysis enforces cascade_invocations. Use `..interceptors.addAll`
on the Dio instance directly. Also add the imports the example needs
(env.dart, talker_provider.dart, connectivity_interceptor.dart).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Add explicit `Color` type to `AppColors` in `flai-material-theming`

**Background:** `very_good_analysis` enables `always_specify_types`, which requires explicit type annotations on top-level and static fields. The current `AppColors` example uses inferred types (`static const primaryColor = Color(0xFF...)`).

**Files:**
- Modify: `skills/material-theming/SKILL.md` (Custom Colors Class section, ~line 30)

- [ ] **Step 1: Replace the `AppColors` code block**

Locate the section starting `### Custom Colors Class` (around line 28). Replace the code block with:

````markdown
```dart
abstract class AppColors {
  static const Color primaryColor = Color(0xFF4F46E5);
  static const Color secondaryColor = Color(0xFF9C27B0);
  static const Color errorColor = Color(0xFFDC2626);
  static const Color surfaceColor = Color(0xFFFAFAFA);

  // Dark theme variants
  static const Color primaryColorDark = Color(0xFF6366F1);
  static const Color surfaceColorDark = Color(0xFF1F1F1F);
}
```
````

The change: explicit `Color` type on each field, plus dark variants that the dark theme example references but were not previously declared.

- [ ] **Step 2: Verify**

Run:
```bash
grep -n "static const Color" /Users/silmi/workspace/devtools/flai-plugin/skills/material-theming/SKILL.md
```

Expected: at least 6 matches.

- [ ] **Step 3: Commit**

```bash
git add skills/material-theming/SKILL.md
git commit -m "$(cat <<'EOF'
fix(material-theming): add explicit Color types to AppColors fields

very_good_analysis enables always_specify_types. Top-level and static
fields require explicit type annotations. Also add the dark variants
that the dark theme example references.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Add complete `main.dart` template to `flai-logging`

**Background:** Audit revealed that `flai-logging` mentions Riverpod observer and Route observer in Core Standards but has no single complete `main.dart` template. The current "Wiring in main.dart" section (line 44-71) is partial. vgv-migration needs one canonical place to delegate app bootstrap, including TalkerScreen route for debug menu.

**Files:**
- Modify: `skills/logging/SKILL.md` (Wiring in main.dart section)

- [ ] **Step 1: Replace the Wiring in main.dart section**

Locate `## Wiring in main.dart` (around line 43). Replace **everything from that heading up to but not including `## Dio Integration`** with:

````markdown
## Wiring in main.dart

The app bootstrap is the single source of truth for logger wiring. Every flai app's `main.dart` follows this exact shape — it initializes Talker once, attaches the Riverpod observer to the `ProviderScope`, and overrides `talkerProvider` so every other layer (Dio, GoRouter, Notifiers) receives the same instance.

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:talker_flutter/talker_flutter.dart';
import 'package:talker_riverpod_logger/talker_riverpod_logger.dart';

import 'app/app.dart';
import 'core/logging/talker_provider.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final talker = TalkerFlutter.init();

  runApp(
    ProviderScope(
      observers: [
        TalkerRiverpodObserver(
          talker: talker,
          settings: const TalkerRiverpodLoggerSettings(
            printProviderAdded: true,
            printProviderUpdated: true,
            printProviderDisposed: false,
            printProviderFailed: true,
          ),
        ),
      ],
      overrides: [
        talkerProvider.overrideWithValue(talker),
      ],
      child: const App(),
    ),
  );
}
```

Why each piece matters:

| Wiring | Effect if missing |
| --- | --- |
| `TalkerFlutter.init()` before `runApp` | App boots, but no Talker instance exists for the first frame's logs |
| `TalkerRiverpodObserver` in `observers:` | Provider lifecycle events (added, updated, failed) silently disappear |
| `talkerProvider.overrideWithValue(talker)` | Every consumer of `talkerProvider` would otherwise create its own Talker, losing log unification |
| `TalkerRouteObserver` on GoRouter (see Router section) | Navigation events disappear — hard to debug deep links |
| `TalkerDioLogger` on Dio (see Dio section) | HTTP requests/responses disappear from logs |

## Debug Menu — TalkerScreen Route

Expose the in-app log viewer in debug builds via a dedicated route:

```dart
// core/router/router.dart (excerpt — full router lives in flai-navigation)
import 'package:flutter/foundation.dart';
import 'package:talker_flutter/talker_flutter.dart';

@TypedGoRoute<TalkerScreenRoute>(
  name: 'talkerLogs',
  path: '/debug/logs',
)
@immutable
class TalkerScreenRoute extends GoRouteData {
  const TalkerScreenRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) {
    final talker = TalkerFlutter.init();
    return TalkerScreen(talker: talker);
  }
}
```

Gate this route behind `kDebugMode` if you want to hide it in release builds. Surface a navigation entry from a developer settings page or a long-press gesture on the main app bar.

````

The change: complete `main.dart` template with all observers wired, "why each piece matters" table, and a TalkerScreen route example using TypedGoRoute (consistent with `flai-navigation`).

- [ ] **Step 2: Verify the section is in place**

Run:
```bash
grep -c "TalkerRiverpodObserver\|TalkerScreen\|TalkerRouteObserver" /Users/silmi/workspace/devtools/flai-plugin/skills/logging/SKILL.md
```

Expected: at least 4 matches across the file.

- [ ] **Step 3: Commit**

```bash
git add skills/logging/SKILL.md
git commit -m "$(cat <<'EOF'
feat(logging): add complete main.dart wiring template and TalkerScreen route

Centralizes app bootstrap as the single source of truth so vgv-migration
can delegate without duplication. Adds:

- Full main.dart with TalkerRiverpodObserver in ProviderScope
- talkerProvider.overrideWithValue for instance unification
- "why each piece matters" reference table
- TalkerScreen route example for in-app log viewer

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Replace `skills/vgv-migration/SKILL.md` with orchestrator version

**Background:** This is the central rewrite. The new SKILL.md is an orchestrator: each phase tells the AI to invoke a specific subordinate skill, run a specific command, or scaffold a small specific file. No more inline reproduction of code that already lives in other skills. Sample feature is changed from `counter` to `posts` so it exercises networking, serialization, error handling, and async state.

**Files:**
- Replace: `skills/vgv-migration/SKILL.md`

- [ ] **Step 1: Overwrite the file with new content**

Replace the entire contents of `skills/vgv-migration/SKILL.md` with:

````markdown
---
name: flai-vgv-migration
description: Migrate a VeryGood CLI Flutter project to flai standards. Use when the project was scaffolded with `very_good create` and needs refactoring into the flai Clean Architecture + Riverpod stack with a battery-included `posts` starter feature, full logger wiring, Material 3 theme, l10n, and tests passing very_good_analysis.
allowed-tools: Read Glob Grep Bash Edit Write
model: sonnet
effort: high
---

# VGV → flai Migration

Step-by-step orchestrator that migrates a `very_good create` scaffold to flai standards. Each phase **delegates to a single-purpose flai skill** for the canonical pattern — this skill does **not** redefine code that lives elsewhere. The result is a battery-included starter: it compiles, passes `dart analyze`, has a working `posts` feature consuming `https://jsonplaceholder.typicode.com`, and is ready to extend.

## Core Standards

Apply these standards throughout the migration:

- **Delegate, don't duplicate** — when scaffolding `failures.dart`, `dio_provider.dart`, etc., **invoke the relevant flai skill** for the canonical template; do not re-derive patterns
- **Install packages with `dart pub add`** — never edit `pubspec.yaml` by hand; pub solver picks the latest compatible version automatically
- **Verify per phase, not at the end** — after each scaffold phase, run `dart analyze` and fix violations before moving on; fail-fast prevents end-of-migration surprises
- **Preserve VGV infrastructure** — keep `analysis_options.yaml` (`very_good_analysis`), `.github/workflows/`, `test/helpers/`, `l10n.yaml`, `lib/l10n/`, `lefthook.yml`, `.gitignore`
- **Replace ALL Bloc/Cubit usage** — Riverpod with `@riverpod` only
- **Lulus `dart analyze` end-to-end** — output must pass `very_good_analysis` without `// ignore` comments except where explicitly allowed (`invalid_annotation_target` for Freezed)
- **Generate CLAUDE.md as the final step** — the enforcement contract for the migrated project

## Pre-Flight Check

Verify the project was created with `very_good create` before proceeding:

```bash
test -f lib/app/view/app.dart && \
test -d lib/counter && \
grep -q "flutter_bloc" pubspec.yaml && \
grep -q "very_good_analysis" pubspec.yaml && \
echo "VGV project confirmed"
```

All checks must pass. If any fails, this is not a VGV scaffold — invoke `flai-architecture` for greenfield scaffolding instead.

## Phase 1 — Cleanup

Remove VGV defaults that conflict with flai structure.

```bash
# Remove Bloc packages
dart pub remove bloc flutter_bloc bloc_test

# Remove VGV scaffold directories
rm -rf lib/app lib/counter
rm -rf test/app test/counter

# Keep test/helpers/, analysis_options.yaml, l10n.yaml, lib/l10n/, .github/workflows/
```

`mocktail` is shared between VGV and flai — leave it installed.

## Phase 2 — Install flai Packages

Add packages with `dart pub add`. No version constraints — pub solver picks the latest compatible version.

```bash
# State management
dart pub add flutter_riverpod riverpod_annotation

# Networking
dart pub add dio retrofit connectivity_plus

# Models & errors
dart pub add freezed_annotation json_annotation fpdart

# Storage
dart pub add shared_preferences flutter_secure_storage isar path_provider

# Navigation
dart pub add go_router

# Logging
dart pub add talker talker_dio_logger talker_riverpod_logger talker_flutter

# Environment config
dart pub add envied

# Code generators (dev)
dart pub add build_runner riverpod_generator riverpod_lint custom_lint \
  freezed json_serializable retrofit_generator envied_generator \
  isar_generator go_router_builder --dev

# Pin analyzer to resolve freezed + go_router_builder conflict
dart pub add analyzer:^6.7.0 --dev

flutter pub get
```

After install, verify no `pub get` errors:

```bash
flutter pub deps --json > /dev/null && echo "deps OK"
```

## Phase 3 — Scaffold `lib/core/`

Create the core directory tree, then delegate each file's content to its source-of-truth skill.

```bash
mkdir -p lib/core/config lib/core/database lib/core/error \
  lib/core/extensions lib/core/http lib/core/logging \
  lib/core/providers lib/core/router lib/core/theme \
  lib/core/widgets
```

For each file below, **read the listed flai skill** and copy its canonical template into the target file. **Adapt only environment-specific values** (e.g., default `API_BASE_URL`).

| Target file | Source skill | Notes |
| --- | --- | --- |
| `lib/core/error/failures.dart` | `flai-error-handling` (Base Failure Class) | Use the canonical sealed `Failure` (network/unauthorized/unknown) — do **not** invent extra variants |
| `lib/core/config/env.dart` | `flai-environment-config` (Env Class) | Default `API_BASE_URL=https://jsonplaceholder.typicode.com`, no `apiKey` (jsonplaceholder needs none) |
| `lib/core/logging/talker_provider.dart` | `flai-logging` (Talker Provider) | Path is `core/logging/`, not `core/providers/` |
| `lib/core/providers/connectivity_provider.dart` | `flai-connectivity` | Includes both `connectivityInstanceProvider` and `connectivityProvider` |
| `lib/core/http/connectivity_interceptor.dart` | `flai-networking` (Connectivity-Aware Interceptor) | |
| `lib/core/http/dio_provider.dart` | `flai-networking` (Dio Setup) | |
| `lib/core/router/router.dart` | `flai-navigation` (Type-Safe Routes + Logging Integration) | TypedGoRoute, with `TalkerRouteObserver` |
| `lib/core/theme/app_colors.dart` | `flai-material-theming` (Custom Colors Class) | |
| `lib/core/theme/app_theme.dart` | `flai-material-theming` (Light and Dark Theme Variants) | Both light and dark `ThemeData` |
| `.env` and `.env.example` | `flai-environment-config` | Set `API_BASE_URL=https://jsonplaceholder.typicode.com` in both |

Also create the localization l10n delegate extension:

```dart
// lib/core/extensions/l10n_extension.dart
import 'package:flutter/widgets.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

extension AppLocalizationsX on BuildContext {
  AppLocalizations get l10n => AppLocalizations.of(this)!;
}
```

**Verify**:

```bash
dart format lib/core
dart analyze lib/core
```

Fix any violations before proceeding. The `.g.dart` and `.freezed.dart` files do not exist yet — analyze will warn about missing parts; that is expected and resolved in Phase 7.

## Phase 4 — Wire `lib/main.dart` and `lib/app/app.dart`

Invoke `flai-logging` (Wiring in main.dart section) for the canonical `main.dart`. Adapt the import path of `app/app.dart` to match the project name.

`lib/app/app.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../core/router/router.dart';
import '../core/theme/app_theme.dart';

class App extends ConsumerWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: 'flai Starter',
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      routerConfig: router,
    );
  }
}
```

**Verify**: `dart format lib && dart analyze lib`

## Phase 5 — Scaffold the `posts` Feature (Battery-Included Sample)

`posts` exercises every flai layer end-to-end: Retrofit API → repository with `Either<Failure, T>` → `AsyncNotifier` → `ConsumerWidget` page with loading/error/data states. Source: `https://jsonplaceholder.typicode.com/posts`.

Create the directory tree:

```bash
mkdir -p lib/features/posts/data/datasources \
  lib/features/posts/data/models \
  lib/features/posts/data/repositories \
  lib/features/posts/domain/entities \
  lib/features/posts/domain/failures \
  lib/features/posts/domain/repositories \
  lib/features/posts/presentation/notifiers \
  lib/features/posts/presentation/pages \
  lib/features/posts/presentation/widgets
```

### 5a. Domain layer

`lib/features/posts/domain/entities/post.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'post.freezed.dart';

@freezed
class Post with _$Post {
  const factory Post({
    required int id,
    required int userId,
    required String title,
    required String body,
  }) = _Post;
}
```

`lib/features/posts/domain/failures/post_failure.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'post_failure.freezed.dart';

@freezed
sealed class PostFailure with _$PostFailure {
  const factory PostFailure.notFound() = PostNotFound;
  const factory PostFailure.timeout() = PostTimeout;
  const factory PostFailure.network() = PostNetworkFailure;
  const factory PostFailure.unknown() = PostUnknown;
}
```

`lib/features/posts/domain/repositories/post_repository.dart`:

```dart
import 'package:fpdart/fpdart.dart';

import '../entities/post.dart';
import '../failures/post_failure.dart';

abstract interface class PostRepository {
  Future<Either<PostFailure, List<Post>>> getPosts();
  Future<Either<PostFailure, Post>> getPostById(int id);
}
```

### 5b. Data layer

`lib/features/posts/data/models/post_model.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

import '../../domain/entities/post.dart';

part 'post_model.freezed.dart';
part 'post_model.g.dart';

@freezed
class PostModel with _$PostModel {
  const factory PostModel({
    required int id,
    required int userId,
    required String title,
    required String body,
  }) = _PostModel;

  factory PostModel.fromJson(Map<String, dynamic> json) =>
      _$PostModelFromJson(json);
}

extension PostModelX on PostModel {
  Post toDomain() =>
      Post(id: id, userId: userId, title: title, body: body);
}
```

`lib/features/posts/data/datasources/posts_api.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../../core/http/dio_provider.dart';
import '../models/post_model.dart';

part 'posts_api.g.dart';

@RestApi()
abstract class PostsApi {
  factory PostsApi(Dio dio, {String? baseUrl}) = _PostsApi;

  @GET('/posts')
  Future<List<PostModel>> getPosts();

  @GET('/posts/{id}')
  Future<PostModel> getPostById(@Path('id') int id);
}

@riverpod
PostsApi postsApi(PostsApiRef ref) => PostsApi(ref.watch(dioProvider));
```

`lib/features/posts/data/repositories/post_repository_impl.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:fpdart/fpdart.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../domain/entities/post.dart';
import '../../domain/failures/post_failure.dart';
import '../../domain/repositories/post_repository.dart';
import '../datasources/posts_api.dart';

part 'post_repository_impl.g.dart';

class PostRepositoryImpl implements PostRepository {
  const PostRepositoryImpl(this._api);

  final PostsApi _api;

  @override
  Future<Either<PostFailure, List<Post>>> getPosts() async {
    try {
      final models = await _api.getPosts();
      return right(models.map((m) => m.toDomain()).toList());
    } on DioException catch (e) {
      return left(_mapDioException(e));
    } catch (_) {
      return left(const PostFailure.unknown());
    }
  }

  @override
  Future<Either<PostFailure, Post>> getPostById(int id) async {
    try {
      final model = await _api.getPostById(id);
      return right(model.toDomain());
    } on DioException catch (e) {
      if (e.response?.statusCode == 404) {
        return left(const PostFailure.notFound());
      }
      return left(_mapDioException(e));
    } catch (_) {
      return left(const PostFailure.unknown());
    }
  }

  PostFailure _mapDioException(DioException e) {
    return switch (e.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.receiveTimeout =>
        const PostFailure.timeout(),
      DioExceptionType.connectionError => const PostFailure.network(),
      _ => const PostFailure.unknown(),
    };
  }
}

@riverpod
PostRepository postRepository(PostRepositoryRef ref) =>
    PostRepositoryImpl(ref.watch(postsApiProvider));
```

### 5c. Presentation layer

`lib/features/posts/presentation/notifiers/posts_notifier.dart`:

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../data/repositories/post_repository_impl.dart';
import '../../domain/entities/post.dart';

part 'posts_notifier.g.dart';

@riverpod
class PostsNotifier extends _$PostsNotifier {
  @override
  Future<List<Post>> build() async {
    final repo = ref.watch(postRepositoryProvider);
    final result = await repo.getPosts();
    return result.fold((failure) => throw failure, (posts) => posts);
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    final repo = ref.read(postRepositoryProvider);
    state = await AsyncValue.guard(
      () => repo.getPosts().then(
            (result) => result.fold((f) => throw f, (p) => p),
          ),
    );
  }
}
```

`lib/features/posts/presentation/pages/posts_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../../../../core/extensions/l10n_extension.dart';
import '../../../../core/router/router.dart';
import '../../domain/failures/post_failure.dart';
import '../notifiers/posts_notifier.dart';
import '../widgets/post_list_tile.dart';

class PostsPage extends ConsumerWidget {
  const PostsPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final postsAsync = ref.watch(postsNotifierProvider);

    return Scaffold(
      appBar: AppBar(title: Text(context.l10n.postsTitle)),
      body: postsAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (err, _) => _PostsError(
          failure: err is PostFailure ? err : const PostFailure.unknown(),
          onRetry: () => ref.read(postsNotifierProvider.notifier).refresh(),
        ),
        data: (posts) {
          if (posts.isEmpty) {
            return Center(child: Text(context.l10n.postsEmpty));
          }
          return RefreshIndicator(
            onRefresh: ref.read(postsNotifierProvider.notifier).refresh,
            child: ListView.separated(
              itemCount: posts.length,
              separatorBuilder: (_, __) => const Divider(height: 1),
              itemBuilder: (_, i) => PostListTile(
                post: posts[i],
                onTap: () =>
                    PostDetailPageRoute(id: posts[i].id).go(context),
              ),
            ),
          );
        },
      ),
    );
  }
}

class _PostsError extends ConsumerWidget {
  const _PostsError({required this.failure, required this.onRetry});

  final PostFailure failure;
  final VoidCallback onRetry;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final message = switch (failure) {
      PostNotFound() => context.l10n.postsNotFound,
      PostTimeout() => context.l10n.postsTimeout,
      PostNetworkFailure() => context.l10n.postsOffline,
      PostUnknown() => context.l10n.postsUnknownError,
    };

    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              Icons.error_outline,
              size: 48,
              color: Theme.of(context).colorScheme.error,
              semanticLabel: context.l10n.postsErrorIcon,
            ),
            const SizedBox(height: 16),
            Text(message, textAlign: TextAlign.center),
            const SizedBox(height: 16),
            FilledButton(
              onPressed: onRetry,
              child: Text(context.l10n.retry),
            ),
          ],
        ),
      ),
    );
  }
}
```

`lib/features/posts/presentation/pages/post_detail_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../../core/extensions/l10n_extension.dart';
import '../../data/repositories/post_repository_impl.dart';
import '../../domain/entities/post.dart';

part 'post_detail_page.g.dart';

@riverpod
Future<Post> postDetail(PostDetailRef ref, int id) async {
  final repo = ref.watch(postRepositoryProvider);
  final result = await repo.getPostById(id);
  return result.fold((f) => throw f, (p) => p);
}

class PostDetailPage extends ConsumerWidget {
  const PostDetailPage({required this.id, super.key});

  final int id;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final detail = ref.watch(postDetailProvider(id));

    return Scaffold(
      appBar: AppBar(title: Text(context.l10n.postDetailTitle)),
      body: detail.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (err, _) => Center(child: Text('$err')),
        data: (post) => Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                post.title,
                style: Theme.of(context).textTheme.headlineSmall,
              ),
              const SizedBox(height: 12),
              Text(post.body),
            ],
          ),
        ),
      ),
    );
  }
}
```

`lib/features/posts/presentation/widgets/post_list_tile.dart`:

```dart
import 'package:flutter/material.dart';

import '../../domain/entities/post.dart';

class PostListTile extends StatelessWidget {
  const PostListTile({required this.post, required this.onTap, super.key});

  final Post post;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(post.title, maxLines: 1, overflow: TextOverflow.ellipsis),
      subtitle: Text(post.body, maxLines: 2, overflow: TextOverflow.ellipsis),
      onTap: onTap,
    );
  }
}
```

`lib/features/posts/providers.dart`:

```dart
export 'data/datasources/posts_api.dart';
export 'data/models/post_model.dart';
export 'data/repositories/post_repository_impl.dart';
export 'domain/entities/post.dart';
export 'domain/failures/post_failure.dart';
export 'domain/repositories/post_repository.dart';
export 'presentation/notifiers/posts_notifier.dart';
export 'presentation/pages/post_detail_page.dart';
export 'presentation/pages/posts_page.dart';
```

### 5d. Add routes

In `lib/core/router/router.dart`, add the posts routes following the `flai-navigation` TypedGoRoute pattern:

```dart
@TypedGoRoute<PostsPageRoute>(
  name: 'posts',
  path: '/posts',
  routes: [
    TypedGoRoute<PostDetailPageRoute>(
      name: 'postDetail',
      path: ':id',
    ),
  ],
)
@immutable
class PostsPageRoute extends GoRouteData {
  const PostsPageRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) =>
      const PostsPage();
}

@immutable
class PostDetailPageRoute extends GoRouteData {
  const PostDetailPageRoute({required this.id});

  final int id;

  @override
  Widget build(BuildContext context, GoRouterState state) =>
      PostDetailPage(id: id);
}
```

Set `initialLocation: '/posts'` on the `GoRouter` so the app boots into the posts page.

## Phase 6 — Configure `build.yaml`

Invoke `flai-code-generation` for the canonical `build.yaml`. Adapt the `riverpod_generator` paths to include `lib/features/posts/**`.

## Phase 7 — l10n: ARB and Generated Strings

Append the strings the `posts` page references to `lib/l10n/arb/app_en.arb`:

```json
{
  "postsTitle": "Posts",
  "@postsTitle": { "description": "App bar title for the posts list" },

  "postsEmpty": "No posts yet",
  "@postsEmpty": { "description": "Empty state for posts list" },

  "postsNotFound": "We couldn't find that post.",
  "postsTimeout": "Request timed out. Please try again.",
  "postsOffline": "You are offline. Connect to the internet and retry.",
  "postsUnknownError": "Something went wrong.",
  "postsErrorIcon": "Error",
  "postDetailTitle": "Post detail",
  "retry": "Retry"
}
```

Then regenerate:

```bash
flutter gen-l10n
```

## Phase 8 — Run Code Generation

```bash
dart run build_runner build --delete-conflicting-outputs
```

Verify all `.g.dart` and `.freezed.dart` files were generated under `lib/`. Fix any reported errors before proceeding.

## Phase 9 — Adapt Test Helpers

Invoke `flai-testing` (pumpApp Helper section) for the canonical `test/helpers/pump_app.dart` template. Replace any VGV `pumpApp` helper that imports Bloc.

Add a starter test for the posts feature at `test/features/posts/presentation/pages/posts_page_test.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:fpdart/fpdart.dart';
import 'package:my_app/features/posts/data/repositories/post_repository_impl.dart';
import 'package:my_app/features/posts/domain/entities/post.dart';
import 'package:my_app/features/posts/domain/repositories/post_repository.dart';
import 'package:my_app/features/posts/presentation/pages/posts_page.dart';
import 'package:mocktail/mocktail.dart';

import '../../../../helpers/pump_app.dart';

class _MockPostRepository extends Mock implements PostRepository {}

void main() {
  group(PostsPage, () {
    late PostRepository repository;

    setUp(() {
      repository = _MockPostRepository();
    });

    testWidgets('renders posts when repository returns data',
        (tester) async {
      when(() => repository.getPosts()).thenAnswer(
        (_) async => right([
          const Post(id: 1, userId: 1, title: 'Hello', body: 'World'),
        ]),
      );

      await tester.pumpApp(
        const PostsPage(),
        overrides: [
          postRepositoryProvider.overrideWithValue(repository),
        ],
      );
      await tester.pump();

      expect(find.text('Hello'), findsOneWidget);
    });
  });
}
```

Replace `my_app` with the actual project package name (read from `pubspec.yaml` `name:` field).

## Phase 10 — Per-Phase Verification

Run after every scaffold phase, fail-fast:

```bash
dart format --set-exit-if-changed lib test
dart analyze
flutter test
```

If `dart analyze` reports `very_good_analysis` violations, fix them at the source skill (so future migrations are correct), then update the local file. Common violations to watch for: `cascade_invocations`, `prefer_const_constructors`, `directives_ordering`, `always_specify_types`, `unawaited_futures`.

## Phase 11 — Generate `CLAUDE.md`

Create `CLAUDE.md` at the project root:

````markdown
# CLAUDE.md

## Project Standard

This project follows the **flai Flutter standard** — Clean Architecture with Riverpod, fpdart, Material 3, and the flai tech stack. It was scaffolded by migrating from a `very_good create` baseline using `flai-vgv-migration`.

## Required Plugin

Install the flai-plugin before working on this project:

```bash
/plugin marketplace add SimiPrambos/flai-plugin
/plugin install flai-plugin@flai-marketplace
```

**If the plugin is not installed, STOP. Do not generate code without it.**

## Mandatory Skill Invocation

Before writing ANY code, invoke the relevant skill:

| Task | Required Skill |
|------|----------------|
| Creating features, organizing folders | `flai-architecture` |
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

**When in doubt:** Invoke `flai-architecture` first.

## Hard Rules

These rules are NON-NEGOTIABLE.

### Architecture
- ALWAYS use `lib/core/` for global concerns
- ALWAYS use `lib/features/<name>/` with `data/`, `domain/`, `presentation/` subdirectories
- NEVER create cross-feature imports
- NEVER put Flutter imports in `domain/` layer
- NEVER create `utils/` or `common/` catch-all folders

### State Management
- ALWAYS use Riverpod with `@riverpod` code generation
- ALWAYS use `AsyncNotifier` or `Notifier`
- NEVER use Bloc, Cubit, Provider, or `setState`

### Error Handling
- ALWAYS return `Either<Failure, T>` from repositories
- ALWAYS use sealed `Failure` subclasses
- NEVER throw exceptions from repositories

### Packages
- Add packages with `dart pub add <name>` — never edit `pubspec.yaml` by hand
- NEVER add packages that conflict with the flai stack (GetX, Provider, http instead of Dio)

### Code Quality
- ALWAYS run `dart analyze` before considering work complete (must pass `very_good_analysis`)
- ALWAYS run `dart format` on modified files
- ALWAYS write tests for new features

## What's Already Wired

This project ships with:

- ✅ `posts` feature consuming `https://jsonplaceholder.typicode.com` — runnable demo of every layer
- ✅ Talker logger with Riverpod observer, Route observer, Dio interceptor
- ✅ TalkerScreen at `/debug/logs` for in-app log viewing
- ✅ Material 3 theme (light + dark)
- ✅ l10n via ARB files (`lib/l10n/arb/app_en.arb`)
- ✅ TypedGoRoute navigation
- ✅ Connectivity-aware Dio interceptor
- ✅ `pumpApp` test helper

Use `posts` as the reference implementation when adding new features — its shape is the contract.

## Verification

Before claiming any task complete:
1. Invoke the relevant skill(s)
2. Generate/modify code following skill patterns
3. Run `dart analyze` — must pass with no violations
4. Run `dart format` — must pass with no diff
5. Run tests — must pass

## Questions?

If unsure which skill applies or how to implement something within flai standards, ASK before generating code. Do not guess or fall back to generic Flutter patterns.
````

## Phase 12 — Final Validation

Run all checks. All must pass.

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
dart format --set-exit-if-changed lib test
dart analyze
flutter test
```

Then **manually** run the app and verify the smoke test:

```bash
flutter run
```

Expected behavior:
1. App opens on `/posts` route
2. Posts list loads from `jsonplaceholder.typicode.com` (≈100 items)
3. Tapping a post opens detail page
4. Tapping retry on a simulated error reloads the list
5. Talker logs visible at `/debug/logs`

If any step fails, fix at the relevant skill source-of-truth and re-run the migration.

## What to Preserve from VGV

Do NOT remove or modify these VGV artifacts:

- `.github/workflows/` — CI/CD pipelines (verify `build_runner build` step is present; add if missing)
- `test/helpers/` — adapt for Riverpod, keep directory
- `analysis_options.yaml` — `very_good_analysis` rules
- `l10n.yaml` — localization config
- `lib/l10n/` — ARB files (append new keys, don't replace)
- `.gitignore` — VGV's comprehensive ignore file
- `lefthook.yml` — pre-commit hooks
- `pubspec.yaml` `flutter_lints` and SDK constraints — keep if present

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| Editing `pubspec.yaml` by hand to add packages | Picks no version constraint or wrong constraint; can leave conflicting transitives | `dart pub add <name>` lets pub solver pick latest compatible |
| Skipping per-phase `dart analyze` | Lint violations pile up; fixing 50 at the end is harder than 5 per phase | Run after every phase; fail-fast |
| Re-defining `failures.dart`/`dio_provider.dart` instead of invoking source skill | Drift; the migration's version goes stale when the source skill updates | Always read the source skill and adapt only project-specific values |
| Using counter as the sample feature | Counter doesn't exercise networking, serialization, or async errors — leaves blind spots in the starter | Use `posts` (jsonplaceholder) — exercises every layer |
| Keeping Bloc references | Mixed Bloc/Riverpod confuses developers and the AI | Replace ALL Bloc/Cubit usage |
| Removing CI/CD workflows | Loses automated testing; CI was VGV's main value | Preserve `.github/workflows/`; verify `build_runner` step |
| Ignoring `dart analyze` warnings | very_good_analysis violations compound; the starter should be clean from day one | Fix at source skill, then locally |
````

- [ ] **Step 2: Verify line count is reasonable**

Run:
```bash
wc -l /Users/silmi/workspace/devtools/flai-plugin/skills/vgv-migration/SKILL.md
```

Expected: ~600 lines (still substantial because of the inline `posts` feature templates that are project-specific and don't live in any other skill, but ~100 lines less than the original 701 and far less duplication of skill content).

- [ ] **Step 3: Verify cross-references resolve**

Run:
```bash
grep -nE "flai-(error-handling|environment-config|logging|networking|navigation|material-theming|connectivity|architecture|riverpod|serialization|testing|code-generation|internationalization|accessibility)" /Users/silmi/workspace/devtools/flai-plugin/skills/vgv-migration/SKILL.md | wc -l
```

Expected: at least 12 references (one per delegated skill).

- [ ] **Step 4: Commit**

```bash
git add skills/vgv-migration/SKILL.md
git commit -m "$(cat <<'EOF'
refactor(vgv-migration): rewrite as battery-included orchestrator

Replaces the monolithic 700-line guide with a thin orchestrator that
delegates to subordinate flai skills phase-by-phase.

Changes:
- Use `dart pub add` for all package installs (no hand-edited pubspec)
- Pin analyzer via `dart pub add analyzer:^6.7.0 --dev`
- Sample feature `counter` -> `posts` (jsonplaceholder.typicode.com)
  exercises networking, serialization, error handling, async state
- Per-phase `dart analyze` verification (fail-fast)
- Full main.dart wiring delegated to flai-logging
- Material 3 theme delegated to flai-material-theming
- l10n keys appended to existing VGV ARB
- TypedGoRoute navigation per flai-navigation
- TalkerScreen route for /debug/logs
- CLAUDE.md now lists "What's Already Wired" so the user knows the
  starter is end-to-end runnable

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Bump plugin version to 0.2.0

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Update version field**

Edit `.claude-plugin/plugin.json`. Change:

```json
  "version": "0.1.0",
```

to:

```json
  "version": "0.2.0",
```

- [ ] **Step 2: Verify**

Run:
```bash
grep '"version"' /Users/silmi/workspace/devtools/flai-plugin/.claude-plugin/plugin.json
```

Expected: `  "version": "0.2.0",`

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "$(cat <<'EOF'
chore: bump plugin version to 0.2.0

vgv-migration rewrite + skill bug fixes constitute a meaningful
behavior change for users invoking the migration skill.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 8: Update README to advertise battery-included migration

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update the Quick Start "Try these prompts" section**

Locate the section starting `**Try these prompts:**` (around line 70-78). Replace the prompts block with:

````markdown
**Try these prompts:**

```
Scaffold a login feature with Riverpod and a fake repository
Migrate this VGV project to the flai stack with a working posts demo
Add an authenticated Dio client with retry on 401
Write golden tests for the OnboardingPage
```
````

The change: second prompt now says "with a working posts demo" so users know vgv-migration produces a runnable starter, not just a refactor.

- [ ] **Step 2: Find the migration mention and update wording (if any)**

Search for "migration" in the README and replace soft language with the battery-included framing if needed:

```bash
grep -n "migration" /Users/silmi/workspace/devtools/flai-plugin/README.md
```

If a line says something like "Migration-friendly — refactor an existing `very_good create` scaffold to flai standards in one prompt", update it to:

```markdown
6. **Migration-friendly** — refactor an existing `very_good create` scaffold into a battery-included flai starter (working `posts` demo, full logger wiring, Material 3 theme) in one prompt
```

If the wording is already accurate, skip this step.

- [ ] **Step 3: Verify**

Run:
```bash
grep -c "battery-included\|posts demo" /Users/silmi/workspace/devtools/flai-plugin/README.md
```

Expected: at least 1 match.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
docs(readme): advertise battery-included starter from vgv-migration

The rewritten skill now produces a runnable posts demo, not just a
refactor. Update Quick Start prompts and migration tagline so users
know the starter is end-to-end working.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

### Task 9: Manual verification on a real VGV project

**Files:** None — this is a manual verification step the user runs.

- [ ] **Step 1: Create a fresh VGV scaffold for testing**

In a separate working directory (NOT inside flai-plugin):

```bash
cd ~/tmp  # or wherever the user keeps test projects
flutter pub global activate very_good_cli
very_good create flutter_app flai_starter_test --description "flai migration smoke test"
cd flai_starter_test
git init && git add -A && git commit -m "chore: initial vgv scaffold"
```

- [ ] **Step 2: Open in Claude Code with flai-plugin installed**

```bash
claude .
```

Verify the plugin is loaded:

```
/plugins
```

Expected: `flai-plugin` appears as installed.

- [ ] **Step 3: Invoke the migration**

In the Claude Code prompt, type:

```
Migrate this VGV project to the flai stack
```

The AI should invoke `flai-vgv-migration` and walk through phases 1-12. It should:
- Run `dart pub add` commands (no manual pubspec edits)
- Reference subordinate skills for canonical templates
- Run `dart analyze` after each phase
- Produce a runnable app with a `posts` page

- [ ] **Step 4: Verify the smoke test**

After the migration completes:

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
dart analyze
flutter test
flutter run
```

All must pass. The app should boot to a `/posts` route showing a list from jsonplaceholder.

- [ ] **Step 5: Report findings**

If any check fails, capture:
- The phase where it failed
- The exact error message
- The file and line referenced

Open issues at the relevant subordinate skill (where the canonical pattern lives) — fix there first so future migrations are correct.

---

## Self-Review

**Spec coverage:**
- Orchestrator pattern (delegate vs duplicate) — Task 6 ✓
- `dart pub add` instead of manual pubspec — Task 6 Phase 2 ✓
- Pin analyzer via `dart pub add analyzer:^6.7.0 --dev` — Task 6 Phase 2 ✓
- TypedGoRoute (go_router_builder) — Task 6 Phase 5d ✓
- Sample feature replaces counter with posts (jsonplaceholder) — Task 6 Phase 5 ✓
- Logger wiring complete (Riverpod observer + Route observer + Dio interceptor + TalkerScreen) — Task 5 + Task 6 Phase 4 ✓
- very_good_analysis pre-work fixes — Tasks 1, 2, 3, 4 ✓
- main.dart template at flai-logging — Task 5 ✓
- Battery-included must-haves: sample feature ✓, logger wiring ✓, sample tests ✓, env config ✓, error handling demo ✓
- Battery-included nice-to-haves: theme (a) ✓, l10n (b) ✓, accessibility applied inline (c) ✓, TalkerScreen route (d) ✓, README updated (g) Task 8 ✓, CI verify reminder (h) Task 6 Phase 12 ✓
- Default very_good_analysis (no separate skill) — relied on, no action needed ✓
- Smoke test manual by user — Task 9 ✓
- Layer 1 (no version pin) + Layer 2 (link pub.dev) for version strategy — Task 6 Phase 2 (no constraints) ✓
- Skip post-scaffold pub upgrade — not added to plan ✓
- AI-extend pattern postponed — not in plan ✓
- Conventional commits — every commit message follows feat/fix/docs/chore/refactor ✓

**Placeholder scan:** None found. All commands are concrete, all code is shown verbatim, all paths are absolute.

**Type consistency:**
- `PostFailure` variants used in `posts_page.dart` switch (`PostNotFound`, `PostTimeout`, `PostNetworkFailure`, `PostUnknown`) match the definition in `post_failure.dart` ✓
- `PostsApi` factory signature `factory PostsApi(Dio dio, {String? baseUrl}) = _PostsApi;` is consistent with how `flai-networking` defines `SongsApi` (matching the Retrofit `_$Generated` constructor convention) ✓
- `connectivityInstanceProvider` defined in Task 2, referenced in Task 1 ✓
- `routerProvider` referenced in `app.dart` matches the `flai-navigation` Logging Integration section's `router` provider name ✓
