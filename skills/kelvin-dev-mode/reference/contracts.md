# Contracts

Read at preflight. This file is the single source for the `list-tickets` format and for what the three roles must do, so you can supply your own.

**The contract is the baseline; a skill is an optional way to meet it.** A role with no skill in the adapter is done directly from the contract below. A **user-only skill** is a skill that only a user can start, not a model-written prompt; the marker depends on the harness (`harness/`). A worker may be unable to call one. For a named user-only skill, three steps, in order:

- (a) The worker calls the skill if it can.
- (b) Otherwise, if the skill's file is readable, the worker reads that file (and any reference skill it names, such as `tdd`) and follows the steps that apply. Where a step conflicts with the contract, the contract wins: no push, no PR, no `forbidden-files` path, and skip any self-review step that would duplicate the independent review (for example a final two-axis review inside `implement`); the orchestrator arranges the independent review.
- (c) If the file cannot be read either, the worker works directly from the contract below, including the quality baseline for `implement`.

**Prompt line.** The orchestrator's line 1 for a role with a skill is the call, written the way the worker's application calls skills (`harness/`), followed by "if you cannot call it, take steps (b) and (c)"; for a role with no skill it says to work from the contract below.

## list-tickets

The adapter's `list-tickets` command prints one line per open ticket: the ticket number (a `#` before it is fine), then `READY` or `BLOCKED`, then the title, separated by tabs. `BLOCKED` means the ticket has at least one **open** blocker. Preflight runs the command once and refuses the run if any line does not give a number and `READY`/`BLOCKED`.

Runnable example (bash; lists issues labelled `ready-for-agent`, using GitHub's native issue dependencies, the `blockedBy` field):

```bash
gh issue list --state open --label ready-for-agent --limit 200 \
  --json number,title,blockedBy \
  --jq '.[] | "#\(.number)\t\(if ([.blockedBy.nodes[] | select(.state == "OPEN")] | length) > 0 then "BLOCKED" else "READY" end)\t\(.title)"'
```

The example was run with `gh` 2.103.2; other versions are untested, so check that `gh issue list --json blockedBy` works in yours. In PowerShell, quote the `--jq` expression differently.

## Implementation skill (role `implement`)

This is the quality baseline for the implementer, whether or not a skill is named.

- Input: a ticket number. The working directory is a prepared git worktree. How the ticket is read is the executor's business: a bridge worker uses its GitHub connector, a sub-agent uses `gh`.
- Before starting: read the ticket and state its title; read the project's `GLOSSARY.md`, ADRs, coding-standards documents and the like where they exist, and skip them where they do not. If a skill named `tdd` is installed and the worker can call it, call it for the work below; otherwise follow the next list.
- Must (when working from the contract):
  1. Test first, one vertical slice at a time, at the seams the orchestrator names. Write a test that fails and keep its raw failing output; then write the least code that passes. Do not write all the tests first.
  2. Test behavior through the public interface, not the implementation. Take expected values from the acceptance criteria or from an independent example, never by recomputing them with the code's own algorithm.
  3. Before committing, run the project's type check or lint, the relevant tests, and the whole test suite. The commands are the adapter's `local-checks`; if there are none, use the project's own conventions and say in the return what you ran and what you did not.
  4. Check that the diff matches the ticket's scope before committing (for example `git diff --stat` shows no whole-file rewrite). Do not change a file's line-ending style; if a whole-file change appears, restore the file and commit again.
  5. Commit in the worktree.
- Must not: push, open a PR, touch a path in `forbidden-files`, refactor in this loop (that is left to the review phase), or run its own two-axis review (independent review is the orchestrator's job).
- Questions: on an ambiguity (a ticket reference or a requirement is unclear) the worker guesses nothing and writes no file; its final reply for the turn is `QUESTION: <one question>`. The orchestrator answers from the Spec (`SKILL.md`, Decisions for workers) or marks the ticket `blocked`; the answer goes to the same implementer through the executor's follow-up message, and the worker continues in the same context. Answering is not a fix round. A ticket allows 3 questions; the fourth marks it `blocked` with the reason. A `QUESTION` is for an ambiguity with no Spec basis; the Decisions in the return are small decisions already made with a Spec basis. Seen on 2026-11-01 with a Claude sub-agent and a Codex sub-agent (Codex: `harness/codex.md`).
- Must return: the ticket title and acceptance criteria it read; the Decisions it made; the raw failing and passing test output and the check output; for each acceptance criterion, its evidence (a test name or command output), and an explicit "no evidence" where there is none; what it could not verify.
- Done when: every acceptance criterion has evidence or an explicit "no evidence", and the commit exists; or the turn ends in a `QUESTION`.

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
