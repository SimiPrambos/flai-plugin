---
name: flai-networking
description: HTTP networking with Dio and Retrofit, and real-time communication with Socket.IO. Use when setting up the HTTP client, writing Retrofit API services, handling network errors, implementing retry logic, or creating WebSocket connections.
allowed-tools: Read Glob Grep
model: sonnet
---

# Networking

HTTP networking via Dio + Retrofit, and real-time communication via Socket.IO. All networking is wired through Riverpod providers.

## Core Standards

Apply these standards to ALL networking work:

- **Single `dioProvider` singleton** — one Dio instance for the entire app, defined in `core/http/dio_provider.dart`
- **`TalkerDioLogger` on the Dio instance** — every request and response is logged automatically
- **Connectivity-aware interceptor on Dio** — fail fast when offline; retry only on timeout
- **Retrofit services return raw `Future<T>`** — the repository layer wraps results in `Either<Failure, T>`; never return Either from Retrofit
- **Never hardcode base URLs** — always read from `Env.apiBaseUrl`
- **Socket.IO via a service class** — wrap `socket_io_client` in a service with a `StreamController.broadcast()`; expose data through Riverpod `StreamProvider`
- **Explicit websocket transport** — always set `setTransports(['websocket'])` to prevent polling fallback

## Dio Setup

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

## Retrofit Service

Define API services with `@RestApi` and `@riverpod`. Return raw types — no Either.

```dart
// features/songs/data/datasources/songs_api.dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'songs_api.g.dart';

@RestApi()
abstract class SongsApi {
  factory SongsApi(Dio dio, {String? baseUrl}) = _SongsApi;

  @GET('/songs')
  Future<List<SongModel>> getSongs();

  @GET('/songs/{id}')
  Future<SongModel> getSongById(@Path('id') String id);

  @POST('/songs')
  Future<SongModel> createSong(@Body() Map<String, dynamic> body);
}

@riverpod
SongsApi songsApi(SongsApiRef ref) {
  return SongsApi(ref.watch(dioProvider));
}
```

## Repository Error Wrapping

The repository catches Dio exceptions and maps them to domain failures:

```dart
// features/songs/data/repositories/songs_repository_impl.dart
class SongsRepositoryImpl implements SongsRepository {
  const SongsRepositoryImpl(this._api);
  final SongsApi _api;

  @override
  Future<Either<SongFailure, List<Song>>> getSongs() async {
    try {
      final models = await _api.getSongs();
      return right(models.map((m) => m.toDomain()).toList());
    } on DioException catch (e) {
      return left(_mapDioException(e));
    } catch (e) {
      return left(const SongFailure.unknown());
    }
  }

  SongFailure _mapDioException(DioException e) {
    return switch (e.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.receiveTimeout =>
        const SongFailure.timeout(),
      DioExceptionType.badResponse when e.response?.statusCode == 401 =>
        const SongFailure.unauthorized(),
      DioExceptionType.badResponse when e.response?.statusCode == 404 =>
        const SongFailure.notFound(),
      _ => const SongFailure.unknown(),
    };
  }
}
```

## Connectivity-Aware Interceptor

```dart
// core/http/connectivity_interceptor.dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:dio/dio.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

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

## Socket.IO Pattern

```dart
// features/chat/data/services/chat_socket_service.dart
import 'dart:async';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:socket_io_client/socket_io_client.dart' as io;

part 'chat_socket_service.g.dart';

class ChatSocketService {
  ChatSocketService._();

  late io.Socket _socket;
  final _messageController = StreamController<ChatMessage>.broadcast();

  Stream<ChatMessage> get messageStream => _messageController.stream;

  void connect(String url, String token) {
    _socket = io.io(
      url,
      io.OptionBuilder()
          .setTransports(['websocket'])
          .disableAutoConnect()
          .setExtraHeaders({'Authorization': 'Bearer $token'})
          .build(),
    );
    _socket
      ..connect()
      ..on('message', (data) {
        _messageController.add(ChatMessage.fromJson(data as Map<String, dynamic>));
      });
  }

  void sendMessage(String text) => _socket.emit('message', {'text': text});

  void dispose() {
    _socket.disconnect();
    _messageController.close();
  }
}

@riverpod
ChatSocketService chatSocketService(ChatSocketServiceRef ref) {
  final service = ChatSocketService._();
  ref.onDispose(service.dispose);
  return service;
}

@riverpod
Stream<ChatMessage> chatMessages(ChatMessagesRef ref) {
  return ref.watch(chatSocketServiceProvider).messageStream;
}
```

## Quick Reference

| Package | Purpose |
| --- | --- |
| `dio` | HTTP client |
| `retrofit` | Code-generated REST API client |
| `socket_io_client` | WebSocket/Socket.IO client |
| `talker_dio_logger` | Automatic request/response logging |

| Command | Purpose |
| --- | --- |
| `dart run build_runner build --delete-conflicting-outputs` | Generate Retrofit and Riverpod code |
