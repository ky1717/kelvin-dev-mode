# Path check

The orchestrator's comparison of a worktree's changed paths with the implementer's returned list (`contracts.md`, The orchestrator's commit, steps 0 and 1). This file is the single source of the algorithm.

## Rules

1. Read the worktree with `git -C "<worktree>" status --porcelain=v1 -z -uall` and split the output at NUL. Each record is `XY <path>`. When X or Y is `R` or `C`, the next record is a second path: under `-z` the order is the new path, then the old path (without `-z` it is `old -> new`).
2. A file renamed without `git mv` shows as ` D <old>` plus `?? <new>`: treat it as a deletion of the old path and a creation of the new one. The returned list names both paths.
3. Both sides are sets of path strings, compared and sorted by code point (not by locale: `sort` and `comm` gave false differences under different locales).
4. Ignored files do not appear in the status. A returned path that is ignored counts as "in the list, not in the worktree" and goes back as the mechanical question.
5. Give every path to git after `--`. A path with spaces, non-ASCII characters or a leading `-` is safe that way; in PowerShell 5.1 put the path in quotes (tested safe).
6. The `-z` bytes of non-ASCII paths decoded correctly as UTF-8 in a test on Windows; whether they come out garbled on a console that is not UTF-8 was not tested (printing such paths to a cp1252 console raised `UnicodeEncodeError` in that test).

The reference implementation below was run on 2026-11-01 on a toy repository with a rename made by `git mv`, a path with spaces, a path with Chinese characters, a baseline file, a returned path that does not exist and the same path passed as a handled forbidden path; it reported the differences as expected.

Known limit: a baseline path (for example a lockfile that `install` changed) that the worker edits again without naming it leaves no difference, so that edit is not committed.

## Pseudocode

```text
listed         = listed - handled             # forbidden paths already restored or deleted (contracts.md, step 2)
worktree_paths = status_paths(worktree)
baseline_only  = baseline - listed            # install changed it, the worker did not
current        = worktree_paths - baseline_only
only_in_list   = listed - current             # named by the worker, absent in the worktree
only_in_tree   = current - listed             # changed in the worktree, not named
if both are empty: the check passes
else: send the sorted differences back as one question
```

## Reference implementation (Python, standard library)

```python
import subprocess

def status_paths(worktree):
    out = subprocess.run(
        ["git", "-C", worktree, "status", "--porcelain=v1", "-z", "-uall"],
        capture_output=True, check=True).stdout
    records = out.split(b"\0")
    paths, i = set(), 0
    while i < len(records):
        record = records[i]
        i += 1
        if not record:
            continue
        xy, path = record[:2].decode(), record[3:].decode("utf-8", "surrogateescape")
        paths.add(path)
        if "R" in xy or "C" in xy:
            paths.add(records[i].decode("utf-8", "surrogateescape"))
            i += 1
    return paths

def check(worktree, baseline, listed, handled=()):
    listed = set(listed) - set(handled)
    current = status_paths(worktree) - (baseline - listed)
    return sorted(listed - current), sorted(current - listed)

# baseline = status_paths(worktree)   # once, after the worktree exists and `install` has finished
# only_in_list, only_in_tree = check(worktree, baseline, set(returned_paths), handled_forbidden)
```
