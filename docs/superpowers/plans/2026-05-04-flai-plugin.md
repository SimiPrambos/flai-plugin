# flai-plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `flai-plugin`, a Claude Code plugin that enforces Flutter project standards via 19 skills, 2 PostToolUse hooks, and a dart MCP server.

**Architecture:** Documentation-only repository (no Dart/Flutter source). All value lives in `skills/*/SKILL.md` markdown files and `hooks/scripts/*.sh` shell scripts. Structure mirrors `vgv-ai-flutter-plugin` at `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin`.

**Tech Stack:** Markdown, bash, Claude Code plugin format (`claude plugin validate`). Skills are YAML-frontmatter + markdown. Hooks are bash scripts reading JSON payloads via `jq`.

---

## File Map

```
flai-plugin/
├── .claude-plugin/plugin.json
├── .github/
│   └── workflows/
│       ├── ci.yaml
│       └── release_please.yaml
├── .gitignore
├── .mcp.json
├── .release-please-config.json
├── .release-please-manifest.json
├── CHANGELOG.md
├── CLAUDE.md
├── README.md
├── config/
│   └── cspell.json
├── hooks/
│   ├── hooks.json
│   └── scripts/
│       ├── analyze.sh
│       └── format.sh
└── skills/
    ├── architecture/SKILL.md
    ├── riverpod/SKILL.md
    ├── networking/SKILL.md
    ├── serialization/SKILL.md
    ├── error-handling/SKILL.md
    ├── local-storage/SKILL.md
    ├── environment-config/SKILL.md
    ├── logging/SKILL.md
    ├── connectivity/SKILL.md
    ├── code-generation/SKILL.md
    ├── navigation/SKILL.md
    ├── testing/SKILL.md
    ├── internationalization/SKILL.md
    ├── accessibility/SKILL.md
    ├── material-theming/SKILL.md
    ├── static-security/SKILL.md
    ├── ui-package/SKILL.md
    ├── license-compliance/SKILL.md
    └── sdk-upgrade/SKILL.md
```

---

## Task 1: Project Scaffold

**Files:**
- Create: `.gitignore`
- Create: `CHANGELOG.md`

- [ ] **Step 1: Create `.gitignore`**

```
.DS_Store
*.log
node_modules/
.env
```

- [ ] **Step 2: Create `CHANGELOG.md`**

```markdown
# Changelog

All notable changes to this project will be documented in this file.
```

- [ ] **Step 3: Commit**

```bash
git add .gitignore CHANGELOG.md
git commit -m "chore: add project scaffold"
```

---

## Task 2: Plugin Manifest, MCP, and cspell

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.mcp.json`
- Create: `config/cspell.json`

- [ ] **Step 1: Create `.claude-plugin/plugin.json`**

```json
{
  "name": "flai-plugin",
  "version": "0.1.0",
  "description": "Best-practice skills for Flutter development with Clean Architecture, Riverpod, and the flai standard tech stack — plus automated dart analyze and format hooks.",
  "license": "MIT",
  "keywords": [
    "accessibility",
    "analyze",
    "architecture",
    "clean-architecture",
    "code-generation",
    "connectivity",
    "dart",
    "dio",
    "environment-config",
    "error-handling",
    "feature-based",
    "flutter",
    "format",
    "fpdart",
    "freezed",
    "go-router",
    "hooks",
    "i18n",
    "internationalization",
    "isar",
    "json-serializable",
    "l10n",
    "license-compliance",
    "local-storage",
    "logging",
    "material-3",
    "mobile-security",
    "navigation",
    "networking",
    "retrofit",
    "riverpod",
    "sdk-upgrade",
    "secure-storage",
    "serialization",
    "shared-preferences",
    "socket-io",
    "talker",
    "testing",
    "theming",
    "typed-go-route",
    "ui-package",
    "wcag",
    "widget-testing"
  ]
}
```

- [ ] **Step 2: Create `.mcp.json`**

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": [
        "mcp-server"
      ]
    }
  }
}
```

- [ ] **Step 3: Create `config/cspell.json`**

```json
{
  "language": "en",
  "words": [
    "autoDispose",
    "Bidirectionality",
    "Bienvenido",
    "buildrunner",
    "bypassable",
    "codegeneration",
    "connectivity",
    "dartdoc",
    "CSPRNG",
    "dismissable",
    "elemento",
    "elementos",
    "envied",
    "fpdart",
    "freezed",
    "frontmatter",
    "GHSA",
    "goldens",
    "hoverable",
    "isar",
    "jailbroken",
    "json",
    "lerp",
    "LTRB",
    "mapbox",
    "microtasks",
    "mocktail",
    "monorepo",
    "Mundo",
    "notifier",
    "pasteable",
    "prefs",
    "pubspec",
    "retrofit",
    "riverpod",
    "serialization",
    "socketio",
    "stdio",
    "talker",
    "typedef",
    "WCAG",
    "widgetbook",
    "Widgetbook",
    "xxlg"
  ],
  "flagWords": []
}
```

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/plugin.json .mcp.json config/cspell.json
git commit -m "chore: add plugin manifest, MCP config, and cspell"
```

---

## Task 3: Hooks

**Files:**
- Create: `hooks/hooks.json`
- Create: `hooks/scripts/analyze.sh`
- Create: `hooks/scripts/format.sh`

- [ ] **Step 1: Create `hooks/hooks.json`**

```json
{
  "description": "flai-plugin hooks for Dart and Flutter development",
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/analyze.sh",
            "timeout": 30
          },
          {
            "type": "command",
            "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/format.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 2: Create `hooks/scripts/analyze.sh`**

```bash
#!/bin/bash
set -euo pipefail

# Read the hook payload from stdin
input=$(cat)

# Check jq availability
if ! command -v jq &>/dev/null; then
  echo "analyze hook: jq not found, skipping" >&2
  exit 0
fi

# Extract file path from the tool input
file_path=$(jq -r '.tool_input.file_path // empty' <<< "$input")

# Skip if no file path or not a Dart file
if [[ -z "$file_path" || "$file_path" != *.dart ]]; then
  exit 0
fi

# Run dart analyze on the single file
output=$(dart analyze "$file_path" 2>&1) || {
  echo "$output" >&2
  exit 2
}
```

- [ ] **Step 3: Create `hooks/scripts/format.sh`**

```bash
#!/bin/bash
set -euo pipefail

# Read the hook payload from stdin
input=$(cat)

# Check jq availability
if ! command -v jq &>/dev/null; then
  echo "format hook: jq not found, skipping" >&2
  exit 0
fi

# Extract file path from the tool input
file_path=$(jq -r '.tool_input.file_path // empty' <<< "$input")

# Skip if no file path or not a Dart file
if [[ -z "$file_path" || "$file_path" != *.dart ]]; then
  exit 0
fi

# Run dart format on the single file (auto-fix, always exit 0)
dart format "$file_path" &>/dev/null || true
```

- [ ] **Step 4: Make scripts executable**

```bash
chmod +x hooks/scripts/analyze.sh hooks/scripts/format.sh
```

- [ ] **Step 5: Commit**

```bash
git add hooks/
git commit -m "feat: add PostToolUse hooks for dart analyze and format"
```

---

## Task 4: CI/CD Workflows

**Files:**
- Create: `.github/workflows/ci.yaml`
- Create: `.github/workflows/release_please.yaml`
- Create: `.release-please-manifest.json`
- Create: `.release-please-config.json`

- [ ] **Step 1: Create `.github/workflows/ci.yaml`**

```yaml
name: ci

on:
  pull_request:
    branches:
      - main

jobs:
  markdown:
    name: 📝 Markdown Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint Markdown
        uses: DavidAnson/markdownlint-cli2-action@v19
        with:
          globs: |
            **/*.md
            !CHANGELOG.md
  spelling:
    name: ✍️ Spelling Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Spell Check Code
        uses: streetsidesoftware/cspell-action@v6
        with:
          files: |
            **/*.md
            !CHANGELOG.md
          config: 'config/cspell.json'
  detect-changed-skills:
    name: 🔎 Detect Changed Skills
    runs-on: ubuntu-latest
    outputs:
      skills: ${{ steps.changed-skills.outputs.skills }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Get changed skills
        id: changed-skills
        run: |
          BASE=${{ github.event.pull_request.base.sha }}
          HEAD=${{ github.event.pull_request.head.sha }}
          SKILLS=$(git diff --name-only $BASE $HEAD | \
            { grep 'SKILL.md' || true; } | \
            xargs -I {} dirname {} | \
            sort -u | \
            jq -R -s -c 'split("\n") | map(select(length > 0))')
          echo "skills=$SKILLS" >> "$GITHUB_OUTPUT"
          echo "Changed skills: $SKILLS"
  validate-skills:
    name: 🔍 Validate Skills
    needs: detect-changed-skills
    if: needs.detect-changed-skills.outputs.skills != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        skill: ${{ fromJson(needs.detect-changed-skills.outputs.skills) }}
    steps:
      - uses: actions/checkout@v4
      - name: Validate ${{ matrix.skill }}
        id: validate
        uses: Flash-Brew-Digital/validate-skill@v1
        with:
          path: ${{ matrix.skill }}
          validate-references: 'true'
          fail-on-warning: 'true'
          ignore-rules: 'name-match-directory,unknown-field'
      - name: Check results
        if: always()
        run: |
          echo "Valid: ${{ steps.validate.outputs.valid }}"
          echo "Errors: ${{ steps.validate.outputs.errors }}"
          echo "Warnings: ${{ steps.validate.outputs.warnings }}"
  plugin-validate:
    name: 📦 Plugin Validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Validate Plugin
        run: claude plugin validate .
```

- [ ] **Step 2: Create `.github/workflows/release_please.yaml`**

```yaml
name: release_please

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  create_release_pr:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v5
        with:
          token: ${{ secrets.RELEASE_PLEASE_TOKEN }}
          manifest-file: ".release-please-manifest.json"
          config-file: ".release-please-config.json"
```

- [ ] **Step 3: Create `.release-please-manifest.json`**

```json
{
  ".": "0.1.0"
}
```

- [ ] **Step 4: Create `.release-please-config.json`**

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "changelog-sections": [
    { "type": "feat", "section": "Features" },
    { "type": "fix", "section": "Bug Fixes" },
    { "type": "refactor", "section": "Refactors" },
    { "type": "chore", "section": "Miscellaneous Chores" },
    { "type": "docs", "section": "Docs" }
  ],
  "pull-request-header": ":rotating_light: There are changes ready for release :rocket:\n\nℹ Merge this PR once the team confirms the release is ready.\n",
  "pull-request-title-pattern": "chore: ${version}",
  "include-component-in-tag": false,
  "bump-minor-pre-major": true,
  "bump-patch-for-minor-pre-major": true,
  "packages": {
    ".": {
      "release-type": "simple",
      "changelog-path": "CHANGELOG.md",
      "extra-files": [
        {
          "type": "json",
          "path": ".claude-plugin/plugin.json",
          "jsonpath": "$.version"
        }
      ]
    }
  },
  "exclude-paths": [
    ".github",
    ".release-please-manifest.json",
    ".release-please-config.json",
    "CONTRIBUTING.md"
  ]
}
```

- [ ] **Step 5: Commit**

```bash
git add .github/ .release-please-manifest.json .release-please-config.json
git commit -m "ci: add CI workflow and release-please configuration"
```

---

## Task 5: CLAUDE.md and README.md

**Files:**
- Create: `CLAUDE.md`
- Create: `README.md`

- [ ] **Step 1: Create `CLAUDE.md`**

```markdown
# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

flai-plugin is a Claude Code plugin that enforces Flutter project standards using a Clean Architecture + Riverpod tech stack. It is a **documentation-only repository** — there is no Dart/Flutter source code, no `pubspec.yaml`, and no tests. All value lives in the markdown skill files.

## Repository Structure

```text
.mcp.json                # MCP server configuration (Dart)
.claude-plugin/
  plugin.json          # Plugin manifest (name, version, tags)
hooks/
  hooks.json           # Hook definitions (PostToolUse)
  scripts/
    analyze.sh         # Runs dart analyze on modified .dart files
    format.sh          # Runs dart format on modified .dart files
skills/
  architecture/SKILL.md
  riverpod/SKILL.md
  networking/SKILL.md
  serialization/SKILL.md
  error-handling/SKILL.md
  local-storage/SKILL.md
  environment-config/SKILL.md
  logging/SKILL.md
  connectivity/SKILL.md
  code-generation/SKILL.md
  navigation/SKILL.md
  testing/SKILL.md
  internationalization/SKILL.md
  accessibility/SKILL.md
  material-theming/SKILL.md
  static-security/SKILL.md
  ui-package/SKILL.md
  license-compliance/SKILL.md
  sdk-upgrade/SKILL.md
```

## Skill File Format

Every `SKILL.md` follows this structure:

1. **YAML frontmatter** with fields:
   - `name` _(required)_ — prefixed with `flai-`, lowercase letters, numbers, and hyphens only
   - `description` _(required)_ — when the skill should be triggered
   - `allowed-tools` _(optional)_ — space-separated tool list
   - `model` _(optional)_ — `haiku` or `sonnet`
   - `effort` _(optional)_ — `low`, `medium`, or `high`
2. **H1 title** — human-readable skill name
3. **Core Standards** — enforced directives, always first
4. **Content sections** — architecture, code examples, workflows, anti-patterns

## Writing Conventions

- Frame standards as clear directives — no soft language ("consider", "prefer")
- Use fenced code blocks with language identifiers for all examples
- Provide complete, copy-pasteable snippets, not fragments
- Reference packages by full name (e.g., `package:riverpod`)
- Include anti-patterns alongside correct patterns when helpful
- Align pipe characters vertically in all markdown tables

## Adding a New Skill

1. Create `skills/<skill_name>/SKILL.md` following the format above
2. Update keywords in `.claude-plugin/plugin.json`
3. Update the skills table in `README.md`
4. Update the repository structure in `CLAUDE.md`

## Hooks

The `hooks/` directory contains PostToolUse hooks defined in `hooks.json`.

### PostToolUse Hooks

- `Edit|Write` matcher → `analyze.sh` — runs `dart analyze` on the modified `.dart` file; exits 2 on failure (blocking)
- `Edit|Write` matcher → `format.sh` — runs `dart format` on the modified `.dart` file; always exits 0 (non-blocking)

Both scripts require **jq** to parse the hook payload (they skip gracefully if `jq` is not installed).

## Commits

Use conventional commits: `type(scope): description`

Examples: `feat: add riverpod skill`, `fix: correct networking skill example`, `docs: update README`
```

- [ ] **Step 2: Create `README.md`**

```markdown
# flai-plugin

A [Claude Code][claude_code_link] plugin that enforces Flutter project standards with Clean Architecture, Riverpod state management, and the flai standard tech stack.

[claude_code_link]: https://claude.ai/code

## Overview

flai-plugin is a collection of contextual best-practices skills that Claude uses when helping you write Flutter and Dart code. Each skill provides opinionated guidance covering architecture patterns, naming conventions, folder structures, code examples, testing strategies, and anti-patterns to avoid.

## Installation

```bash
claude plugin install /path/to/flai-plugin
```

## Skills

| Skill | Description |
| ----- | ----------- |
| [**Architecture**](skills/architecture/SKILL.md) | Clean Architecture (Feature-Based) — `core/` and `features/` structure, layer boundaries, dependency rules, and monorepo scaling |
| [**Riverpod**](skills/riverpod/SKILL.md) | State management and DI with Riverpod — `@riverpod` code generation, `AsyncNotifier`/`Notifier` patterns, and provider organization |
| [**Networking**](skills/networking/SKILL.md) | HTTP with Dio and Retrofit, real-time with Socket.IO — singleton Dio setup, connectivity-aware interceptors, and StreamProvider for WebSocket |
| [**Serialization**](skills/serialization/SKILL.md) | Data models with Freezed and json_serializable — immutable models, union types, JSON parsing, and analyzer version pinning |
| [**Error Handling**](skills/error-handling/SKILL.md) | Functional error handling with fpdart — `Either` propagation through layers, sealed `Failure` hierarchy, and `TaskEither` for complex chains |
| [**Local Storage**](skills/local-storage/SKILL.md) | Three-tier storage — SharedPreferences for prefs, FlutterSecureStorage for secrets, Isar for complex entities with reactive queries |
| [**Environment Config**](skills/environment-config/SKILL.md) | Secrets and configuration with envied — `obfuscate: true`, `.env.example`, and `--dart-define` for CI/CD releases |
| [**Logging**](skills/logging/SKILL.md) | Structured logging with Talker — Dio logger, Riverpod observer, route observer, and log level conventions |
| [**Connectivity**](skills/connectivity/SKILL.md) | Network status with connectivity_plus — `StreamProvider`, fail-fast Dio interceptor, and offline handling patterns |
| [**Code Generation**](skills/code-generation/SKILL.md) | build_runner — `build.yaml` path restriction, analyzer version pinning, generated files policy, and common conflict fixes |
| [**Navigation**](skills/navigation/SKILL.md) | GoRouter routing — `@TypedGoRoute` type-safe routes, deep linking, redirects, and `TalkerRouteObserver` |
| [**Testing**](skills/testing/SKILL.md) | Unit, widget, and golden tests — `ProviderContainer` overrides, `mocktail` mocking, Either test patterns, and `pumpApp` helpers |
| [**Internationalization**](skills/internationalization/SKILL.md) | i18n/l10n — ARB files, `context.l10n` patterns, pluralization, and RTL/directional layout |
| [**Accessibility**](skills/accessibility/SKILL.md) | WCAG 2.1 AA — semantics, screen reader support, touch targets, focus management, color contrast, and motion sensitivity |
| [**Material Theming**](skills/material-theming/SKILL.md) | Material 3 theming — `ColorScheme`, `TextTheme`, component themes, spacing systems, and light/dark mode |
| [**Security**](skills/static-security/SKILL.md) | Flutter-specific static security — secrets management, `flutter_secure_storage`, certificate pinning, and OWASP Mobile Top 10 |
| [**UI Package**](skills/ui-package/SKILL.md) | Reusable Flutter UI packages — custom widget libraries with `ThemeExtension` theming, barrel exports, and widget tests |
| [**License Compliance**](skills/license-compliance/SKILL.md) | Dependency license auditing — permissive, copyleft, and unknown license categorization |
| [**SDK Upgrade**](skills/sdk-upgrade/SKILL.md) | Dart and Flutter SDK version bumps — CI wildcard pinning, pubspec exact pinning, and upgrade workflow |

## Hooks

| Hook | Behavior |
| ---- | -------- |
| **Analyze** | Runs `dart analyze` on modified `.dart` files after every Edit/Write; blocking on failure |
| **Format** | Runs `dart format` on modified `.dart` files after every Edit/Write; non-blocking |

### Prerequisites

- **Dart SDK** on your `PATH`
- **jq** for hook payload parsing (hooks skip gracefully if missing)
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md README.md
git commit -m "docs: add CLAUDE.md and README.md"
```

---

## Task 6: flai-architecture Skill

**Files:**
- Create: `skills/architecture/SKILL.md`

- [ ] **Step 1: Create `skills/architecture/SKILL.md`**

```markdown
---
name: flai-architecture
description: Clean Architecture (Feature-Based) structure for Flutter. Use when creating new features, setting up project structure, organizing imports between layers, reviewing folder layout, or scaffolding a new Flutter project.
allowed-tools: Read Glob Grep
model: sonnet
effort: high
---

# Clean Architecture (Feature-Based)

Feature-based Clean Architecture for Flutter — each feature owns its data, domain, and presentation layers. A shared `core/` layer holds global configuration and cross-cutting concerns.

## Core Standards

Apply these standards to ALL architectural work:

- **`lib/core/` for global concerns** — dio config, isar init, base failure classes, extensions, constants, and global providers live here; nothing feature-specific
- **`lib/features/<name>/` for each feature** — every feature has its own `data/`, `domain/`, and `presentation/` subdirectories
- **No cross-feature imports** — features never import directly from other features; all shared data flows through the domain layer (repository abstractions)
- **Domain layer is pure Dart** — no Flutter or package imports in `domain/`; only Dart SDK and fpdart
- **Feature-level `providers.dart`** — each feature exposes its Riverpod providers from a single file for discoverability
- **No `utils/` or `common/` catch-all folders** — every file belongs to a specific layer in a specific feature or to `core/`
- **Dependency direction is inward** — `presentation` depends on `domain`; `data` depends on `domain`; `domain` depends on nothing

## Directory Structure

```text
lib/
├── core/
│   ├── config/
│   │   └── env.dart              # envied environment config
│   ├── database/
│   │   └── isar_provider.dart    # single Isar instance provider
│   ├── error/
│   │   └── failures.dart         # base Failure sealed class
│   ├── extensions/               # Dart/Flutter extension methods
│   ├── http/
│   │   └── dio_provider.dart     # singleton Dio provider
│   ├── l10n/                     # generated localization files
│   ├── router/
│   │   └── router.dart           # go_router configuration
│   └── providers.dart            # global providers (talker, connectivity)
│
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/
│   │   │   │   ├── auth_remote_datasource.dart
│   │   │   │   └── auth_local_datasource.dart
│   │   │   ├── models/
│   │   │   │   └── user_model.dart          # @freezed + @JsonSerializable
│   │   │   └── repositories/
│   │   │       └── auth_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   └── user.dart                # @freezed, no JSON
│   │   │   ├── failures/
│   │   │   │   └── auth_failure.dart        # sealed AuthFailure extends Failure
│   │   │   └── repositories/
│   │   │       └── auth_repository.dart     # abstract interface
│   │   ├── presentation/
│   │   │   ├── notifiers/
│   │   │   │   └── auth_notifier.dart       # @riverpod AsyncNotifier
│   │   │   ├── pages/
│   │   │   │   └── login_page.dart
│   │   │   └── widgets/
│   │   └── providers.dart                   # feature provider exports
│   │
│   └── <other_features>/
│
└── main.dart
```

## Layer Responsibilities

### Data Layer

Owns all I/O. Communicates with APIs, databases, and device storage. Converts raw data (JSON, Isar objects) into domain entities.

- `datasources/` — remote (Retrofit/Dio) and local (Isar/SharedPreferences/SecureStorage) data sources
- `models/` — `@freezed` classes with `@JsonSerializable` for JSON serialization
- `repositories/` — implements the domain repository abstraction; wraps data source calls in `Either<Failure, T>`

### Domain Layer

Pure business logic. No Flutter or external package dependencies (except `fpdart`).

- `entities/` — `@freezed` classes without JSON annotations
- `failures/` — sealed failure subclasses specific to this feature
- `repositories/` — abstract repository interfaces returning `Future<Either<Failure, T>>`

### Presentation Layer

UI and state. Consumes domain repositories via Riverpod.

- `notifiers/` — `@riverpod` `AsyncNotifier<T>` or `Notifier<T>` classes; convert `Either` results to `AsyncValue`
- `pages/` — full-screen widgets registered as GoRouter routes
- `widgets/` — reusable widgets scoped to this feature

## core/ Contents

| File/Directory | Purpose |
| --- | --- |
| `core/config/env.dart` | envied `Env` class with all environment variables |
| `core/database/isar_provider.dart` | Single `FutureProvider<Isar>` initialized once |
| `core/error/failures.dart` | Sealed `Failure` base class |
| `core/http/dio_provider.dart` | Singleton `dioProvider` with all interceptors attached |
| `core/router/router.dart` | GoRouter instance with all top-level `@TypedGoRoute` definitions |
| `core/providers.dart` | `talkerProvider`, `connectivityProvider`, and other global providers |

## Monorepo Scaling

When the project grows beyond a single app, move `core/` and each `features/<name>/` into separate Dart packages under a `packages/` directory. The internal `data/domain/presentation` structure stays identical — only the package boundaries change.

```text
packages/
├── core/                    # shared utilities and config
├── feature_auth/            # auth feature package
│   ├── lib/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── pubspec.yaml
└── feature_home/
apps/
└── flutter_app/             # thin shell, wires packages together
```

Use path dependencies during development:

```yaml
dependencies:
  feature_auth:
    path: ../../packages/feature_auth
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| --- | --- | --- |
| `features/auth/` importing from `features/profile/` | Creates circular dependencies; breaks feature isolation | Move shared data to a domain entity and have both features depend on the same repository abstraction |
| Flutter imports in `domain/` | Domain becomes tied to the Flutter SDK; can't test without Flutter | Domain uses only Dart SDK and `fpdart` |
| Single `repositories/` folder at root | Loses feature boundaries; hard to navigate as the codebase grows | Each feature owns its own repository implementation |
| `utils/` folder at root | Becomes a catch-all; obscures responsibility | Name folders by specific purpose: `extensions/`, `constants/`, `formatters/` |
| Business logic in presentation notifiers | Duplicates logic; hard to test without UI | Move logic to domain use cases or repository methods |
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add skills/architecture/
git commit -m "feat: add flai-architecture skill"
```

---

## Task 7: flai-riverpod Skill

**Files:**
- Create: `skills/riverpod/SKILL.md`

- [ ] **Step 1: Create `skills/riverpod/SKILL.md`**

```markdown
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

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add skills/riverpod/
git commit -m "feat: add flai-riverpod skill"
```

---

## Task 8: flai-networking Skill

**Files:**
- Create: `skills/networking/SKILL.md`

- [ ] **Step 1: Create `skills/networking/SKILL.md`**

```markdown
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

part 'dio_provider.g.dart';

@Riverpod(keepAlive: true)
Dio dio(DioRef ref) {
  final talker = ref.watch(talkerProvider);
  final dio = Dio(
    BaseOptions(
      baseUrl: Env.apiBaseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
    ),
  );
  dio.interceptors.addAll([
    TalkerDioLogger(
      talker: talker,
      settings: const TalkerDioLoggerSettings(
        printRequestHeaders: false,
        printResponseMessage: true,
      ),
    ),
    ref.watch(connectivityInterceptorProvider),
  ]);
  return dio;
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
  return ConnectivityInterceptor(ref.watch(connectivityProvider.notifier));
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/networking/
git commit -m "feat: add flai-networking skill"
```

---

## Task 9: flai-serialization Skill

**Files:**
- Create: `skills/serialization/SKILL.md`

- [ ] **Step 1: Create `skills/serialization/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/serialization/
git commit -m "feat: add flai-serialization skill"
```

---

## Task 10: flai-error-handling Skill

**Files:**
- Create: `skills/error-handling/SKILL.md`

- [ ] **Step 1: Create `skills/error-handling/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/error-handling/
git commit -m "feat: add flai-error-handling skill"
```

---

## Task 11: flai-local-storage Skill

**Files:**
- Create: `skills/local-storage/SKILL.md`

- [ ] **Step 1: Create `skills/local-storage/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/local-storage/
git commit -m "feat: add flai-local-storage skill"
```

---

## Task 12: flai-environment-config Skill

**Files:**
- Create: `skills/environment-config/SKILL.md`

- [ ] **Step 1: Create `skills/environment-config/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/environment-config/
git commit -m "feat: add flai-environment-config skill"
```

---

## Task 13: flai-logging Skill

**Files:**
- Create: `skills/logging/SKILL.md`

- [ ] **Step 1: Create `skills/logging/SKILL.md`**

```markdown
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

```dart
// lib/main.dart
void main() async {
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
            printProviderDisposed: false,  // reduce noise
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/logging/
git commit -m "feat: add flai-logging skill"
```

---

## Task 14: flai-connectivity Skill

**Files:**
- Create: `skills/connectivity/SKILL.md`

- [ ] **Step 1: Create `skills/connectivity/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/connectivity/
git commit -m "feat: add flai-connectivity skill"
```

---

## Task 15: flai-code-generation Skill

**Files:**
- Create: `skills/code-generation/SKILL.md`

- [ ] **Step 1: Create `skills/code-generation/SKILL.md`**

```markdown
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
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/code-generation/
git commit -m "feat: add flai-code-generation skill"
```

---

## Task 16: flai-navigation Skill (Adapted from VGV)

**Files:**
- Create: `skills/navigation/SKILL.md`

Source: `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/navigation/SKILL.md`

- [ ] **Step 1: Copy and adapt the VGV navigation skill**

Copy the VGV navigation SKILL.md verbatim, then apply these changes:

1. Change frontmatter `name: vgv-navigation` → `name: flai-navigation`
2. Remove `model: sonnet` line (keep as default)
3. Remove the VGV-specific intro paragraph and replace with a neutral one
4. In the **Common Patterns → Adding a New Route** section, add step 5.5 after "Run build_runner":

```markdown
5.5. Wire `TalkerRouteObserver` to the router if not already done (see Logging section)
```

5. Add a new **Logging Integration** section before **Quick Reference**:

```markdown
## Logging Integration

Attach `TalkerRouteObserver` to GoRouter so all navigation events are logged automatically:

```dart
// core/router/router.dart
import 'package:go_router/go_router.dart';
import 'package:talker_flutter/talker_flutter.dart';

GoRouter buildRouter(Talker talker) => GoRouter(
  observers: [TalkerRouteObserver(talker)],
  routes: $appRoutes,
  // redirects, initialLocation, etc.
);
```

Provide via Riverpod:

```dart
@Riverpod(keepAlive: true)
GoRouter router(RouterRef ref) {
  final talker = ref.watch(talkerProvider);
  return buildRouter(talker);
}
```
```

6. Add `talker_flutter` to the Quick Reference packages table:

```markdown
| `talker_flutter` | `TalkerRouteObserver` for navigation logging |
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/navigation/
git commit -m "feat: add flai-navigation skill (adapted from VGV)"
```

---

## Task 17: flai-testing Skill (Adapted from VGV)

**Files:**
- Create: `skills/testing/SKILL.md`

Source: `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/testing/SKILL.md`

- [ ] **Step 1: Copy and adapt the VGV testing skill**

Copy the VGV testing SKILL.md verbatim, then apply these changes:

1. Change frontmatter `name: vgv-testing` → `name: flai-testing`
2. Change `description` to remove VGV branding — replace `package:bloc_test` with `package:riverpod_test`
3. Remove `mcp__very-good-cli__test` from `allowed-tools`
4. Update the description:

```yaml
description: Best practices for Dart unit tests, Flutter widget tests, and golden file tests. Use when writing, modifying, or reviewing tests that use package:test, package:flutter_test, package:mocktail, or package:riverpod_test.
```

5. Remove the **Core Standards** bullet: `"Use package:bloc_test..."` and `"Mock Blocs and Cubits..."` from Widget Testing Standards table.

6. Replace the Widget Testing Standards table row `"Mock Blocs and Cubits — Use MockBloc/MockCubit from package:bloc_test"` with:

```markdown
| **Override Riverpod providers** | Use `ProviderContainer` overrides in unit tests; use `ProviderScope` overrides in widget tests |
```

7. Add a new **Testing Riverpod Providers** section after the **Mocking with Mocktail** section:

```markdown
## Testing Riverpod Providers

Test `AsyncNotifier` and `Notifier` classes by overriding their dependencies in a `ProviderContainer`. Never mock the notifier itself — mock its dependencies.

### Unit Testing a Notifier

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:riverpod/riverpod.dart';

class _MockSongRepository extends Mock implements SongRepository {}

void main() {
  group(SongsNotifier, () {
    late SongRepository repository;
    late ProviderContainer container;

    setUp(() {
      repository = _MockSongRepository();
      container = ProviderContainer(
        overrides: [
          songRepositoryProvider.overrideWithValue(repository),
        ],
      );
    });

    tearDown(container.dispose);

    group('build', () {
      test('returns songs when repository succeeds', () async {
        when(() => repository.getSongs()).thenAnswer(
          (_) async => right([Song(id: '1', title: 'Test')]),
        );

        final result = await container.read(songsNotifierProvider.future);

        expect(result, hasLength(1));
        expect(result.first.title, equals('Test'));
      });

      test('throws $SongFailure when repository fails', () async {
        when(() => repository.getSongs()).thenAnswer(
          (_) async => left(const SongFailure.notFound()),
        );

        expect(
          container.read(songsNotifierProvider.future),
          throwsA(isA<SongFailure>()),
        );
      });
    });
  });
}
```

### Widget Testing with ProviderScope Overrides

```dart
testWidgets('renders song list when loaded', (tester) async {
  final mockRepo = _MockSongRepository();
  when(() => mockRepo.getSongs()).thenAnswer(
    (_) async => right([Song(id: '1', title: 'Test Song')]),
  );

  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        songRepositoryProvider.overrideWithValue(mockRepo),
      ],
      child: const MaterialApp(home: SongsPage()),
    ),
  );
  await tester.pumpAndSettle();

  expect(find.text('Test Song'), findsOneWidget);
});
```

### Mocking Either Results

```dart
// Success
when(() => mock.getSongs()).thenAnswer(
  (_) async => right([Song(id: '1', title: 'Test')]),
);

// Failure
when(() => mock.getSongs()).thenAnswer(
  (_) async => left(const SongFailure.notFound()),
);
```

### riverpod_lint Note

`riverpod_lint` provides useful analyzer warnings for common Riverpod mistakes, but it is **incompatible with `flutter_test`** in the same project. If you want lint rules, add `riverpod_lint` and run `dart analyze` separately from tests. Do not add `riverpod_lint` if it causes `flutter test` to fail.
```

8. Update the **Additional Resources** section — remove `references/widget-tests.md` BLoC-specific references.

9. Update the `pumpApp` helper to include a `ProviderScope` wrapper:

```dart
// test/helpers/pump_app.dart
extension PumpApp on WidgetTester {
  Future<void> pumpApp(
    Widget widget, {
    List<Override> overrides = const [],
  }) {
    return pumpWidget(
      ProviderScope(
        overrides: overrides,
        child: MaterialApp(
          home: widget,
        ),
      ),
    );
  }
}
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/testing/
git commit -m "feat: add flai-testing skill (adapted from VGV)"
```

---

## Task 18: Rebrand-Only Adapted Skills

These four skills are copied from VGV with only branding changes (name prefix, remove VGV links/mentions).

**Files:**
- Create: `skills/internationalization/SKILL.md`
- Create: `skills/accessibility/SKILL.md`
- Create: `skills/material-theming/SKILL.md`
- Create: `skills/ui-package/SKILL.md`

Sources:
- `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/internationalization/SKILL.md`
- `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/accessibility/SKILL.md`
- `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/material-theming/SKILL.md`
- `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/ui-package/SKILL.md`

For each skill, also copy its `references/` subdirectory if one exists.

- [ ] **Step 1: Copy and rebrand internationalization**

Copy `vgv-ai-flutter-plugin/skills/internationalization/` to `skills/internationalization/`.

Change in `SKILL.md`:
- `name: vgv-internationalization` → `name: flai-internationalization`
- Remove any mentions of "Very Good Ventures" or "VGV"

Copy `references/` subdirectory if it exists.

- [ ] **Step 2: Copy and rebrand accessibility**

Copy `vgv-ai-flutter-plugin/skills/accessibility/` to `skills/accessibility/`.

Change in `SKILL.md`:
- `name: vgv-accessibility` → `name: flai-accessibility`
- Remove any mentions of "Very Good Ventures" or "VGV"

Copy `references/` subdirectory if it exists.

- [ ] **Step 3: Copy and rebrand material-theming**

Copy `vgv-ai-flutter-plugin/skills/material-theming/` to `skills/material-theming/`.

Change in `SKILL.md`:
- `name: vgv-material-theming` → `name: flai-material-theming`
- Remove any mentions of "Very Good Ventures" or "VGV"

Copy `references/` subdirectory if it exists.

- [ ] **Step 4: Copy and rebrand ui-package**

Copy `vgv-ai-flutter-plugin/skills/ui-package/` to `skills/ui-package/`.

Change in `SKILL.md`:
- `name: vgv-ui-package` → `name: flai-ui-package`
- Remove any mentions of "Very Good Ventures" or "VGV"
- Remove any references to `mcp__very-good-cli__create`; replace with `flutter create --template=package`
- Remove `allowed-tools` entry for VGV MCP tools if present

Copy `references/` subdirectory if it exists.

- [ ] **Step 5: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 6: Commit**

```bash
git add skills/internationalization/ skills/accessibility/ skills/material-theming/ skills/ui-package/
git commit -m "feat: add rebrand-only adapted skills (i18n, accessibility, theming, ui-package)"
```

---

## Task 19: flai-static-security Skill (Adapted from VGV)

**Files:**
- Create: `skills/static-security/SKILL.md`

Source: `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/static-security/SKILL.md`

- [ ] **Step 1: Copy and adapt the VGV static-security skill**

Copy the VGV static-security SKILL.md verbatim (including `references/` subdirectory), then apply these changes:

1. Change frontmatter `name: vgv-static-security` → `name: flai-static-security`
2. Remove `mcp__very-good-cli__packages_check_licenses` from `allowed-tools`
3. Remove any mentions of "Very Good Ventures" or VGV security guides
4. In the **Secure Data Storage** section, replace the storage guidance with the three-tier pattern:

Replace the existing storage explanation with:

```markdown
## Secure Data Storage

Use the three-tier storage strategy:

- **`SharedPreferences`** — non-sensitive key-value data only (theme, locale, onboarding flags)
- **`package:flutter_secure_storage`** — auth tokens, refresh tokens, API keys, passwords; backed by iOS Keychain and Android Keystore
- **`Isar`** — complex entity data; does not encrypt by default (do not store raw secrets)

```dart
// ❌ JWT in SharedPreferences — plaintext, unencrypted
final prefs = await SharedPreferences.getInstance();
prefs.setString('auth_token', jwt);

// ✅ JWT in FlutterSecureStorage — hardware-backed encryption
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
);
await storage.write(key: 'auth_token', value: jwt);
```

Never store passwords, session tokens, PII, or private keys in SharedPreferences or Isar without explicit encryption.
```

5. Update the **Secrets & API Keys** section to reference envied:

After the "Never commit .env files" paragraph, add:

```markdown
Use `package:envied` with `obfuscate: true` for compile-time environment configuration. See the `flai-environment-config` skill for setup. Note: even obfuscated values in the binary are not equivalent to backend-served secrets — use backend services for production API keys.
```

- [ ] **Step 2: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 3: Commit**

```bash
git add skills/static-security/
git commit -m "feat: add flai-static-security skill (adapted from VGV)"
```

---

## Task 20: flai-license-compliance and flai-sdk-upgrade Skills (Adapted from VGV)

**Files:**
- Create: `skills/license-compliance/SKILL.md`
- Create: `skills/sdk-upgrade/SKILL.md`

Sources:
- `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/license-compliance/SKILL.md`
- `/Users/silmi/workspace/devtools/flai-plugin/vgv-ai-flutter-plugin/skills/dart-flutter-sdk-upgrade/SKILL.md`

- [ ] **Step 1: Copy and adapt license-compliance**

Copy the VGV license-compliance SKILL.md, then apply:

1. `name: vgv-license-compliance` → `name: flai-license-compliance`
2. Remove `mcp__very-good-cli__packages_check_licenses` from `allowed-tools`; add `Bash`
3. Replace Very Good CLI license check command with:

```markdown
Run `dart pub deps --json | dart run dependency_validator` or manually inspect `pubspec.lock` for license metadata. For a structured audit, use `dart pub global activate pana` and run `pana .` to get package metadata including licenses.

Alternatively, use the `pub.dev` API or check each package's `pubspec.yaml` for a `license` field manually.
```

4. Remove any mentions of "Very Good Ventures" or VGV.

- [ ] **Step 2: Copy and adapt sdk-upgrade**

Copy the VGV dart-flutter-sdk-upgrade SKILL.md, then apply:

1. `name: vgv-dart-flutter-sdk-upgrade` → `name: flai-sdk-upgrade`
2. Keep SDK versioning guidance: CI uses `^MAJOR.MINOR.x`, pubspec uses `^MAJOR.MINOR.PATCH`
3. Replace VGV CI workflow references with generic GitHub Actions Flutter setup:

Replace VGV CI-specific steps with:

```markdown
### Updating CI Workflows

In your GitHub Actions workflow, update the Flutter version:

```yaml
- uses: subosito/flutter-action@v2
  with:
    flutter-version: '^3.24.x'  # CI uses wildcard patch
    channel: stable
```

### Updating pubspec.yaml

```yaml
environment:
  sdk: ^3.5.0         # Dart SDK — exact minor, wildcard patch not standard
  flutter: ^3.24.0    # Flutter SDK constraint
```
```

4. Remove any mentions of "Very Good Ventures" or VGV.

- [ ] **Step 3: Validate**

```bash
claude plugin validate .
```

- [ ] **Step 4: Commit**

```bash
git add skills/license-compliance/ skills/sdk-upgrade/
git commit -m "feat: add flai-license-compliance and flai-sdk-upgrade skills (adapted from VGV)"
```

---

## Task 21: Final Validation

- [ ] **Step 1: Run full plugin validation**

```bash
claude plugin validate .
```

Expected output: `Plugin is valid` with no errors or warnings.

- [ ] **Step 2: Verify all 19 skill directories exist**

```bash
ls skills/
```

Expected:
```
accessibility     code-generation   error-handling    local-storage     navigation        sdk-upgrade
architecture      connectivity      internationalization  logging       riverpod          serialization
environment-config  licensing-compliance material-theming  networking    static-security   testing
                                                                         ui-package
```

- [ ] **Step 3: Verify git history uses conventional commits**

```bash
git log --oneline
```

All commits should follow `type: description` format.

- [ ] **Step 4: Verify hooks are executable**

```bash
ls -la hooks/scripts/
```

Both `.sh` files should have `x` permission.

- [ ] **Step 5: Commit any remaining fixes**

If any issues were found and fixed in steps 1-4:

```bash
git add -A
git commit -m "fix: resolve plugin validation issues"
```
