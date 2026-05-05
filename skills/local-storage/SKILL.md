---
name: flai-local-storage
description: Local data persistence with Isar, SharedPreferences, and FlutterSecureStorage. Use when reading or writing local data, setting up database schemas, choosing a storage strategy, or implementing offline-first features.
allowed-tools: Read Glob Grep
model: sonnet
---

# Local Storage

Three-tier local storage: SharedPreferences for simple non-sensitive preferences, FlutterSecureStorage for secrets and tokens, and Isar for complex relational data with reactive queries.

## Core Standards

Apply these standards to ALL local storage work:

- **Never store auth tokens or sensitive data in SharedPreferences** — SharedPreferences is plaintext; use FlutterSecureStorage for anything sensitive
- **Single `isarProvider`** — one `FutureProvider<Isar>` in `core/database/isar_provider.dart`; never open multiple Isar instances
- **Service classes per entity group** — wrap Isar collections in service classes; never access `isar.collection<T>()` directly from providers
- **`StreamProvider` for reactive Isar queries** — use Isar's `.watch()` to create reactive streams; expose via `StreamProvider`
- **FlutterSecureStorage for all token and key storage** — backed by iOS Keychain and Android Keystore

## When to Use Each

| Storage | Use for | Never use for |
| --- | --- | --- |
| `SharedPreferences` | Theme, locale, onboarding state, feature flags | Tokens, passwords, PII, API keys |
| `FlutterSecureStorage` | Auth tokens, refresh tokens, API keys, encryption keys | Large binary data |
| `Isar` | User data, cached API responses, offline content | Simple key-value settings |

## Isar Setup

```dart
// core/database/isar_provider.dart
import 'package:isar/isar.dart';
import 'package:path_provider/path_provider.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'isar_provider.g.dart';

@Riverpod(keepAlive: true)
Future<Isar> isar(IsarRef ref) async {
  final dir = await getApplicationDocumentsDirectory();
  return Isar.open(
    [UserSchema, MessageSchema],  // add all schemas here
    directory: dir.path,
  );
}
```

## Isar Service Pattern

```dart
// features/messages/data/services/message_service.dart
import 'package:isar/isar.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'message_service.g.dart';

class MessageService {
  const MessageService(this._isar);
  final Isar _isar;

  Future<void> save(MessageModel message) =>
      _isar.writeTxn(() => _isar.messages.put(message));

  Future<List<MessageModel>> getAll() =>
      _isar.messages.where().findAll();

  Stream<List<MessageModel>> watchAll() =>
      _isar.messages.where().watch(fireImmediately: true);

  Future<void> deleteById(int id) =>
      _isar.writeTxn(() => _isar.messages.delete(id));
}

@riverpod
Future<MessageService> messageService(MessageServiceRef ref) async {
  final isar = await ref.watch(isarProvider.future);
  return MessageService(isar);
}

// Reactive query exposed as StreamProvider
@riverpod
Stream<List<MessageModel>> allMessages(AllMessagesRef ref) async* {
  final service = await ref.watch(messageServiceProvider.future);
  yield* service.watchAll();
}
```

## SharedPreferences Pattern

```dart
// core/providers/preferences_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:shared_preferences/shared_preferences.dart';

part 'preferences_provider.g.dart';

@Riverpod(keepAlive: true)
Future<SharedPreferences> sharedPreferences(SharedPreferencesRef ref) =>
    SharedPreferences.getInstance();

// Feature-level preference example
@riverpod
class ThemeNotifier extends _$ThemeNotifier {
  @override
  Future<bool> build() async {
    final prefs = await ref.watch(sharedPreferencesProvider.future);
    return prefs.getBool('dark_mode') ?? false;
  }

  Future<void> toggle() async {
    final prefs = await ref.read(sharedPreferencesProvider.future);
    final next = !(state.value ?? false);
    await prefs.setBool('dark_mode', next);
    state = AsyncData(next);
  }
}
```

## FlutterSecureStorage Pattern

```dart
// core/providers/secure_storage_provider.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'secure_storage_provider.g.dart';

@Riverpod(keepAlive: true)
FlutterSecureStorage secureStorage(SecureStorageRef ref) =>
    const FlutterSecureStorage(
      aOptions: AndroidOptions(encryptedSharedPreferences: true),
    );

// Token management example
@riverpod
class AuthTokenNotifier extends _$AuthTokenNotifier {
  static const _key = 'auth_token';

  @override
  Future<String?> build() async {
    final storage = ref.watch(secureStorageProvider);
    return storage.read(key: _key);
  }

  Future<void> save(String token) async {
    await ref.read(secureStorageProvider).write(key: _key, value: token);
    state = AsyncData(token);
  }

  Future<void> clear() async {
    await ref.read(secureStorageProvider).delete(key: _key);
    state = const AsyncData(null);
  }
}
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `prefs.setString('token', jwt)` | JWT stored as plaintext; readable by anyone with USB debugging | Use `FlutterSecureStorage.write()` |
| Multiple `Isar.open()` calls | Multiple open instances cause errors and data inconsistency | Single `isarProvider` in `core/`; inject everywhere |
| `isar.collection<T>()` directly in providers | Bypasses service layer; hard to test | Wrap in service classes |
| `FutureProvider` on Isar queries | One-time read; doesn't update when data changes | Use `StreamProvider` with `isar.watch()` |
