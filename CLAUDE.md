# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

flai-plugin is a Claude Code plugin that enforces Flutter project standards using a Clean Architecture + Riverpod tech stack. It is a **documentation-only repository** — there is no Dart/Flutter source code, no `pubspec.yaml`, and no tests. All value lives in the markdown skill files.

## Repository Structure

```
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
