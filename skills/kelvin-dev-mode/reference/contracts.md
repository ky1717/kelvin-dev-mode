# Contracts

Read at preflight. This file is the single source for the `list-tickets` format and for what the three roles must do, so you can supply your own.

**The contract is the baseline; a skill is an optional way to meet it.** A role with no skill in the adapter is done directly from the contract below. A **user-only skill** is a skill that only a user can start, not a model-written prompt; the marker depends on the harness (`harness/`). A worker may be unable to call one. For a named user-only skill, three steps, in order:

- (a) The worker calls the skill if it can.
- (b) Otherwise, if the skill's file is readable, the worker reads that file (and any reference skill it names, such as `tdd`) and follows the steps that apply. Where a step conflicts with the contract, the contract wins: no push, no PR, no `forbidden-files` path, and skip any self-review step that would duplicate the independent review (for example a final two-axis review inside `implement`); the orchestrator arranges the independent review.
- (c) If the file cannot be read either, the worker works directly from the contract below, including the quality baseline for `implement`.

**Prompt line.** The orchestrator's line 1 for a role with a skill is the call, written the way the worker's application calls skills (`harness/`), followed by "if you cannot call it, take steps (b) and (c)"; for a role with no skill it says to work from the contract below.

## list-tickets

The adapter's `list-tickets` command prints one line per open ticket: the ticket number (a `#` before it is fine), then `READY` or `BLOCKED`, then the title, separated by tabs. `BLOCKED` means the ticket has at least one **open** blocker. Preflight runs the command once and refuses the run if any line does not give a number and `READY`/`BLOCKED`. A ticket just created or just relabelled can take some time to show in the list (about 20 seconds measured, 2026-11-05).

Runnable example (bash; lists issues labelled `ready-for-agent`, using GitHub's native issue dependencies, the `blockedBy` field):

```bash
gh issue list --state open --label ready-for-agent --limit 200 \
  --json number,title,blockedBy \
  --jq '.[] | "#\(.number)\t\(if ([.blockedBy.nodes[] | select(.state == "OPEN")] | length) > 0 then "BLOCKED" else "READY" end)\t\(.title)"'
```

The example was run with `gh` 2.103.2; other versions are untested, so check that `gh issue list --json blockedBy` works in yours. In PowerShell, quote the `--jq` expression differently.

## Implementation skill (role `implement`)

This is the quality baseline for the implementer, whether or not a skill is named.

- Input: a ticket number. The working directory is a prepared git worktree. How the ticket is read is the executor's business: a bridge or cli worker uses the read tool its application provides (`executors/bridge.md`, Reading tickets; `executors/cli.md`), a sub-agent uses `gh`.
- Before starting: read the ticket and state its title; read the project's `GLOSSARY.md`, ADRs, coding-standards documents and the like where they exist, and skip them where they do not. If a skill named `tdd` is installed and the worker can call it, call it for the work below; otherwise follow the next list.
- Must (when working from the contract):
  1. Test first, one vertical slice at a time, at the seams the orchestrator names. Write a test that fails and keep its raw failing output; then write the least code that passes. Do not write all the tests first.
  2. Test behavior through the public interface, not the implementation. Take expected values from the acceptance criteria or from an independent example, never by recomputing them with the code's own algorithm.
  3. Before returning, run the project's type check or lint, the relevant tests, and the whole test suite. The commands are the adapter's `local-checks`; if there are none, use the project's own conventions and say in the return what you ran and what you did not.
  4. Check that the diff matches the ticket's scope before returning (for example `git diff --stat` shows no whole-file rewrite). Do not change a file's line-ending style; if a whole-file change appears, restore the file.
  5. Leave the changes in the worktree, uncommitted.
- Must not: push, open a PR, `git add`, commit, touch a path in `forbidden-files`, refactor in this loop (that is left to the review phase), or run its own two-axis review (independent review is the orchestrator's job).
- Questions: on an ambiguity (a ticket reference or a requirement is unclear) the worker guesses nothing and writes no file; its final reply for the turn is `QUESTION: <one question>`. The orchestrator answers from the Spec (`SKILL.md`, Decisions for workers) or marks the ticket `blocked`; the answer goes to the same implementer through the executor's follow-up message, and the worker continues in the same context. Answering is not a fix round. A ticket allows 3 questions; the fourth marks it `blocked` with the reason. A `QUESTION` is for an ambiguity with no Spec basis; the Decisions in the return are small decisions already made with a Spec basis. Seen on 2026-11-01 with a Claude sub-agent and a Codex sub-agent (Codex: `harness/codex.md`).
- Must return: the ticket title and acceptance criteria it read; the list of changed paths, one per line, as plain relative paths (no links, no absolute paths), cumulative (every path it has changed since the last commit, repeating paths from an earlier uncommitted turn); the Decisions it made; the raw failing and passing test output and the check output; for each acceptance criterion, its evidence (a test name or command output), and an explicit "no evidence" where there is none; what it could not verify.
- Done when: every acceptance criterion has evidence or an explicit "no evidence", and the changed files sit in the worktree and match the returned list; or the turn ends in a `QUESTION`.

## The orchestrator's commit

After each of the implementer's turns that ends in changes (the first one and every fix round) the orchestrator commits them. The commit is the hand-over object: reviewers, `patch-id`, `ready to merge` and the PR work from it.

0. When a worktree is created and `install` has finished, record its changed paths once as the baseline (`install` may change a lockfile): `reference/path-check.md`. Record `git -C "<worktree>" rev-parse HEAD` as well, and again after each commit.
1. First check that `git -C "<worktree>" rev-parse HEAD` equals the recorded value; a different value means the worker committed by itself, which is an unexpected change (`reference/unsandboxed-workers.md`). Then drop from the returned list the forbidden paths that step 2 has already restored or deleted. Then compare the worktree's changed paths with the list as `reference/path-check.md` says. A path in only one of them goes back to the implementer as one question. This mechanical question is not a fix round and does not count toward the 3-question limit.
2. Stage path by path: `git -C "<worktree>" add -- <path>` for each listed path that does not match `forbidden-files` (in PowerShell 5.1 a path with spaces needs quotes; not tested). A listed forbidden path is not staged, and it is named in the PR and the report; the PR uses `Refs #N`. Undo the worker's change without `git add`: a path that git tracks (`git -C "<worktree>" ls-files --error-unmatch -- <path>` succeeds) is restored with `git -C "<worktree>" restore --worktree -- <path>`; an untracked, newly created path is deleted with `git -C "<worktree>" clean -f -- <path>`; a path that is also in the baseline (for example a lockfile that `install` changed) is left as it is. Restoring or deleting is not editing a forbidden path.
3. Commit with the message `<ticket title> (#N)` and the machine's configured git identity. A fix round makes a new commit.
4. When the `pre-commit` hook fails, send its output to the implementer as one fix round; the implementer fixes the files, you clear the index with `git -C "<worktree>" reset`, and repeat steps 1 to 3.
5. Merge commit (Prepare): first restore each tracked path of the baseline (a status code other than `??`) with `git -C "<worktree>" restore --worktree -- <path>`, because an uncommitted lockfile blocks the merge ("Please commit your changes or stash them"). Then run `git -C "<worktree>" merge origin/<default>`. On a conflict the implementer, as one fix round (with none left the ticket is `blocked` with the reason "merge conflict, no fix round left"), resolves the markers in the conflicted files only and runs no git; you `git add` the resolved paths and finish with `git commit --no-edit`. In a merge state skip step 1: the status then lists the upstream paths. After the merge (and the conflict wrap-up) run the adapter's `install` again and record the baseline again.

## A worker's refusal or error

When a worker reports a permission refusal or an error, write its text word for word in the ledger and the report, without summarizing it. In one complete real run (2026-11-03) a compound command was refused whole by a deny rule, and the summary "git status refused" misled the operator.

## Heads of a PR

The reviewed head is the SHA that the last review returning green on both axes covered. The final head is the SHA after Prepare has merged `origin/<default>` and pushed. The `patch-id` comparison uses these two heads. When it differs, the reviewers' worktrees move to the final head and the re-check is done there.

## The preflight probe file

The probe worker writes `probe.txt` with the single line `probe` in the probe worktree and replies when done. A byte-order mark at the start of the file (Codex writes one) does not affect the pass. The orchestrator checks that the file exists, stages and commits it (`SKILL.md`, preflight step 2). Send the probe worker's commands one at a time. Checking the git remote or `gh auth` is a preflight check of the orchestrator itself (`SKILL.md`, preflight step 2), not a task for the worker.

## A rule that changes during a run

A PR opened under the rule in force at that time keeps that rule. Before its merge gate, bring it up to the current rule's pre-merge checks: Prepare, the `patch-id` comparison, and a `ready to merge` comment with the full hash. Commits already made are not redone and get no path check. One ledger comment says that the PR keeps the earlier rule and which checks were added.

## Evidence only CI can give

An acceptance item that only CI can prove (for example "20 CI runs", cache hit and miss logs, run numbers) is produced by the orchestrator, not the implementer: open a Draft PR that changes workflows only and triggers the runs, link the runs in the body of the PR under review, then close the Draft PR and delete its branch. The Draft PR never merges. It is the one exception to `forbidden-files` (it may edit a workflow), and only for the orchestrator and only for evidence that CI alone can give. A claim that a change removes a flaky test needs a control run on the unfixed code, and the control must be able to go red. If it cannot, the PR says the evidence cannot tell fixed from unfixed and uses `Refs #N`, not `Closes #N`.

## Review skill (role `review`)

- Input: `<base>`, the merge-base of the PR branch and the default branch.
- Must: review the diff since `<base>` along two axes, done by two mutually independent read-only agents, one per axis: **Standards** (the repo's coding standards) and **Spec** (does the code do what the ticket asked). Who spawns them is the executor's business.
- Must return, per axis: `green`, or a list of findings. Each finding states a severity label (the reviewer's choice, for example P1 to P3), its trigger condition, and who is affected. The two axes stay in separate, labelled reports.
- Judge the committed content, not a working copy: read files with `git show <ref>:<path>` and the scope of the change with `gh pr diff`, because line-ending conversion at checkout produces false findings. Each finding says how it was confirmed.
- Done when: both axes have returned.

## PR skill (role `pr`)

- Input: the committed branch and its ticket.
- Must produce: a PR body with three sections: **Summary**, **Evidence**, **Merge danger**. The orchestrator puts `Closes #N` (or `Refs #N`) on line 1.
- Done when: the body text is returned. The skill need not open the PR.
