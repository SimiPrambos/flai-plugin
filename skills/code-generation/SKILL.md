---
name: flai-code-generation
description: Code generation with build_runner. Use when running build_runner, configuring build.yaml, adding new generators (freezed, riverpod_generator, retrofit), troubleshooting generated file conflicts, or managing analyzer version compatibility.
allowed-tools: Read Glob Grep Bash
model: sonnet
---

# Code Generation

Code generation is central to this stack — Freezed, Riverpod, Retrofit, and go_router_builder all rely on `build_runner`. Proper `build.yaml` configuration and analyzer version pinning prevent common conflicts.

## Core Standards

Apply these standards to ALL code generation work:

- **Generated files are committed to git** — `.g.dart` and `.freezed.dart` files are checked in; CI does not regenerate them
- **Never manually edit generated files** — changes are overwritten on the next build; edit the source file and regenerate
- **`build.yaml` restricts generators to relevant paths** — avoids scanning unrelated files; significantly speeds up build times
- **Pin `analyzer` version in `pubspec.yaml`** — `go_router_builder` and `freezed` require different analyzer versions; explicit pinning prevents silent breakage
- **`--delete-conflicting-outputs` on every build** — prevents stale generated files from blocking new builds

## Commands

```bash
# Full build (run after adding/changing any annotated class)
dart run build_runner build --delete-conflicting-outputs

# Watch mode (run during active development)
dart run build_runner watch --delete-conflicting-outputs

# Clean generated files (run when switching branches or resolving conflicts)
dart run build_runner clean
```

## build.yaml Configuration

```yaml
# build.yaml (at project root)
targets:
  $default:
    builders:
      # Restrict freezed to model and entity files only
      freezed:
        generate_for:
          include:
            - lib/features/**/models/**.dart
            - lib/features/**/entities/**.dart
            - lib/features/**/failures/**.dart
            - lib/core/error/**.dart

      # Restrict json_serializable to data layer models
      json_serializable:
        generate_for:
          include:
            - lib/features/**/models/**.dart

      # Restrict riverpod_generator to notifiers and provider files
      riverpod_generator:
        generate_for:
          include:
            - lib/features/**/notifiers/**.dart
            - lib/features/**/providers.dart
            - lib/features/**/datasources/**.dart
            - lib/features/**/repositories/**.dart
            - lib/features/**/services/**.dart
            - lib/core/**_provider.dart
            - lib/core/providers.dart

      # Restrict retrofit to API datasource files
      retrofit_generator:
        options:
          format_output: false  # avoids conflicts with dart format hook
        generate_for:
          include:
            - lib/features/**/datasources/**_api.dart

      # Restrict go_router_builder to router file only
      go_router_builder:
        generate_for:
          include:
            - lib/core/router/router.dart
```

## Analyzer Version Pinning

Add to `pubspec.yaml` dev_dependencies:

```yaml
dev_dependencies:
  # Explicit analyzer pin to resolve go_router_builder + freezed conflict.
  # go_router_builder requires <6.0.0; freezed requires ^6.0.0.
  # Check pub.dev for the latest compatible version when upgrading packages.
  analyzer: ^6.7.0
  build_runner: ^2.4.13
  freezed: ^2.5.7
  go_router_builder: ^2.7.1
  json_serializable: ^6.8.0
  retrofit_generator: ^9.1.5
  riverpod_generator: ^2.4.3
```

After changing versions, run:

```bash
dart pub get
dart pub deps
```

Verify no `Incompatible version constraints` errors in the output.

## .gitignore for Generated Files

Generated files should be committed, but add them explicitly to ensure they are tracked:

```gitignore
# Generated files are committed — do NOT add *.g.dart or *.freezed.dart here
# Only ignore build artifacts
.dart_tool/build/
```

## Common Conflicts and Fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Incompatible version constraints on analyzer` | `go_router_builder` and `freezed` require different analyzer versions | Pin `analyzer` explicitly in `dev_dependencies` |
| `Existing output ... conflicts with an output` | Stale generated file from a previous build | Run `dart run build_runner build --delete-conflicting-outputs` |
| `Could not find package 'riverpod_annotation'` | Missing `riverpod_annotation` in `dependencies` (not just `dev_dependencies`) | Add `riverpod_annotation` to `dependencies` |
| Build hangs with no output | `build_runner watch` is already running in another terminal | Kill the other process; only one watcher at a time |
| Generated file not updating | Source file not in the `generate_for` path in `build.yaml` | Update `build.yaml` to include the file's path |

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| Editing `.g.dart` or `.freezed.dart` directly | Changes are lost on next `build_runner` run | Edit the source file and regenerate |
| `.g.dart` in `.gitignore` | CI must regenerate; slow builds; merge conflicts on generation | Commit generated files |
| No `build.yaml` restrictions | Every file is scanned; build time grows with project size | Use `generate_for` to target specific paths |
| Running `build_runner` without `--delete-conflicting-outputs` | Stale outputs block new generation | Always include the flag |
