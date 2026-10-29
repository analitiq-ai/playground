---
# planner's spec_facts.ts sets this block once the spec passes planner's checks; leave it as it is.
codeworks:
  v: 1
  kind: feature
  root_cause:
    observed: none
---
# Feature Specification: Title-to-branch-name slug script

**Feature Branch**: `feat/24-slug-branch-name`

**Created**: 2026-10-29

**Status**: Draft

**Input**: User description: "https://github.com/analitiq-ai/playground/issues/24 — Add `bin/slug.py`, a script that turns an issue title into a branch-name slug: lowercase, runs of `[a-z0-9]` joined by single hyphens, with other characters dropped."

## Clarifications

### Session 2026-10-29

- Q: Should the script read the title only from command-line arguments, or also from standard input when no argument is given? → A: Arguments only.
- Q: Should the output be the slug alone, or the full branch name with a type prefix and issue number? → A: Slug only; the caller composes the branch name.
- Q: Should the slug be capped in length or word count? → A: No cap; choosing which words name the change is the caller's job.
- Q: When the title leaves no `[a-z0-9]` characters, should the script fail or print an empty line? → A: Fail: error on the error stream, non-zero exit, nothing on standard output.
- Q: Should letters outside ASCII be dropped as separators or transliterated to ASCII? → A: Dropped as separators; only ASCII `A–Z` is lowercased.
- Q: Should every argument count as title text, including ones starting with `-`, or should the script accept options? → A: Every argument is title text; the script has no options, not even `--help`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Get a slug from an issue title (Priority: P1)

A developer or agent starting work on an issue runs `bin/slug.py` with the issue title and gets back the slug
part of the branch name, so the same title always yields the same slug no matter who derives it.

**Why this priority**: It is the whole feature. Without it, slugs stay hand-made and drift between people and agents.

**Independent Test**: Run the script with a known title and compare its output to the expected slug.

**Acceptance Scenarios**:

1. **Given** the title `Add bin/slug.py: turn a title into a branch name`, **When** the script is run with it, **Then** it prints `add-bin-slug-py-turn-a-title-into-a-branch-name` and exits with status 0.
2. **Given** the title `Fix NULL pointer in v2 API!!`, **When** the script is run with it, **Then** it prints `fix-null-pointer-in-v2-api`.
3. **Given** the same title run twice, **When** both outputs are compared, **Then** they are identical.
4. **Given** a title passed as several separate words (unquoted), **When** the script is run, **Then** the words are treated as one title joined by single spaces, giving the same slug as the quoted title.
5. **Given** the title `--force push fix` or `--help`, **When** the script is run with it, **Then** it prints `force-push-fix` or `help` respectively and exits with status 0 (verifies the no-options decision).

---

### User Story 2 - Fail loudly on unusable input (Priority: P2)

A caller (often a script composing `type/issue-number-slug`) passes a title that leaves no `[a-z0-9]` characters,
or passes nothing. The script refuses rather than printing an empty slug that would produce a malformed branch
name such as `feat/24-`.

**Why this priority**: A silent empty result propagates into a broken branch name; failing early keeps the caller honest.

**Independent Test**: Run the script with no title, and with a title of only punctuation, and check the exit status and error output.

**Acceptance Scenarios**:

1. **Given** no title argument (even with text on standard input), **When** the script is run, **Then** it does not read standard input, prints a usage message to the error stream, prints nothing to standard output, and exits non-zero.
2. **Given** the title `!!! ??? ---`, **When** the script is run, **Then** it prints an error naming the title to the error stream, prints nothing to standard output, and exits non-zero.

---

### Edge Cases

- Leading/trailing separators (`  --Hello World--  `) → `hello-world`; the slug never starts or ends with a hyphen.
- Runs of separators (`a   ---   b`) → `a-b`; never two hyphens in a row.
- Digits are kept and are not treated as separators (`Release 2.0.1` → `release-2-0-1`).
- Non-ASCII letters are separators, not transliterated (`Café menu` → `caf-menu`; `naïve` → `na-ve`).
- Characters outside ASCII `A–Z` are never lowercased into `[a-z]` (e.g. the Kelvin sign `K` or `İ` do not become `k`/`i`); only ASCII `A–Z` map to `a–z`.
- A title made only of non-ASCII characters (`日本語`) → error per User Story 2.
- Very long titles → the full slug is printed; no truncation.
- Newlines, tabs and other whitespace inside the title are separators like any other dropped character.
- An empty-string argument (`""`, e.g. an unset `$TITLE`) is a title with no `[a-z0-9]` run → error per FR-009, exit 1; FR-008 applies only when there are no arguments at all.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The script MUST live at `bin/slug.py` and be runnable directly from the repository root.
- **FR-002**: The script MUST take the title from its command-line arguments; multiple arguments MUST be joined with a single space into one title. Every argument is title text, including ones starting with `-`; the script has no options.
- **FR-003**: The script MUST map ASCII uppercase `A–Z` to lowercase `a–z` and treat every other character not in `[a-z0-9]` as a separator.
- **FR-004**: The script MUST output the maximal runs of `[a-z0-9]`, in order, joined by a single hyphen, followed by one newline, on standard output.
- **FR-005**: The output MUST NOT begin or end with a hyphen and MUST NOT contain consecutive hyphens.
- **FR-006**: The script MUST NOT add a type prefix, issue number, or any other text; it outputs the slug only.
- **FR-007**: The script MUST NOT truncate the slug.
- **FR-008**: When no arguments are given, the script MUST write a usage message to the error stream, write nothing to standard output, and exit non-zero.
- **FR-009**: When the title yields no `[a-z0-9]` run, the script MUST write an error to the error stream, write nothing to standard output, and exit non-zero.
- **FR-010**: The script MUST be deterministic: the same title always yields the same output and exit status.

### Key Entities

- **Title**: free text, typically a GitHub issue title.
- **Slug**: one or more `[a-z0-9]+` runs joined by single hyphens; matches `^[a-z0-9]+(-[a-z0-9]+)*$`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For every acceptance scenario and edge case listed above, the script's output and exit status match the stated expectation (100% pass).
- **SC-002**: Every non-error output matches the pattern `^[a-z0-9]+(-[a-z0-9]+)*$`.
- **SC-003**: Running the script twice on the same title produces byte-identical output in 100% of tested cases.
- **SC-004**: The slug depends only on the title: running under a different locale or environment gives byte-identical output.

## Assumptions

- Input is by argument only; reading from standard input is out of scope (callers can use `"$(cat)"` or command substitution).
- Composing the full branch name (`type/issue-number-slug`) is the caller's job; this script produces only the slug segment.
- No word or length cap: the issue fixes only the character rule, and any cap would be a separate decision about how "short" a branch name should be.
- "Lowercase" means ASCII `A–Z` → `a–z` only; the issue's "other characters dropped" rule covers everything else, so non-ASCII letters are dropped rather than transliterated or Unicode-case-folded.
- Empty result is an error, not empty output, because an empty slug would silently produce a malformed branch name.
- No existing slug code or `bin/` directory exists in the repo to reuse or align with.

## Why this is worth doing

### harm
Answer: Nothing breaks. Branch names keep being derived by hand from issue titles, so the same title can yield different slugs from different people or agents.
Source: user: issue #24 asks for a new script; code: no slug logic exists in the repo (only README.md, codeworks.toml, .github/, .specify/, .claude/).

### reachable
Answer: predicted. There is no slug tool on main, so every branch name is hand-made.
Source: code: main at fc7d3ff has no bin/ directory and no slug code.

### change-alters
Answer: Yes. A title passed to bin/slug.py gives one deterministic slug (lowercase, runs of [a-z0-9] joined by single hyphens, other characters dropped), so the same title always gives the same name.
Source: user: issue #24 body.

### owner
Answer: Only a prose rule (the branch-naming rules: lowercase kebab-case, type/issue-number-short-description). Nothing executes or checks it.
Source: code: no CI job or script in .github/ checks branch names; user: branch naming rules in the project's GitHub workflow rules.

### inherited
Answer: No. There is no specs/ folder on main; this is the first spec.
Source: code: main has no specs/ directory.

### needs-spec
Answer: It has decisions. The request fixes the character rule, but not: how the title is passed (argument or stdin), whether the output carries a type prefix or issue number, whether the word count is capped, and what happens when no [a-z0-9] run remains (empty output or error).
Source: user: issue #24 body names only the character rule.

## Root cause

Not a defect report.
