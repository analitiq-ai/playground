# Research: Title-to-branch-name slug script

**Feature**: `feat/24-slug-branch-name` | **Spec**: [spec.md](./spec.md) | **Date**: 2026-10-29

No NEEDS CLARIFICATION remained in Technical Context; the spec's clarification session settled input, output,
length and failure behaviour. The decisions below settle how each mechanism is built.

## R1: Runtime — Python 3 standard library only, executable script

- **Decision**: `bin/slug.py` is a single file with a `#!/usr/bin/env python3` shebang and the executable bit set,
  importing only the standard library (`re`, `string`, `sys`). Code stays within Python 3.9 syntax.
- **Rationale**: FR-001 fixes the `.py` path and requires running it directly from the repo root. The repo has no
  Python project, lockfile or virtualenv, so any third-party dependency would add setup that the task does not need.
- **Existing tools**: None fit. Looked at: repo root (`README.md`, `codeworks.toml`), `.github/` (only the PR-gate
  workflow and composite action, no Python), `.specify/scripts/` (bash only, no slug helper). No `bin/`,
  `pyproject.toml` or `requirements*.txt` exists.
- **Alternatives considered**: bash + `tr`/`sed` — rejected: the path is fixed as `.py`, and `tr`/`sed` character
  classes depend on locale, which breaks FR-010 determinism.

## R2: Lowercasing — ASCII-only translation table, not `str.lower()`

- **Decision**: Lowercase with `title.translate(str.maketrans(string.ascii_uppercase, string.ascii_lowercase))`.
- **Rationale**: FR-003 and the edge cases require that only `A–Z` map to `a–z`. `str.lower()` and `str.casefold()`
  apply Unicode case mapping: the Kelvin sign `K` (U+212A) becomes `k`, and `İ` (U+0130) becomes `i̇`, which would
  leak an `i` into the slug. The translation table touches exactly 26 code points.
- **Existing tools**: `str.lower()` / `str.casefold()` — do not fit, Unicode mapping as above. `bytes.lower()` on an
  ASCII-encoded copy — would need an encode step with an error handler for non-ASCII input; the table is simpler.
- **Alternatives considered**: `unicodedata.normalize` + transliteration — rejected by the clarification "dropped as
  separators, not transliterated".

## R3: Run extraction — `re.findall(r"[a-z0-9]+", …)` joined by `-`

- **Decision**: `"-".join(re.findall(r"[a-z0-9]+", lowered))`.
- **Rationale**: Taking maximal `[a-z0-9]` runs and joining them is FR-004 stated directly, and gives FR-005 for free:
  no leading/trailing hyphen, no double hyphen, because hyphens are only ever inserted between non-empty runs.
  Explicit ranges match ASCII only; `\w` and `\d` would match Unicode letters and digits (e.g. `٣`).
- **Existing tools**: `re` (stdlib) — fits. `re.sub(r"[^a-z0-9]+", "-", s).strip("-")` — works too, but needs a
  separate strip step to meet FR-005; `findall` + `join` has no post-fix-up.

## R4: Argument handling — raw `sys.argv[1:]`, no option parser

- **Decision**: `title = " ".join(sys.argv[1:])`. No `argparse`/`getopt`. Standard input is never read.
- **Rationale**: FR-002 and the clarification make every argument title text, `--help` included. Any option parser
  would intercept `-`-prefixed arguments. Joining with one space makes quoted and unquoted titles equivalent
  (US1 scenario 4).
- **Existing tools**: `argparse` — does not fit, it claims `-h`/`--help` and rejects unknown `--force`. A `--`
  separator would force callers to remember it, which is the drift this tool removes.

## R5: Failure signalling — stderr message, exit 2 for usage, exit 1 for empty slug, empty stdout

- **Decision**:
  - No arguments → `usage: bin/slug.py TITLE...` on stderr, exit 2.
  - Title yields no run → `slug.py: no [a-z0-9] characters in title: <repr(title)>` on stderr, exit 1.
  - Success → slug + `\n` on stdout, exit 0. Stdout is written only after the slug is known to be non-empty.
- **Rationale**: FR-008/FR-009 require non-zero exit and empty stdout. Exit 2 for misuse follows the POSIX/`argparse`
  convention, letting callers tell "called wrong" from "title unusable". `repr()` keeps whitespace and control
  characters in the title visible and on one line; Python's stderr uses `backslashreplace`, so undecodable argv
  bytes (surrogate-escaped) cannot crash the error path.
- **Existing tools**: `sys.exit(str)` — prints to stderr but always exits 1, so it cannot give the usage case its own
  code; `print(..., file=sys.stderr)` + `sys.exit(n)` is used instead.

## R6: Tests — stdlib `unittest`, black-box via `subprocess`

- **Decision**: `tests/test_slug.py` runs `bin/slug.py` with `subprocess.run([sys.executable, SCRIPT, *args])` and
  asserts stdout, stderr and exit status per acceptance scenario and edge case. Run with
  `python3 -m unittest discover -s tests`. One extra test runs the script by path (no interpreter prefix) to cover
  the shebang/executable bit (FR-001). One test runs the same title (`'İx Kelvin K'`) under `LC_ALL=C` and
  `LC_ALL=tr_TR.UTF-8` (skipped if that locale is absent) and asserts byte-identical output (FR-010, SC-004).
- **Rationale**: The contract is the CLI — streams and exit codes — so the tests exercise it, not an imported
  function. `unittest` needs no install. Tests are written first and must fail (script absent) before the script
  exists.
- **Existing tools**: None fit. Looked at: repo has no test runner, no `tests/` directory, and the only CI workflow
  (`pr-gate.yml`) posts a check run and runs no tests. `pytest` — not a dependency of this repo; adding it is setup
  the task does not need.
- **Out of scope**: wiring the tests into CI. This throwaway repo has no test workflow; adding one is a separate
  decision from the slug script.
