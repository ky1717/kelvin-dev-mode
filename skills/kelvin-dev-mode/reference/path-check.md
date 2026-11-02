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

## Git Bash (no Python)

`status_paths` prints the changed paths one per line, sorted in byte order (`LC_ALL=C`, which for UTF-8 is code point order). `check` takes sorted files of paths (the baseline, the returned list, the handled forbidden paths) and prints the two differences.

```bash
# status_paths <worktree>: the changed paths, one per line, sorted by byte (= code point) order
status_paths() {
  local skip=
  git -C "$1" status --porcelain=v1 -z -uall |
  while IFS= read -r -d '' rec; do
    if [ -n "$skip" ]; then printf '%s\n' "$rec"; skip=; continue; fi   # second path of an R or C record
    printf '%s\n' "${rec:3}"
    case "${rec:0:2}" in *R*|*C*) skip=1 ;; esac
  done | LC_ALL=C sort
}

# check <worktree> <baseline.txt> <listed.txt> <handled.txt>   (all sorted with LC_ALL=C, one path per line)
check() {
  local wt=$1 base=$2 listed=$3 handled=$4 t; t=$(mktemp -d)
  LC_ALL=C comm -23 "$listed" "$handled" > "$t/listed"                 # listed - handled
  LC_ALL=C comm -23 "$base" "$t/listed" > "$t/baseline_only"           # baseline - listed
  status_paths "$wt" | LC_ALL=C comm -23 - "$t/baseline_only" > "$t/current"
  echo "only in list:";  LC_ALL=C comm -23 "$t/listed" "$t/current"
  echo "only in tree:";  LC_ALL=C comm -13 "$t/listed" "$t/current"
  rm -rf "$t"
}
```

Run on 2026-11-03 in Git Bash on Windows, in a toy repository with a new file, a file name with a space, a file name in Chinese, a rename made by `git mv` (the record is `R  moved.txt` and then `keep.txt`: new path first), and a rename made by `mv` (` D old.txt` plus `?? renamed.txt`). The seven paths came out as expected; `check` reported nothing when the list was complete, reported a listed path that does not exist, passed it once it was handled, reported an unnamed path, and ignored a baseline path. Not tested: other operating systems, an ignored file in the returned list, a non-UTF-8 console.

## PowerShell 5.1 (no Python)

Read the git output as bytes, so the pipeline does not re-encode it (the claim that the pipeline re-encodes non-ASCII output was not tested here; the snippet avoids the question). `Get-StatusPaths` returns the paths in ordinal order; `Compare-Paths` returns the two differences.

```powershell
# Get-StatusPaths <worktree>: the changed paths as a sorted string array (ordinal = code point order)
function Get-StatusPaths([string]$Worktree) {
    $psi = New-Object System.Diagnostics.ProcessStartInfo
    $psi.FileName = 'git'
    $psi.Arguments = '-C "' + $Worktree + '" status --porcelain=v1 -z -uall'
    $psi.UseShellExecute = $false
    $psi.RedirectStandardOutput = $true
    $proc = [System.Diagnostics.Process]::Start($psi)
    $ms = New-Object System.IO.MemoryStream
    $proc.StandardOutput.BaseStream.CopyTo($ms)      # raw bytes: no re-encoding by the pipeline
    $proc.WaitForExit()
    $records = [System.Text.Encoding]::UTF8.GetString($ms.ToArray()).Split([char]0)
    $paths = New-Object 'System.Collections.Generic.HashSet[string]'
    $i = 0
    while ($i -lt $records.Length) {
        $rec = $records[$i]; $i++
        if ($rec.Length -lt 4) { continue }
        [void]$paths.Add($rec.Substring(3))
        $xy = $rec.Substring(0, 2)
        if ($xy.Contains('R') -or $xy.Contains('C')) { [void]$paths.Add($records[$i]); $i++ }   # second path: the old name
    }
    $sorted = New-Object 'System.Collections.Generic.List[string]' (, $paths)
    $sorted.Sort([System.StringComparer]::Ordinal)
    return $sorted.ToArray()
}

# Compare-Paths <worktree> <baseline> <listed> <handled>: returns the two differences
function Compare-Paths([string]$Worktree, [string[]]$Baseline, [string[]]$Listed, [string[]]$Handled) {
    $listed = New-Object 'System.Collections.Generic.HashSet[string]' (, ([string[]]($Listed | Where-Object { $Handled -notcontains $_ })))
    $baselineOnly = $Baseline | Where-Object { -not $listed.Contains($_) }
    $current = New-Object 'System.Collections.Generic.HashSet[string]' (, ([string[]](Get-StatusPaths $Worktree | Where-Object { $baselineOnly -notcontains $_ })))
    [pscustomobject]@{
        OnlyInList = @($listed | Where-Object { -not $current.Contains($_) } | Sort-Object { $_ } )
        OnlyInTree = @($current | Where-Object { -not $listed.Contains($_) } | Sort-Object { $_ })
    }
}
```

Run on 2026-11-03 with Windows PowerShell 5.1 (5.1.26100) in the same toy repository and with the same five checks; the Chinese file name decoded correctly (U+4E2D U+6587). The two differences are sorted for display with the default `Sort-Object`, which is not ordinal. Not tested: a non-UTF-8 console, an ignored file in the returned list, PowerShell 7.
