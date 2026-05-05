---
name: flai-serialization
description: Data models and JSON serialization with Freezed and json_serializable. Use when creating data models, domain entities, JSON parsing code, or union/sealed types for state representation.
allowed-tools: Read Glob Grep
model: sonnet
---

# Serialization & Models

Immutable data models using Freezed with JSON serialization via json_serializable. Domain entities use Freezed without JSON annotations.

## Core Standards

Apply these standards to ALL model and serialization work:

- **`@freezed` for all data models and domain entities** — generates `copyWith`, `==`, `hashCode`, and `toString` automatically
- **Never apply `@freezed` to Notifier state** — Riverpod `Notifier`/`AsyncNotifier` classes are not Freezed; use primitive types or plain Dart classes for their state
- **Data layer models have `fromJson`/`toJson`** — via `@JsonSerializable()` inside Freezed; domain entities do not
- **Union types (sealed classes) via Freezed** — use `@freezed` factories for sealed state variants and domain failure types
- **Disable `invalid_annotation_target` warning** — required for Freezed to work with `@JsonSerializable` without linting errors
- **Pin `analyzer` version** — `go_router_builder` and `freezed` require different analyzer versions; pin to a compatible version in `pubspec.yaml`

## Data Layer Model (with JSON)

```dart
// features/songs/data/models/song_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'song_model.freezed.dart';
part 'song_model.g.dart';

@freezed
class SongModel with _$SongModel {
  const factory SongModel({
    required String id,
    required String title,
    required String artist,
    @JsonKey(name: 'cover_url') String? coverUrl,
  }) = _SongModel;

  factory SongModel.fromJson(Map<String, dynamic> json) =>
      _$SongModelFromJson(json);
}

// Extension to convert to domain entity
extension SongModelX on SongModel {
  Song toDomain() => Song(id: id, title: title, artist: artist);
}
```

## Domain Entity (no JSON)

```dart
// features/songs/domain/entities/song.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'song.freezed.dart';

@freezed
class Song with _$Song {
  const factory Song({
    required String id,
    required String title,
    required String artist,
  }) = _Song;
}
```

## Union Types (Sealed Classes)

Use Freezed factories to create sealed class hierarchies for domain failures and result types:

```dart
// features/songs/domain/failures/song_failure.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'song_failure.freezed.dart';

@freezed
sealed class SongFailure with _$SongFailure {
  const factory SongFailure.notFound() = SongNotFound;
  const factory SongFailure.unauthorized() = SongUnauthorized;
  const factory SongFailure.timeout() = SongTimeout;
  const factory SongFailure.unknown() = SongUnknown;
}
```

Pattern match in the presentation layer:

```dart
result.fold(
  (failure) => switch (failure) {
    SongNotFound() => 'Song not found',
    SongUnauthorized() => 'Please log in',
    SongTimeout() => 'Request timed out',
    SongUnknown() => 'Something went wrong',
  },
  (song) => song.title,
);
```

## analysis_options.yaml Configuration

Add to suppress Freezed annotation warnings:

```yaml
analyzer:
  errors:
    invalid_annotation_target: ignore
```

## Analyzer Version Pinning

`go_router_builder` and `freezed` require different analyzer versions. Pin explicitly in `pubspec.yaml` to avoid conflicts:

```yaml
dev_dependencies:
  # Pin analyzer to a version compatible with both freezed and go_router_builder.
  # go_router_builder requires analyzer >=4.4.0 <6.0.0
  # freezed requires analyzer ^6.0.0 — check pub.dev for the latest compatible range.
  analyzer: ^6.7.0
  build_runner: ^2.4.0
  freezed: ^2.5.0
  go_router_builder: ^2.7.0
  json_serializable: ^6.8.0
```

Run `dart pub deps` to verify no version conflicts after changing these.

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `@freezed` on a Riverpod `Notifier` class | Freezed generates immutable classes; Notifiers are mutable stateful objects | Use plain classes or primitive types for notifier state |
| `fromJson` in domain entities | Couples domain layer to JSON format; domain should be pure | Keep JSON parsing in data models; convert to domain entities with extension methods |
| Manual `==` and `hashCode` on model classes | Error-prone; breaks when fields are added | Always use `@freezed` which generates these automatically |
| Nullable fields without `@JsonKey(name: ...)` | Field name mismatch between API snake_case and Dart camelCase | Use `@JsonKey(name: 'snake_case_name')` on all fields that differ |
