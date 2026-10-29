# CLI Contract: `bin/slug.py`

## Synopsis

```text
bin/slug.py TITLE...
```

- Every argument is title text; arguments are joined with one space. No options exist (`--help`, `-x` are title
  text). Standard input is never read.

## Output

| Case | stdout | stderr | Exit |
|---|---|---|---|
| Title has ≥1 `[a-z0-9]` run | slug, then `\n` | nothing | `0` |
| Title has no run | nothing | `slug.py: no [a-z0-9] characters in title: '<title>'` (Python `repr`) | `1` |
| No arguments | nothing | `usage: bin/slug.py TITLE...` | `2` |

## Slug rule

1. Map ASCII `A–Z` → `a–z`; no other character is case-mapped.
2. Take maximal runs of `[a-z0-9]`, in order.
3. Join with `-`.

Result always matches `^[a-z0-9]+(-[a-z0-9]+)*$`. No length cap, no prefix, no issue number.

## Examples

| Arguments | stdout | Exit |
|---|---|---|
| `'Add bin/slug.py: turn a title into a branch name'` | `add-bin-slug-py-turn-a-title-into-a-branch-name` | 0 |
| `'Fix NULL pointer in v2 API!!'` | `fix-null-pointer-in-v2-api` | 0 |
| `Fix NULL pointer` (3 args) | `fix-null-pointer` | 0 |
| `'--force push fix'` | `force-push-fix` | 0 |
| `--help` | `help` | 0 |
| `'  --Hello World--  '` | `hello-world` | 0 |
| `'a   ---   b'` | `a-b` | 0 |
| `'Release 2.0.1'` | `release-2-0-1` | 0 |
| `'Café menu'` | `caf-menu` | 0 |
| `'naïve'` | `na-ve` | 0 |
| `'K'` (U+212A Kelvin) | — | 1 |
| `'İx'` (U+0130) | `x` | 0 |
| `'日本語'` | — | 1 |
| `'!!! ??? ---'` | — | 1 |
| `''` (empty string) | — | 1 |
| *(none)* | — | 2 |

The consumer composes the branch name. Capture the slug in a plain assignment so its exit status is checked
(`set -e` ignores a failing substitution embedded in another command's arguments):

```bash
slug=$(bin/slug.py "$TITLE") || exit
branch="feat/24-$slug"
```
