---
name: flai-error-handling
description: Functional error handling with fpdart. Use when defining failure types, writing repository return types, handling errors across layers, or converting Either results in the presentation layer.
allowed-tools: Read Glob Grep
model: sonnet
---

# Error Handling

Functional error handling using `Either<Failure, T>` from `package:fpdart`. Errors are explicit values that propagate through the data and domain layers; the presentation layer converts them to `AsyncValue` states.

## Core Standards

Apply these standards to ALL error handling work:

- **`Either<Failure, T>` propagates through data and domain layers only** — never return Either from a Riverpod notifier or widget method
- **Presentation layer converts Either via `.fold()`** — turn Left into a thrown exception (for AsyncValue.error) or a state object
- **Never throw exceptions across layer boundaries** — catch at the data layer and wrap in `Left(Failure)`; never let DioException or IsarError bubble into domain or presentation
- **Sealed `Failure` hierarchy** — base `Failure` sealed class in `core/error/failures.dart`; feature-specific subclasses in `features/<name>/domain/failures/`
- **`TaskEither` for complex async chains** — use `TaskEither` when composing multiple fallible async operations to avoid nested `.fold()` calls

## Base Failure Class

```dart
// core/error/failures.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'failures.freezed.dart';

@freezed
sealed class Failure with _$Failure {
  const factory Failure.network() = NetworkFailure;
  const factory Failure.unauthorized() = UnauthorizedFailure;
  const factory Failure.unknown([String? message]) = UnknownFailure;
}
```

## Feature Failure Subclass

```dart
// features/auth/domain/failures/auth_failure.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_failure.freezed.dart';

@freezed
sealed class AuthFailure with _$AuthFailure {
  const factory AuthFailure.invalidCredentials() = InvalidCredentials;
  const factory AuthFailure.emailAlreadyInUse() = EmailAlreadyInUse;
  const factory AuthFailure.weakPassword() = WeakPassword;
  const factory AuthFailure.network() = AuthNetworkFailure;
}
```

## Data Layer Usage

```dart
// features/auth/data/repositories/auth_repository_impl.dart
@override
Future<Either<AuthFailure, User>> signIn(String email, String password) async {
  try {
    final userModel = await _api.signIn(email: email, password: password);
    return right(userModel.toDomain());
  } on DioException catch (e) {
    return left(
      e.response?.statusCode == 401
          ? const AuthFailure.invalidCredentials()
          : const AuthFailure.network(),
    );
  } catch (_) {
    return left(const AuthFailure.network());
  }
}
```

## Domain Layer (pass-through)

Domain use cases pass Either through unchanged — they do not catch exceptions:

```dart
// features/auth/domain/use_cases/sign_in_use_case.dart
class SignInUseCase {
  const SignInUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AuthFailure, User>> call(String email, String password) =>
      _repository.signIn(email, password);
}
```

## Presentation Layer Conversion

Convert Either to AsyncValue in the notifier's `build()` or action methods:

```dart
// features/auth/presentation/notifiers/auth_notifier.dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  Future<User?> build() async => null;

  Future<void> signIn(String email, String password) async {
    state = const AsyncLoading();
    final result = await ref.read(signInUseCaseProvider).call(email, password);
    state = result.fold(
      (failure) => AsyncError(failure, StackTrace.current),
      AsyncData.new,
    );
  }
}
```

In the widget:

```dart
ref.listen(authNotifierProvider, (_, state) {
  state.whenOrNull(
    error: (failure, _) => switch (failure as AuthFailure) {
      InvalidCredentials() => ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Invalid email or password')),
        ),
      AuthNetworkFailure() => ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('No internet connection')),
        ),
      _ => null,
    },
  );
});
```

## TaskEither for Complex Chains

Use `TaskEither` when multiple fallible async operations must run in sequence:

```dart
TaskEither<AuthFailure, void> signUpAndWelcome(String email, String password) =>
    TaskEither.tryCatch(
      () => _api.signUp(email: email, password: password),
      (e, _) => const AuthFailure.network(),
    )
        .flatMap(
          (user) => TaskEither.tryCatch(
            () => _emailService.sendWelcome(user.email),
            (e, _) => const AuthFailure.network(),
          ),
        )
        .map((_) => unit);

// Run it:
final result = await signUpAndWelcome(email, password).run();
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `throw` across layer boundaries | Exceptions are invisible in type signatures; callers forget to handle them | Return `Left(Failure)` instead |
| `Either` in a Riverpod notifier's `build()` return type | Presentation shouldn't expose Either; widgets don't know how to handle it | Convert with `.fold()` inside the notifier before emitting state |
| Single `Failure` variant for all errors | Can't distinguish errors in the UI; every failure looks the same | Create sealed failure classes with specific variants per feature |
| Generic `catch (e)` without re-wrapping | Swallows stack traces; loses error context | Catch specific exceptions; log unknown errors via talker before wrapping |
