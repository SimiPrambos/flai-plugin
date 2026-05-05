---
name: flai-connectivity
description: Network connectivity monitoring with connectivity_plus. Use when checking network status, implementing offline handling, showing connectivity banners, or adding network-aware behavior to Dio requests.
allowed-tools: Read Glob Grep
model: sonnet
---

# Connectivity

Network connectivity monitoring using `connectivity_plus`. The app exposes a `connectivityProvider` as a `StreamProvider` that all features can subscribe to. The Dio layer uses a `ConnectivityInterceptor` to fail fast when offline.

## Core Standards

Apply these standards to ALL connectivity work:

- **`connectivityProvider` as `StreamProvider<List<ConnectivityResult>>`** — single source of truth for network status; all features subscribe to this provider
- **Never access `Connectivity()` directly in widgets or notifiers** — always use the provider
- **Connectivity interceptor handles fail-fast** — Dio rejects requests immediately when offline; the UI layer does not need to guard every API call
- **Distinguish "no connection" from "timeout"** — offline → fail fast and show banner; timeout → retry with backoff

## Connectivity Provider

```dart
// core/providers/connectivity_provider.dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'connectivity_provider.g.dart';

@Riverpod(keepAlive: true)
Stream<List<ConnectivityResult>> connectivity(ConnectivityRef ref) =>
    Connectivity().onConnectivityChanged;

bool isOnline(List<ConnectivityResult> result) =>
    result.isNotEmpty && !result.contains(ConnectivityResult.none);
```

## Offline Banner Widget

```dart
// core/widgets/connectivity_banner.dart
class ConnectivityBanner extends ConsumerWidget {
  const ConnectivityBanner({required this.child, super.key});
  final Widget child;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final connectivityAsync = ref.watch(connectivityProvider);

    return Column(
      children: [
        connectivityAsync.when(
          data: (result) => isOnline(result)
              ? const SizedBox.shrink()
              : ColoredBox(
                  color: Colors.red,
                  child: Padding(
                    padding: const EdgeInsets.all(8),
                    child: Text(
                      context.l10n.noInternetConnection,
                      style: const TextStyle(color: Colors.white),
                    ),
                  ),
                ),
          loading: SizedBox.shrink.new,
          error: (_, __) => const SizedBox.shrink(),
        ),
        Expanded(child: child),
      ],
    );
  }
}
```

## Dio Connectivity Interceptor

See the `flai-networking` skill for the full `ConnectivityInterceptor` implementation. The interceptor is added to the Dio instance in `core/http/dio_provider.dart`.

## Checking Connectivity Once

When you need a one-time check (e.g., before a manual refresh):

```dart
Future<void> refresh() async {
  final result = await Connectivity().checkConnectivity();
  if (!isOnline(result)) {
    state = AsyncError(const Failure.network(), StackTrace.current);
    return;
  }
  state = const AsyncLoading();
  // ... proceed with network call
}
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `Connectivity().checkConnectivity()` in every widget | Bypasses the provider; creates duplicate streams | Subscribe to `connectivityProvider` instead |
| Blocking API calls when offline in the UI | Poor UX; shows loading spinner forever | Let the Dio interceptor reject offline requests immediately |
| Treating `ConnectivityResult.wifi` as guaranteed internet | Device may be on a network with no route | Connectivity result indicates network interface, not actual internet reachability |