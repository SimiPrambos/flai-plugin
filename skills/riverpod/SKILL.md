---
name: flai-riverpod
description: State management and dependency injection with Riverpod. Use when creating, modifying, or reviewing Riverpod providers, notifiers, state classes, or ref.watch/ref.read usage.
allowed-tools: Read Glob Grep
model: sonnet
---

# Riverpod

State management and dependency injection using Riverpod with code generation. All providers use `@riverpod` annotations — no manual `Provider()`, `StateNotifierProvider()`, or `ChangeNotifierProvider()`.

## Core Standards

Apply these standards to ALL Riverpod work:

- **`@riverpod` code generation mandatory** — never write `Provider(...)`, `FutureProvider(...)`, or `StateNotifierProvider(...)` manually
- **`AsyncNotifier<T>` for async state, `Notifier<T>` for sync state** — never use `StateNotifier<T>`
- **Providers are `autoDispose` by default** — code generation applies `autoDispose` automatically; use `@Riverpod(keepAlive: true)` only for global singleton providers
- **`ref.watch()` in `build()` methods only** — establishes subscriptions and triggers rebuilds
- **`ref.read()` in callbacks and event handlers only** — avoids stale closures; never use `ref.read()` in `build()`
- **One provider per file where possible** — keeps files focused and reduces build_runner output conflicts
- **Feature-level `providers.dart` barrel file** — re-export all providers for a feature from one file

## AsyncNotifier Pattern

Use `AsyncNotifier<T>` when the initial state requires an async operation (API call, database read).

```dart
// features/songs/presentation/notifiers/songs_notifier.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'songs_notifier.g.dart';

@riverpod
class SongsNotifier extends _$SongsNotifier {
  @override
  Future<List<Song>> build() async {
    final repo = ref.watch(songRepositoryProvider);
    return (await repo.getSongs()).fold(
      (failure) => throw failure,
      (songs) => songs,
    );
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(
      () => ref.read(songRepositoryProvider).getSongs().then(
            (result) => result.fold((f) => throw f, (s) => s),
          ),
    );
  }
}
```

Consume in a widget:

```dart
class SongsPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final songsAsync = ref.watch(songsNotifierProvider);
    return songsAsync.when(
      loading: () => const CircularProgressIndicator(),
      error: (err, st) => Text('Error: $err'),
      data: (songs) => ListView.builder(
        itemCount: songs.length,
        itemBuilder: (context, i) => ListTile(title: Text(songs[i].title)),
      ),
    );
  }
}
```

## Notifier Pattern

Use `Notifier<T>` when the state is synchronous (no async initialization needed).

```dart
// features/counter/presentation/notifiers/counter_notifier.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'counter_notifier.g.dart';

@riverpod
class CounterNotifier extends _$CounterNotifier {
  @override
  int build() => 0;

  void increment() => state++;
  void decrement() => state--;
  void reset() => state = 0;
}
```

## Simple Provider Patterns

```dart
// Simple value provider (autoDispose by default)
@riverpod
String greeting(GreetingRef ref) => 'Hello';

// FutureProvider equivalent
@riverpod
Future<User> currentUser(CurrentUserRef ref) async {
  final repo = ref.watch(userRepositoryProvider);
  return (await repo.getCurrentUser()).fold((f) => throw f, (u) => u);
}

// StreamProvider equivalent
@riverpod
Stream<List<Message>> messages(MessagesRef ref) {
  return ref.watch(socketServiceProvider).messageStream;
}

// Singleton provider (not autoDispose)
@Riverpod(keepAlive: true)
Dio dio(DioRef ref) {
  final talker = ref.watch(talkerProvider);
  return Dio()..interceptors.add(TalkerDioLogger(talker: talker));
}
```

## Provider Organization

Structure providers at the feature level:

```dart
// features/songs/providers.dart
export 'data/repositories/song_repository_provider.dart';
export 'presentation/notifiers/songs_notifier.dart';
```

Global providers live in `core/providers.dart`:

```dart
// core/providers.dart
export 'config/env.dart';
export 'database/isar_provider.dart';
export 'http/dio_provider.dart';
export 'providers/talker_provider.dart';
export 'providers/connectivity_provider.dart';
```

## ref.watch vs ref.read

```dart
// ✅ ref.watch in build() — subscribes and rebuilds when value changes
@override
Widget build(BuildContext context, WidgetRef ref) {
  final count = ref.watch(counterNotifierProvider);
  return Text('$count');
}

// ✅ ref.read in callbacks — one-time read without subscription
ElevatedButton(
  onPressed: () => ref.read(counterNotifierProvider.notifier).increment(),
  child: const Text('Increment'),
);

// ❌ ref.read in build() — stale value, won't rebuild on change
@override
Widget build(BuildContext context, WidgetRef ref) {
  final count = ref.read(counterNotifierProvider); // WRONG
  return Text('$count');
}

// ❌ ref.watch in callbacks — creates new subscriptions on every event
onPressed: () {
  final count = ref.watch(counterNotifierProvider); // WRONG
}
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `StateNotifierProvider<MyNotifier, MyState>` | Deprecated; verbose; no code generation | Use `@riverpod` with `Notifier<T>` |
| `Provider((ref) => MyService())` (manual) | No code generation; manual lifecycle management | Use `@riverpod` function provider |
| `ref.watch()` inside a callback | Creates subscriptions outside `build()`; memory leaks | Use `ref.read()` in callbacks |
| One providers.dart file for the entire app | Hard to navigate; slow code generation for large projects | Feature-level providers.dart files |
| `keepAlive: true` on everything | Providers never dispose; memory grows unbounded | Use `keepAlive: true` only for true singletons (Dio, Isar, Talker) |
```