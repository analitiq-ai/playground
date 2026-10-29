# Implementation Plan: Title-to-branch-name slug script

**Branch**: `feat/24-slug-branch-name` | **Date**: 2026-10-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/24-slug-branch-name/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Add `bin/slug.py`, a standalone stdlib-only Python 3 script that joins its arguments into a title, lowercases
ASCII `A–Z` only, and prints the maximal `[a-z0-9]` runs joined by `-`. No arguments → usage on stderr, exit 2;
no runs → error on stderr, exit 1; stdout stays empty on both. Verified by a black-box `unittest` suite that runs the
script as a subprocess.

## Technical Context

**Language/Version**: Python 3, syntax kept to 3.9 (R1)

**Primary Dependencies**: None — stdlib `re`, `string`, `sys` (R1)

**Storage**: N/A

**Testing**: stdlib `unittest`, subprocess black-box, `python3 -m unittest discover -s tests` (R6)

**Target Platform**: Any POSIX shell with `python3` on `PATH` (macOS dev machines, Linux CI runners)

**Project Type**: Single-file CLI script

**Performance Goals**: N/A — one short string per invocation

**Constraints**: Deterministic and locale-independent (FR-010); ASCII-only case mapping (FR-003); stdout empty on
any error (FR-008, FR-009)

**Scale/Scope**: 1 script (~30 lines), 1 test module

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is the unfilled template — no ratified principles, so no gates apply. Checked
instead against the org coding principles: single pure transformation, no options/dependencies (YAGNI), fail loud
on bad input, red tests first (R6). **PASS** pre-research and post-design.

## Project Structure

### Documentation (this feature)

```text
specs/24-slug-branch-name/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions R1–R6
├── data-model.md        # Phase 1 — Title, Slug, outcomes
├── quickstart.md        # Phase 1 — validation guide
├── contracts/
│   └── cli.md           # Phase 1 — CLI I/O contract and examples
└── tasks.md             # Phase 2 (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
bin/
└── slug.py              # new, executable (chmod +x)

tests/
└── test_slug.py         # new, unittest, runs bin/slug.py via subprocess
```

**Structure Decision**: Repo has no source tree; `bin/` path is fixed by FR-001, `tests/` added beside it.
No package, no `__init__.py`, no `pyproject.toml`.

## Complexity Tracking

No violations.

## Mechanisms

- Executable stdlib-only Python script at `bin/slug.py`: R1
- ASCII-only lowercasing via `str.maketrans` table: R2
- `[a-z0-9]+` run extraction and `-` join: R3
- Raw-argv title assembly, no option parser, no stdin: R4
- Exit codes and stderr messages for usage / empty slug: R5
- Black-box `unittest` subprocess suite in `tests/test_slug.py`: R6
