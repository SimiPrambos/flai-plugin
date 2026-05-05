---
name: flai-environment-config
description: Environment configuration with envied. Use when managing API base URLs, secrets, feature flags, environment-specific settings, or .env files.
allowed-tools: Read Glob Grep
model: sonnet
---

# Environment Config

Environment configuration using `envied` for local development and `--dart-define` for CI/CD release builds. Secrets are never hardcoded in source code.

## Core Standards

Apply these standards to ALL environment configuration work:

- **`.env` in `.gitignore`** — never commit real environment values; commit `.env.example` with placeholder values instead
- **`obfuscate: true` on all secret fields** — envied compiles obfuscated values into the binary; plain `@EnviedField` is readable via `strings` binary analysis
- **All envied fields in `lib/core/config/env.dart`** — single source of truth for all configuration
- **`--dart-define` for CI/CD release builds** — envied reads `.env` locally; CI passes values via `--dart-define` without committing secrets
- **Provide `.env.example`** — always commit a `.env.example` showing required variables with placeholder values

## Env Class

```dart
// lib/core/config/env.dart
import 'package:envied/envied.dart';

part 'env.g.dart';

@Envied(path: '.env')
abstract class Env {
  @EnviedField(varName: 'API_BASE_URL')
  static const String apiBaseUrl = _Env.apiBaseUrl;

  @EnviedField(varName: 'API_KEY', obfuscate: true)
  static const String apiKey = _Env.apiKey;

  @EnviedField(varName: 'SOCKET_URL')
  static const String socketUrl = _Env.socketUrl;

  @EnviedField(varName: 'ENVIRONMENT', defaultValue: 'development')
  static const String environment = _Env.environment;
}
```

## .env.example

Commit this file to describe required variables:

```bash
# API configuration
API_BASE_URL=https://api.example.com
API_KEY=your_api_key_here
SOCKET_URL=wss://ws.example.com

# App environment: development | staging | production
ENVIRONMENT=development
```

## .env (local only, gitignored)

```bash
# Real values — never commit this file
API_BASE_URL=https://api.myapp.com
API_KEY=sk-real-api-key-here
SOCKET_URL=wss://ws.myapp.com
ENVIRONMENT=development
```

## CI/CD with --dart-define

In GitHub Actions or other CI systems, pass values at build time:

```yaml
# .github/workflows/build.yaml
- name: Build APK
  run: |
    flutter build apk \
      --dart-define=API_BASE_URL=${{ secrets.API_BASE_URL }} \
      --dart-define=API_KEY=${{ secrets.API_KEY }} \
      --dart-define=SOCKET_URL=${{ secrets.SOCKET_URL }} \
      --dart-define=ENVIRONMENT=production
```

For `--dart-define` to work with envied, the `Env` fields must also accept environment variables. Add a separate env class for CI or use `String.fromEnvironment`:

```dart
// When not using envied in CI, read --dart-define directly:
static const String apiBaseUrl = String.fromEnvironment('API_BASE_URL');
```

Consider maintaining two env classes: `Env` (envied, for local) and `EnvCI` (fromEnvironment, for CI).

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `const apiKey = 'sk-abc123'` | Hardcoded secret; compiled into binary; extractable via reverse engineering | Use envied with `obfuscate: true` |
| Committing `.env` to git | Exposes real secrets in version history forever | `.gitignore` the `.env` file; commit `.env.example` only |
| `@EnviedField` without `obfuscate: true` for secrets | Value readable via `strings` on the binary | Always use `obfuscate: true` for API keys and passwords |
| `--dart-define` values in local development | Easy to forget; requires long build commands | Use envied + `.env` locally; `--dart-define` only in CI |
