---
name: flai-logging
description: Structured application logging with Talker. Use when setting up logging, adding log statements, integrating Talker with Dio or Riverpod, wiring up route observers, or replacing print() calls.
allowed-tools: Read Glob Grep
model: sonnet
---

# Logging

Structured logging using Talker and its ecosystem. A single `talkerProvider` is the source of truth; all other integrations (Dio, Riverpod, router) receive a reference to the same Talker instance.

## Core Standards

Apply these standards to ALL logging work:

- **Single `talkerProvider` in `core/`** — one Talker instance shared across all integrations
- **`TalkerDioLogger` on every Dio instance** — all HTTP requests and responses are logged automatically
- **`TalkerRiverpodObserver` in `ProviderScope`** — provider lifecycle events are logged; configure to hide dispose noise in production
- **`TalkerRouteObserver` on go_router** — all navigation events are logged
- **No `print()`, `debugPrint()`, or `log()` calls** — everything goes through talker
- **Use appropriate log levels** — `debug` for development noise, `info` for notable events, `warning` for recoverable issues, `error` + `handle` for caught exceptions

## Talker Provider

```dart
// core/providers/talker_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:talker_flutter/talker_flutter.dart';

part 'talker_provider.g.dart';

@Riverpod(keepAlive: true)
Talker talker(TalkerRef ref) {
  return TalkerFlutter.init(
    settings: const TalkerSettings(
      enabled: true,
      useConsoleLogs: true,
    ),
  );
}
```

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

## Dio Integration

```dart
// core/http/dio_provider.dart (relevant section)
dio.interceptors.add(
  TalkerDioLogger(
    talker: ref.watch(talkerProvider),
    settings: const TalkerDioLoggerSettings(
      printRequestHeaders: false,   // hide in production
      printResponseMessage: true,
      printErrorData: true,
      printErrorHeaders: false,
    ),
  ),
);
```

## Router Integration

```dart
// core/router/router.dart
import 'package:go_router/go_router.dart';
import 'package:talker_flutter/talker_flutter.dart';

GoRouter buildRouter(Talker talker) => GoRouter(
  observers: [TalkerRouteObserver(talker)],
  routes: $appRoutes,
);
```

## Log Levels

```dart
// Usage examples
talker.debug('Fetching user profile for id=$userId');          // dev noise
talker.info('User signed in: $email');                         // notable event
talker.warning('Cache miss — falling back to network');        // recoverable
talker.error('Failed to parse response', err, stackTrace);     // logged error
talker.handle(exception, stackTrace, 'Context message');       // caught exception
```

## In-App Log Viewer

Display logs in debug builds using `TalkerScreen`:

```dart
// Add to your developer menu or debug build
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (_) => TalkerScreen(talker: context.read(talkerProvider)),
  ),
);
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `print('debug: $value')` | Not structured; disabled in release; no log levels | `talker.debug('$value')` |
| `debugPrint('error: $e')` | Flutter-only; no log levels; not filterable | `talker.error('$e', e, stackTrace)` |
| Multiple Talker instances | Logs split across instances; observers miss events | Single `talkerProvider` injected everywhere |
| `TalkerDioLoggerSettings(printRequestHeaders: true)` in production | Leaks auth tokens in logs | Disable header logging in non-debug builds |
| Logging sensitive data (`password`, token values) | Readable in crash reports and device logs | Log only non-sensitive identifiers |
