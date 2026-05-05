---
name: flai-license-compliance
description: >
  Audits package dependency licenses for Dart and Flutter projects.
  Flags non-compliant or unknown licenses and produces a compliance summary.
  Use when user says "check licenses", "license audit", "are our dependencies compliant",
  "check dependency licenses", "license compliance", "review package licenses",
  "scan for license issues", or "pre-release license check".
argument-hint: "[project-directory]"
allowed-tools: Read Glob Grep Bash
model: sonnet
effort: medium
---

# License Compliance

Dependency license auditor for Dart and Flutter projects — verifies that all package dependencies use licenses compatible with the project's requirements.

---

## Core Standards

Apply these standards to ALL license compliance work:

- **Run license check commands** on the target project directory to display full license information
- **A missing license is not "no license"** — it means "all rights reserved" by default; always flag
- **Transitive dependencies matter** — a permissive package that depends on a GPL package still carries the GPL obligation
- **Flag for manual review when in doubt** — never assume compliance without a clear license identifier

---

## License Categories

| Category | Licenses | Risk | Guidance |
| --- | --- | --- | --- |
| **Permissive** | MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0 | Low | Safe for any use |
| **Weak copyleft** | LGPL-2.1, LGPL-3.0, MPL-2.0 | Medium | Safe for dynamic linking; flag for static linking or modification |
| **Strong copyleft** | GPL-2.0, GPL-3.0, AGPL-3.0 | High | May require the entire project to adopt the same license |
| **Unknown/Missing** | None detected | High | Flag immediately for manual review |

---

## Audit Process

### 1. Run License Check

Run `dart pub deps --json | dart run dependency_validator` or manually inspect `pubspec.lock` for license metadata. For a structured audit, use `dart pub global activate pana` and run `pana .` to get package metadata including licenses.

Alternatively, use the `pub.dev` API or check each package's `pubspec.yaml` for a `license` field manually.

### 2. Categorize Results

Classify each dependency license using the categories above. Pay attention to:

- Direct dependencies with strong copyleft licenses
- Transitive dependencies that introduce copyleft obligations
- Packages with no license or an unrecognized license identifier

### 3. Report Findings

Produce a structured compliance report:

```markdown
## License Compliance Report

### Summary
- Total dependencies scanned: N
- Compliant: N
- Flagged: N

### Flagged Dependencies
| Package | License | Risk | Recommendation |
| --- | --- | --- | --- |
| package_name | GPL-3.0 | High | Replace or obtain exception |

### Compliant Dependencies
All other dependencies use permissive licenses (MIT, BSD, Apache 2.0).

### Recommendations
1. [Most urgent action]
2. [Next action]
```