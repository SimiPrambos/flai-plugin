# VGV Migration Skill & Starter Template Design

**Date:** 2026-05-05  
**Status:** Approved  
**Author:** Claude Opus 4.7

## Problem Statement

The team needs a standardized Flutter boilerplate that enforces flai standards (Clean Architecture + Riverpod + flai tech stack) while preventing junior developers from breaking conventions through AI-assisted development. Current pain points:

- Manual project setup leads to inconsistent architecture
- Developers prompt AI without following standards
- No enforcement mechanism for flai conventions
- Mixed skill levels on the team require guardrails

## Goals

1. **Productivity boost** — developers start with correct structure, not blank canvas
2. **Standard enforcement** — make it hard to break flai conventions
3. **Leverage existing tools** — use VeryGood CLI for CI/CD, testing infrastructure
4. **Separation of concerns** — plugin (standards) separate from starter (implementation)

## Solution Overview

Two-repo strategy with three-layer enforcement:

### Repo 1: `flai-plugin` (this repo)
Documentation-only plugin with skills and hooks. **New addition:** `flai-vgv-migration` skill that guides VGV CLI output → flai standard transformation.

### Repo 2: `flai-starter` (new, separate repo)
Working Flutter application created by applying the migration skill to VGV CLI output. Contains CLAUDE.md that mandates flai-plugin usage.

### Enforcement Layers

1. **CLAUDE.md** (instruction-level) — loaded every conversation, mandates skill invocation
2. **Skills** (pattern-level) — 18+ skills with standards, examples, anti-patterns
3. **Hooks** (code-level) — `dart analyze` and `dart format` on every Edit/Write

## Architecture

### flai-plugin Changes

```
flai-plugin/
├── skills/
│   ├── vgv-migration/
│   │   └── SKILL.md          # NEW: VGV → flai migration guide
│   ├── architecture/SKILL.md
│   ├── riverpod/SKILL.md
│   └── ... (16 other skills)
├── .claude-plugin/
│   └── plugin.json           # UPDATED: add vgv-migration keywords
└── README.md                 # UPDATED: document migration skill
```

### flai-starter Structure (created separately)

```
flai-starter/
├── CLAUDE.md                 # Enforcement contract
├── lib/
│   ├── main.dart
│   ├── app/
│   │   ├── app.dart
│   │   └── app_router.dart
│   ├── core/
│   │   ├── config/env.dart
│   │   ├── database/isar_provider.dart
│   │   ├── error/failures.dart
│   │   ├── http/dio_provider.dart
│   │   └── providers.dart
│   └── features/
│       └── counter/          # Example feature (3-layer)
│           ├── data/
│           ├── domain/
│           ├── presentation/
│           └── providers.dart
├── test/
│   └── helpers/              # VGV test helpers (adapted for Riverpod)
├── .github/workflows/        # VGV CI/CD (preserved)
├── pubspec.yaml              # flai stack packages
├── build.yaml
├── analysis_options.yaml
└── .env.example
```

## Workflow

### One-Time: Building flai-starter (you do this)

1. Run `very_good create flutter_app flai_starter`
2. Install `flai-plugin` in Claude Code
3. Invoke `flai-vgv-migration` skill
4. Claude refactors VGV scaffold → flai standard
5. Validate: `flutter pub get`, `dart analyze`, `flutter test`, `flutter run`
6. Push as `flai-starter` repo

### Daily: Team Using flai-starter

1. Clone `flai-starter`
2. Install `flai-plugin` (if not already installed team-wide)
3. Rename package/app
4. `flutter pub get && flutter run`
5. CLAUDE.md automatically enforces plugin usage
6. Start building features with Claude guided by skills

## flai-vgv-migration Skill

### Purpose
Step-by-step guide for transforming VGV CLI output into flai standard.

### Key Sections

1. **Pre-Migration Checklist** — verify VGV scaffold structure
2. **Migration Steps:**
   - Update `pubspec.yaml` (remove Bloc, add flai stack)
   - Restructure `lib/` (create `core/`, `features/`)
   - Implement core layer (dio, isar, failures, env, router)
   - Rewrite counter feature (Bloc → Riverpod Clean Architecture)
   - Adapt test helpers for Riverpod
   - Generate CLAUDE.md from template
   - Preserve VGV infrastructure (CI/CD, analysis_options.yaml, l10n)
3. **Post-Migration Validation** — checklist to verify success

### CLAUDE.md Template (embedded in skill)

The skill includes a complete CLAUDE.md template as a code block. Key elements:

```markdown
# CLAUDE.md

## Required Plugin
Install flai-plugin before working on this project.

## Mandatory Skill Invocation
[Table mapping tasks → required skills for all 18 skills]

## Hard Rules
- Architecture: ALWAYS use core/ + features/, NEVER cross-feature imports
- State: ALWAYS Riverpod, NEVER Bloc/setState
- Errors: ALWAYS Either<Failure, T>, NEVER throw from repositories
- Packages: ALWAYS use pubspec.yaml packages, NEVER add without approval
- Quality: ALWAYS pass dart analyze, ALWAYS format, ALWAYS test

## Verification
Before claiming complete: invoke skills, generate code, analyze, format, test
```

## VGV → flai Transformation

### What VGV Generates

- Flat `lib/` structure (no `core/` or `features/`)
- Bloc/Cubit for state management
- Basic l10n setup
- Excellent CI/CD (GitHub Actions)
- Solid testing infrastructure
- Strong `analysis_options.yaml`

### What Gets Refactored

| VGV Output | flai Result |
|------------|-------------|
| Flat `lib/app/`, `lib/counter/` | `lib/core/` + `lib/features/counter/` |
| Bloc/Cubit | Riverpod `@riverpod` code generation |
| VGV error handling | fpdart `Either` + sealed `Failure` |
| http or none | Dio + Retrofit with interceptors |
| No storage strategy | SharedPreferences + SecureStorage + Isar |
| Basic l10n | Same l10n with `context.l10n` patterns |

### What Gets Preserved

- `.github/workflows/` — CI/CD pipelines
- `test/helpers/` — test utilities (adapted for Riverpod)
- `analysis_options.yaml` — VGV lint rules (supplemented)
- l10n setup — ARB files, configuration

## Package Changes

### Removed (VGV defaults)
- `bloc`
- `flutter_bloc`
- `bloc_test`

### Added (flai stack)
- **State:** `flutter_riverpod`, `riverpod_annotation`, `riverpod_generator`
- **Networking:** `dio`, `retrofit`, `retrofit_generator`
- **Serialization:** `freezed`, `freezed_annotation`, `json_annotation`, `json_serializable`
- **Functional:** `fpdart`
- **Storage:** `isar`, `isar_generator`, `flutter_secure_storage`, `shared_preferences`
- **Navigation:** `go_router`
- **Logging:** `talker`, `talker_dio_logger`, `talker_riverpod_logger`
- **Config:** `envied`, `envied_generator`
- **Connectivity:** `connectivity_plus`
- **Dev:** `build_runner`, `riverpod_lint`, `custom_lint`

## Example Feature: Counter

The counter feature demonstrates all three Clean Architecture layers:

### Domain Layer (pure Dart)
```
domain/
├── entities/counter_state.dart       # @freezed, no JSON
├── failures/counter_failure.dart     # sealed CounterFailure extends Failure
└── repositories/counter_repository.dart  # abstract interface
```

### Data Layer
```
data/
└── repositories/counter_repository_impl.dart  # implements domain interface
```

### Presentation Layer
```
presentation/
├── notifiers/counter_notifier.dart   # @riverpod AsyncNotifier
└── pages/counter_page.dart           # UI
```

Even though counter is trivial (no API, no database), it shows the full pattern developers copy for real features.

## Enforcement Mechanism

### How CLAUDE.md Enforces Standards

1. **Loaded automatically** — every conversation starts with CLAUDE.md in context
2. **Highest priority** — overrides Claude's default behavior
3. **Skill mapping** — removes "which skill do I use?" ambiguity
4. **Hard rules** — catches violations skills might miss
5. **Verification checklist** — prevents "done" claims without validation

### How Skills Guide Implementation

- Invoked by CLAUDE.md mandate or context match
- Provide detailed patterns, examples, anti-patterns
- Reference actual files in the repo ("see `lib/core/http/dio_provider.dart`")
- Cover architecture, testing, error handling for each domain

### How Hooks Validate Code

- Run after every Edit/Write automatically
- `analyze.sh` — blocks on `dart analyze` failure
- `format.sh` — auto-formats, non-blocking
- Catches syntax errors, lint violations, import issues immediately

### The Enforcement Loop

```
Developer asks Claude to add feature
    ↓
CLAUDE.md forces skill invocation (e.g., flai-architecture + flai-riverpod)
    ↓
Skills guide Claude to generate correct code structure
    ↓
Hooks validate generated code
    ↓
If hooks fail → Claude sees error and fixes
    ↓
If hooks pass → work complete
```

## Why This Design Works

### For Junior Developers
- Start with correct structure (no blank canvas)
- CLAUDE.md prevents "just make it work" shortcuts
- Skills teach patterns through examples
- Hooks catch mistakes immediately

### For Senior Developers
- Skip manual boilerplate setup
- Focus on features, not folder structure
- Standards are documented and enforced
- Can override when needed (CLAUDE.md allows asking for exceptions)

### For the Team
- Consistent codebase across projects
- Onboarding is "clone and read CLAUDE.md"
- Standards evolve in one place (flai-plugin)
- VGV's excellent CI/CD and testing infrastructure preserved

## Trade-offs

### Advantages
✅ Strong enforcement without being brittle  
✅ Leverages VGV's CI/CD and testing infrastructure  
✅ Plugin stays lightweight and reusable  
✅ Starter is just Flutter code + CLAUDE.md  
✅ Migration skill is reusable for any VGV project  
✅ Clear separation: plugin = standards, starter = implementation  

### Constraints
⚠️ Requires flai-plugin installation (one-time per developer)  
⚠️ CLAUDE.md template lives in migration skill (one place to maintain)  
⚠️ Initial refactor is substantial (but happens once)  
⚠️ Team must learn flai stack (Riverpod, fpdart, etc.)  

### Rejected Alternatives

**One repo with embedded skills/hooks:**  
- Rejected: Template users vs plugin users are different audiences; mixing them creates confusion

**VGV CLI in daily workflow:**  
- Rejected: Refactor is too large; junior devs would spend first day restructuring instead of building features

**Custom CLI tool:**  
- Rejected: Overkill for current team size; can add later if needed

## Success Criteria

### For flai-plugin
- [ ] `flai-vgv-migration` skill written with complete migration steps
- [ ] CLAUDE.md template embedded in skill as code block
- [ ] `plugin.json` updated with `vgv-migration`, `very-good-cli` keywords
- [ ] README.md documents the migration skill

### For flai-starter (created separately)
- [ ] `flutter pub get` succeeds
- [ ] `dart analyze` passes with zero issues
- [ ] `flutter test` passes all tests
- [ ] `flutter run` launches counter app
- [ ] CLAUDE.md exists and is complete
- [ ] All core/ providers implemented and working
- [ ] Counter feature demonstrates all three layers
- [ ] VGV CI/CD workflows preserved and passing

### For Team Adoption
- [ ] Junior developer can clone and start building features in < 30 minutes
- [ ] Claude follows flai standards without manual skill invocation
- [ ] Hooks catch violations before commit
- [ ] Code reviews show consistent architecture across features

## Implementation Plan

### Phase 1: Write Migration Skill (this repo)
1. Create `skills/vgv-migration/SKILL.md`
2. Document pre-migration checklist
3. Write detailed migration steps with code examples
4. Embed CLAUDE.md template as code block
5. Add post-migration validation checklist

### Phase 2: Update Plugin Metadata (this repo)
1. Update `.claude-plugin/plugin.json` keywords
2. Update `README.md` with migration skill documentation
3. Update `CLAUDE.md` repository structure section

### Phase 3: Build flai-starter (separate repo)
1. Run `very_good create flutter_app flai_starter`
2. Install flai-plugin
3. Invoke `flai-vgv-migration` skill
4. Follow migration steps
5. Validate all success criteria
6. Push to new repo

### Phase 4: Team Rollout
1. Share flai-starter repo with team
2. Document clone → rename → run workflow
3. Ensure all developers have flai-plugin installed
4. Gather feedback on first feature implementations
5. Iterate on skills/CLAUDE.md based on real usage

## Open Questions

None — design is complete and approved.

## References

- VeryGood CLI: https://cli.vgv.dev/
- flai-plugin skills: `/skills/` directory
- Clean Architecture: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- Riverpod: https://riverpod.dev/
