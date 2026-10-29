# Data Model: Title-to-branch-name slug script

No persistence; two values, one pure transformation.

## Title

- **Source**: `sys.argv[1:]` joined by a single space (research R4).
- **Validation**: at least one argument (FR-008). Content unrestricted: any Unicode, whitespace, control characters,
  surrogate-escaped bytes.

## Slug

- **Derived from**: Title → ASCII-only lowercase (R2) → maximal `[a-z0-9]+` runs (R3) → joined by `-`.
- **Invariant**: matches `^[a-z0-9]+(-[a-z0-9]+)*$` (SC-002). Never truncated (FR-007).
- **Validation**: at least one run, otherwise error (FR-009).

## Transformation outcomes

| Input | stdout | stderr | exit |
|---|---|---|---|
| ≥1 arg, ≥1 run | `<slug>\n` | empty | 0 |
| ≥1 arg, 0 runs | empty | error naming the title | 1 |
| 0 args | empty | usage | 2 |

Full I/O contract: [contracts/cli.md](./contracts/cli.md).
