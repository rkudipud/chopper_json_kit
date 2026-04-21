# Chopper JSON Kit — Standalone Package

**Version:** 1.0.2  
**Date:** April 2026  
**Status:** Shippable before Chopper runtime

This package gives teams everything needed to author, validate, and organize Chopper JSON files — no Chopper installation required.

---

## What Is This?

Chopper trims EDA tool domain codebases via three JSON configuration files:

| File | Purpose |
|------|---------|
| **Base JSON** | Defines the minimal viable flow for a domain (files, procs, stages) |
| **Feature JSON** | Extends or overrides the base for optional or project-specific scenarios |
| **Project JSON** | Selects and orders one base + zero or more features for a specific trim run |

You author these JSONs now. When Chopper is released, you run `chopper trim --project project.json` and it does the rest.

---

## Package Contents

```
chopper_json_kit/
├── README.md                        ← You are here
├── AGENTS.md                        ← Agent profile and behavior contract for assistants
├── VERSION.txt                      ← Schema version tracking
├── setup.csh                        ← tcsh/csh environment bootstrap (primary path)
├── setup.ps1                        ← PowerShell environment bootstrap
├── validate_jsons.py                ← One-command schema validation helper
├── schemas/
│   ├── base-v1.schema.json          ← Base JSON schema (authoritative validator)
│   ├── feature-v1.schema.json       ← Feature JSON schema
│   └── project-v1.schema.json       ← Project JSON schema
├── docs/
│   └── JSON_AUTHORING_GUIDE.md      ← Complete field reference, rules, decision guide
├── examples/
│   ├── 01_base_files_only/          ← files.include + files.exclude only
│   ├── 02_base_procs_only/          ← procedures.include + procedures.exclude only
│   ├── 03_base_stages_only/         ← stages only (run-file generation)
│   ├── 04_base_files_and_procs/     ← files + procedures (no stages)
│   ├── 05_base_files_and_stages/    ← files + stages
│   ├── 06_base_procs_and_stages/    ← procedures + stages
│   ├── 07_base_full/                ← files + procedures + stages (maximum control)
│   ├── 08_base_plus_one_feature/    ← base + one feature + project
│   ├── 09_base_plus_multiple_features/ ← base + two independent features + project
│   ├── 10_chained_features_depends_on/ ← three-level depends_on chain + project
│   └── 11_project_base_only/        ← base-only trim (no features)
└── agent/
    └── DOMAIN_ANALYZER.md           ← Agent instructions for codebase analysis and JSON authoring
```

---

## 10-Minute Quick Start

### 0. Bootstrap Python environment on tcsh/csh systems

```tcsh
source setup.csh
```

This creates and activates `.venv` automatically and installs `jsonschema`, which is required for schema validation examples in this repo.

Windows PowerShell:

```powershell
. .\setup.ps1
```

### 1. Choose your starting example

| Your situation | Start with |
|---------------|-----------|
| Need to trim files only | `examples/01_base_files_only/` |
| Need proc-level surgical trimming | `examples/02_base_procs_only/` |
| Have stack files to translate | `examples/03_base_stages_only/` |
| Full control (files + procs + stages) | `examples/07_base_full/` |
| Single optional feature | `examples/08_base_plus_one_feature/` |
| Multiple independent features | `examples/09_base_plus_multiple_features/` |
| Features depend on each other | `examples/10_chained_features_depends_on/` |

### 2. Copy and adapt

```bash
cp -r examples/07_base_full/ my_domain/chopper/
cd my_domain/chopper/
# Edit jsons/base.json: change domain, owner, file lists, stage definitions
```

### 3. Validate against schemas (one command)

```bash
python validate_jsons.py my_domain/
```

Examples:

```bash
python validate_jsons.py
python validate_jsons.py examples/08_base_plus_one_feature/
python validate_jsons.py my_domain/chopper/
```

The script validates Base/Feature/Project JSONs based on `$schema`, prints clear `OK/ERR/SKIP` lines, and returns non-zero on validation failures.

### 4. Set up and use the domain analyzer agent

Use both files below in your assistant:
- `AGENTS.md` for high-level agent profile and repository-specific rules
- `agent/DOMAIN_ANALYZER.md` for the full 8-phase domain analysis protocol

Then ask:

> "Analyze my domain directory at `my_domain/` and help me author the base, feature, and project JSONs."

The agent follows an 8-phase process: inventory → stack extraction → proc extraction → base/feature split → authoring → validation.

### 5. Validate using the Python validator (`validate_jsons.py`)

```bash
python validate_jsons.py
python validate_jsons.py examples/08_base_plus_one_feature/
python validate_jsons.py my_domain/chopper/
```

If `jsonschema` is missing:

```bash
python -m pip install jsonschema
```

---

## Agent Setup and Conversational Workflow

### Agent setup options

1. **Copilot coding agent / cloud agent setup**
   - Keep repository `AGENTS.md` at repo root.
   - Keep protocol file at `agent/DOMAIN_ANALYZER.md`.
   - In the task prompt, point the agent to your domain root and expected output JSON locations.

2. **Chat assistant setup (manual)**
   - Paste `AGENTS.md` as system-level instructions.
   - Paste or attach `agent/DOMAIN_ANALYZER.md` as the domain-analysis playbook.
   - Provide the domain boundary and file inventory before asking for final JSON authoring.

### Conversational facilities (collaborative mode)

The agent is designed to work interactively, not as a blind one-shot generator. Expected checkpoints:

1. Confirm domain boundary (where analysis is allowed).
2. Confirm top-level entry files/procs for call-tree roots.
3. Review inventory classification (core vs optional vs deprecated).
4. Review proc trace log (roots, edges, unresolved calls, coverage).
5. Confirm include/exclude recommendations before final JSON writing.

This conversational loop improves accuracy for `files.include`, `files.exclude`, `procedures.include`, and `procedures.exclude`.

---

## Where to Put Your JSON Files

Convention:

```
<domain_root>/
└── chopper/
    ├── jsons/
    │   ├── base.json
    │   └── features/
    │       ├── feature_a.feature.json
    │       └── feature_b.feature.json
    └── project_abc.json
```

Or at the project level:

```
projects/<PROJECT_ID>/
└── chopper/
    ├── project.json         ← points to domain base + selected features
```

Paths in `project.json` are relative to the domain root (where Chopper will be invoked).

---

## Key Rules (Quick Reference)

**Naming scheme:** Base JSON is `jsons/base.json`; feature JSONs are `jsons/features/<feature_name>.feature.json`.

1. **`$schema` is always required** and must be the exact literal string (e.g., `"chopper/base/v1"`).
2. **Base needs at least one of:** `files`, `procedures`, `stages`.
3. **Arrays must never be empty** when present — `minItems: 1` is enforced by schema.
4. **Paths:** forward slashes only, no `..`, no `//`, no absolute paths.
5. **`depends_on`** uses feature `name` values, not file paths.
6. **Project feature order** must satisfy all `depends_on` declarations (prerequisites first).
7. **`load_from` ≠ `dependencies`:** `load_from` = data predecessor for run script; `dependencies` = stack `D` line (scheduler order).

---

## Input Interaction Matrix

Chopper has four input sets per file. Mixing them creates ambiguity — this matrix resolves all 16 combinations.

**Inputs:** FI = `files.include`, FE = `files.exclude`, PI = `procedures.include`, PE = `procedures.exclude`

**Proc-selection models (choose one per file):**

| Model | Input | Meaning | Surviving procs |
|---|---|---|---|
| **Additive** | PI | "Keep only these procs" | PI procs from this file |
| **Subtractive** | PE | "Keep the file but remove these procs" | All procs minus PE procs |

**Per-file interaction matrix:**

| # | FI | FE | PI | PE | Treatment | Surviving procs | Warning |
|---|---|---|---|---|---|---|---|
| 1 | — | — | — | — | `REMOVE` | — | — |
| 2 | ✓ | — | — | — | `FULL_COPY` | all | — |
| 3 | — | ✓ | — | — | `REMOVE` | — | — |
| 4 | ✓ | ✓ | — | — | `FULL_COPY` (literal) / `REMOVE` (glob) | all / — | — |
| 5 | — | — | ✓ | — | `PROC_TRIM` | PI only | — |
| 6 | — | — | — | ✓ | `PROC_TRIM` | all − PE | — |
| 7 | — | — | ✓ | ✓ | `PROC_TRIM` | PI only (PE ignored) | `VW-12` |
| 8 | ✓ | — | ✓ | — | `FULL_COPY` | all (PI redundant) | `VW-09` |
| 9 | ✓ | — | — | ✓ | `PROC_TRIM` | all − PE | — |
| 10 | ✓ | — | ✓ | ✓ | `PROC_TRIM` | PI only (PE ignored) | `VW-12` |
| 11 | — | ✓ | ✓ | — | `PROC_TRIM` | PI only (FE overridden) | — |
| 12 | — | ✓ | — | ✓ | `REMOVE` | — | `VW-11` |
| 13 | — | ✓ | ✓ | ✓ | `PROC_TRIM` | PI only (PE+FE overridden) | `VW-12` |
| 14 | ✓ | ✓ | ✓ | — | `FULL_COPY` (literal) | all (PI redundant) | `VW-09` |
| 15 | ✓ | ✓ | — | ✓ | `PROC_TRIM` (literal) / `REMOVE` (glob) | all − PE / — | — |
| 16 | ✓ | ✓ | ✓ | ✓ | `PROC_TRIM` | PI only | `VW-12` |

**Key rules:**
- **PE downgrades FULL_COPY:** FI + PE → `PROC_TRIM` (case 9). A file with 100 procs and 4 in PE → 96 survive.
- **FE + PE = both remove:** neither says "keep" → file is removed (case 12). Use PE alone if you want to keep the file.
- **PI wins over PE:** if both reference the same file, PI takes precedence (cases 7, 10, 13, 16).
- **PI overrides FE:** PI forces file survival regardless of FE (cases 11, 13).
- **FI + PI (no PE) stays FULL_COPY:** PI is additive and redundant on a fully included file (cases 8, 14).

---

## Getting Help

- `validate_jsons.py` — one-command schema validation for any file/folder
- `docs/JSON_AUTHORING_GUIDE.md` — full field reference, all rules, decision flowchart
- `agent/DOMAIN_ANALYZER.md` — step-by-step domain analysis instructions for AI assistants
- `examples/` — working JSON files for every combination
- Schema files in `schemas/` are the authoritative validators — when in doubt, validate
