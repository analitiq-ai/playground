# Quickstart: validate `bin/slug.py`

**Prerequisites**: `python3` (3.9+) on `PATH`. Run from the repository root. No install step.

## 1. Automated suite

```bash
python3 -m unittest discover -s tests -v
```

Expected: all tests pass. Covers every row of [contracts/cli.md](./contracts/cli.md#examples), stream separation,
and the determinism check (SC-003).

## 2. Manual smoke checks

```bash
bin/slug.py 'Add bin/slug.py: turn a title into a branch name'; echo "exit=$?"
# add-bin-slug-py-turn-a-title-into-a-branch-name
# exit=0

bin/slug.py --help; echo "exit=$?"
# help
# exit=0

bin/slug.py '!!! ??? ---' 2>/dev/null; echo "exit=$?"
# exit=1            (nothing on stdout)

echo 'piped title' | bin/slug.py 2>/dev/null; echo "exit=$?"
# exit=2            (stdin ignored, nothing on stdout)
```

## 3. Caller composition

```bash
slug=$(bin/slug.py 'Fix NULL pointer in v2 API!!') && echo "feat/24-$slug"
# feat/24-fix-null-pointer-in-v2-api

slug=$(bin/slug.py '!!!') && echo "feat/24-$slug"; echo "exit=$?"
# exit=1            (no malformed feat/24- printed)
```
