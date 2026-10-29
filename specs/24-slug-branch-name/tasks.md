---

description: "Task list for feat/24-slug-branch-name"
---

# Tasks: Title-to-branch-name slug script

**Input**: Design documents from `/specs/24-slug-branch-name/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/cli.md, quickstart.md

**Tests**: Requested — research R6 and the org coding principles require red tests first. Every test task must be
run and seen failing for the stated reason before the matching implementation task starts.

**Organization**: Tasks are grouped by user story. Only two files are created (`bin/slug.py`,
`tests/test_slug.py`), so tasks touching the same file are sequential; [P] is rare by construction.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2)

## Path Conventions

- Single-file CLI at repo root: `bin/slug.py`, `tests/test_slug.py`. No package, no `__init__.py`, no
  `pyproject.toml` (plan.md Structure Decision).
- Run the suite from the repo root: `python3 -m unittest discover -s tests -v`.

---

## Phase 1: Setup (Shared Infrastructure)
Story issue: #29

**Purpose**: Test harness that every story's tests call. No script yet — the suite must fail against an absent
`bin/slug.py`.

- [ ] T001 Create `tests/test_slug.py` with stdlib-only imports (`locale`, `os`, `re`, `subprocess`, `sys`,
  `unittest`, `pathlib.Path`), module constants `SCRIPT = Path(__file__).resolve().parent.parent / "bin" / "slug.py"`
  and `SLUG_RE = re.compile(r"^[a-z0-9]+(-[a-z0-9]+)*$")`, and a helper
  `run(*args, env=None, stdin=subprocess.DEVNULL)` returning `subprocess.run([sys.executable, str(SCRIPT), *args],
  capture_output=True, env=env, stdin=stdin)` (bytes, not text, so byte-identical comparisons are exact). Add a
  `if __name__ == "__main__": unittest.main()` footer. No test classes yet.

---

## Phase 2: Foundational (Blocking Prerequisites)
Story issue: #29

None. The feature is one pure transformation in one file; there is no shared infrastructure beyond T001.

---

## Phase 3: User Story 1 - Get a slug from an issue title (Priority: P1) 🎯 MVP
Story issue: #29

**Goal**: `bin/slug.py TITLE...` prints the slug (maximal ASCII `[a-z0-9]` runs joined by `-`) plus `\n` on stdout,
nothing on stderr, exit 0.

**Independent Test**: `python3 -m unittest tests.test_slug.SlugSuccessTest -v` passes; manually,
`bin/slug.py 'Fix NULL pointer in v2 API!!'` prints `fix-null-pointer-in-v2-api` with exit 0.

### Tests for User Story 1 ⚠️ write first, run, confirm they FAIL

- [ ] T002 [US1] In `tests/test_slug.py` add `class SlugSuccessTest(unittest.TestCase)` with a table-driven
  `test_examples` using `self.subTest` over every exit-0 row of contracts/cli.md Examples, asserting
  `stdout == expected + b"\n"`, `stderr == b""`, `returncode == 0`, and `SLUG_RE.fullmatch(stdout.decode().rstrip("\n"))`
  (SC-002). Rows (single argument each unless noted):
  `'Add bin/slug.py: turn a title into a branch name'` → `add-bin-slug-py-turn-a-title-into-a-branch-name`;
  `'Fix NULL pointer in v2 API!!'` → `fix-null-pointer-in-v2-api`;
  `'--force push fix'` → `force-push-fix`; `'--help'` → `help`;
  `'  --Hello World--  '` → `hello-world`; `'a   ---   b'` → `a-b`; `'Release 2.0.1'` → `release-2-0-1`;
  `'Café menu'` → `caf-menu`; `'naïve'` → `na-ve`; `'İx'` (İ, U+0130) → `x`;
  `'a\tb\nc'` → `a-b-c` (whitespace/control chars are separators); a 300-word title
  `" ".join(["word"] * 300)` → `"-".join(["word"] * 300)` (FR-007, no truncation).
  Write non-ASCII code points as `\u` escapes in source.
- [ ] T003 [US1] In `SlugSuccessTest` add `test_multiple_args_join_with_single_space`: `run("Fix", "NULL", "pointer")`
  and `run("Fix NULL pointer")` both give `b"fix-null-pointer\n"`, exit 0 (FR-002, US1 scenario 4); also
  `run("--force", "push", "fix")` → `b"force-push-fix\n"` (no option parsing, R4).
- [ ] T004 [US1] In `SlugSuccessTest` add `test_deterministic_repeat`: run the same title twice, assert stdout,
  stderr and returncode are byte-identical (FR-010, SC-003, US1 scenario 3).
- [ ] T005 [US1] In `SlugSuccessTest` add `test_locale_independent` for title `'İx Kelvin K'`: run once
  with `env={**os.environ, "LC_ALL": "C"}` and once with `env={**os.environ, "LC_ALL": "tr_TR.UTF-8"}`; assert both
  stdouts are byte-identical and equal `b"x-kelvin\n"` (Kelvin sign U+212A is not lowercased to `k`). Before the
  tr_TR run, probe availability with `locale.setlocale(locale.LC_ALL, "tr_TR.UTF-8")` in a `try`, restore with
  `locale.setlocale(locale.LC_ALL, "")` in `finally`, and `self.skipTest(...)` on `locale.Error` (SC-004, R6).
- [ ] T006 [US1] In `SlugSuccessTest` add `test_runs_directly_by_path`: `subprocess.run([str(SCRIPT), "Hello World"],
  capture_output=True)` → stdout `b"hello-world\n"`, exit 0; this covers the `#!/usr/bin/env python3` shebang and
  the executable bit (FR-001, R1).
- [ ] T007 [US1] Run `python3 -m unittest discover -s tests -v` and confirm every `SlugSuccessTest` test fails
  because `bin/slug.py` does not exist (python: "can't open file" / `FileNotFoundError` for T006) — not because of a
  test bug. Record the failure output in the commit message or PR description.

### Implementation for User Story 1

- [ ] T008 [US1] Create `bin/slug.py`: first line `#!/usr/bin/env python3`; import only `re`, `string`, `sys` (R1,
  Python 3.9 syntax); module-level `_ASCII_LOWER = str.maketrans(string.ascii_uppercase, string.ascii_lowercase)`
  (R2 — never `str.lower()`/`casefold()`); `_RUN = re.compile(r"[a-z0-9]+")` (R3 — explicit ranges, never `\w`/`\d`);
  `def slugify(title: str) -> str: return "-".join(_RUN.findall(title.translate(_ASCII_LOWER)))`;
  `def main(argv) -> int` that builds `title = " ".join(argv)` (R4, no `argparse`, never reads stdin), prints the slug
  to stdout, returns 0; `if __name__ == "__main__": sys.exit(main(sys.argv[1:]))`.
- [ ] T009 [US1] Set the executable bit: `chmod +x bin/slug.py`, and confirm git records mode `100755`
  (`git ls-files -s bin/slug.py` after `git add`) (FR-001).
- [ ] T010 [US1] Run `python3 -m unittest tests.test_slug.SlugSuccessTest -v`; all tests pass (T005 may skip if
  `tr_TR.UTF-8` is absent — note which).

**Checkpoint**: US1 complete — every success row of the contract passes. Error paths are not yet handled.

---

## Phase 4: User Story 2 - Fail loudly on unusable input (Priority: P2)
Story issue: #30

**Goal**: No arguments → usage on stderr, exit 2; title with no `[a-z0-9]` run → error naming the title on stderr,
exit 1; stdout empty in both cases.

**Independent Test**: `python3 -m unittest tests.test_slug.SlugErrorTest -v` passes; manually,
`bin/slug.py '!!!' 2>/dev/null; echo $?` prints only `1`.

### Tests for User Story 2 ⚠️ write first, run, confirm they FAIL

- [ ] T011 [US2] In `tests/test_slug.py` add `class SlugErrorTest(unittest.TestCase)` with `test_no_args_prints_usage`:
  call `subprocess.run([sys.executable, str(SCRIPT)], input=b"piped title\n", capture_output=True)` directly
  (the `run` helper does not feed input; US2 scenario 1 requires stdin to be ignored) and assert
  `stdout == b""`, `stderr == b"usage: bin/slug.py TITLE...\n"`, `returncode == 2` (FR-008, R5).
- [ ] T012 [US2] In `SlugErrorTest` add table-driven `test_no_run_is_an_error` with `self.subTest` over
  `'!!! ??? ---'`, `'日本語'` (日本語), `'K'` (Kelvin sign alone), and `''` (empty-string argument —
  FR-009, not FR-008): assert `stdout == b""`, `returncode == 1`, and
  `stderr == ("slug.py: no [a-z0-9] characters in title: " + repr(title) + "\n").encode()` (contracts/cli.md Output
  table; R5 `repr` form).
- [ ] T013 [US2] In `SlugErrorTest` add `test_undecodable_arg_does_not_crash`: `subprocess.run([sys.executable,
  str(SCRIPT), b"\xff"], capture_output=True)` → `stdout == b""`, `returncode == 1`, stderr starts with
  `b"slug.py: no [a-z0-9] characters in title: "` and contains no `Traceback` (data-model: surrogate-escaped bytes;
  R5 `backslashreplace` on stderr).
- [ ] T014 [US2] Run `python3 -m unittest tests.test_slug.SlugErrorTest -v` and confirm each test fails for the
  expected reason against the US1-only script: T011 gets `b"\n"` on stdout and exit 0; T012/T013 get `b"\n"` on
  stdout and exit 0. Any other failure (e.g. `NameError`, wrong helper) is a test bug — fix the test, not the script.

### Implementation for User Story 2

- [ ] T015 [US2] In `bin/slug.py` `main(argv)`: if `not argv`, `print("usage: bin/slug.py TITLE...",
  file=sys.stderr)` and `return 2` before touching stdout; compute the slug; if empty,
  `print(f"slug.py: no [a-z0-9] characters in title: {title!r}", file=sys.stderr)` and `return 1`; only then print the
  slug to stdout and `return 0` (stdout written only after the slug is known non-empty — R5). Do not use
  `sys.exit(str)` (always exits 1).
- [ ] T016 [US2] Run `python3 -m unittest discover -s tests -v`; both `SlugSuccessTest` and `SlugErrorTest` pass.

**Checkpoint**: Both stories pass; every row of contracts/cli.md Examples and Output is covered by a test.

---

## Phase 5: Polish & Cross-Cutting Concerns
Story issue: #30

- [ ] T017 Run every command in `specs/24-slug-branch-name/quickstart.md` sections 2–3 from the repo root and
  confirm output and `exit=` lines match exactly.
- [ ] T018 Verify constraints in `bin/slug.py`: imports are exactly `re`, `string`, `sys`; no `argparse`, `getopt`,
  `sys.stdin`, `.lower()`, `.casefold()`, `\w`, `\d`; no dead code; comments explain only *why* (e.g. why the
  translation table instead of `str.lower()`), not *what*.
- [ ] T019 Run the tripwire `git log --name-only main..HEAD | sort | uniq -c | sort -rn`; if any file exceeds ~3
  touches, squash/rewrite before opening the PR.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (T001)**: none.
- **Foundational**: empty.
- **US1 (T002–T010)**: depends on T001.
- **US2 (T011–T016)**: depends on T001 and on `bin/slug.py` existing from T008 — the error branches are added to
  the same `main()`. US2 tests (T011–T013) can be written right after T001, but their red run (T014) is meaningful
  only once T008 exists.
- **Polish (T017–T019)**: after T016.

### Within Each User Story

- Tests → red run → implementation → green run. Never write the test in the same step as the code it grades.
- T002–T006 and T011–T013 all edit `tests/test_slug.py` → sequential.
- T008, T009, T015 all touch `bin/slug.py` → sequential.

### Parallel Opportunities

- Only cross-file pairs are parallel: while T008 (`bin/slug.py`) is in progress, T011–T013 (`tests/test_slug.py`)
  can be authored. T018 and T019 are read-only and can run together.

---

## Parallel Example: User Story 1 / 2 overlap

```bash
# After T007 (US1 red run):
Task: "T008 Create bin/slug.py success path"
Task: "T011–T013 Author SlugErrorTest in tests/test_slug.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. T001 → T002–T006 → T007 (red) → T008–T009 → T010 (green).
2. **STOP and VALIDATE**: US1 independent test passes. The script slugs any usable title.

### Incremental Delivery

1. MVP (US1) → commit.
2. US2: T011–T013 → T014 (red) → T015 → T016 (green) → commit.
3. Polish T017–T019 → open DRAFT PR.

---

## Notes

- Commit after each green run (T010, T016); each commit carries the `alq-session-id` trailer.
- Out of scope (R6): CI wiring for the suite — recorded limitation, critique E1.
