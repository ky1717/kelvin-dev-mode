# Ledger

The rules for the ledger issue. `SKILL.md` (Ledger) keeps the core and points here.

## What the ledger is

One issue per repo, found by label `dev-mode-ledger`, never by title; only when no issue has the label, create one titled `kelvin-dev-mode ledger`.

Every state change appends one comment: ticket, state (exactly one of `queued`, `not dispatched` + reason, `implementing`, `reviewing`, `fixing round N`, `ready to merge`, `held at gate` + reason, `merged`, `cleaned`, `blocked` + reason), worker, PR number, head SHA after any push or commit of yours, UTC time. The worker field holds, for each worker of the ticket, the identifiers that the executor file's *start worker* and *send task* rows return or say to record (`executors/`); the name where a harness takes names.

Only you write it; read the clock before you write the UTC time, never an estimate; append before acting. The newest comment per ticket is its state, so a new session resumes from the ledger alone. Name tickets and PRs the way the instruction file asks.

A fact that exists only after an action (a session id, a process id, a PR number, a head SHA) goes in the comment right after the action. The comment before the action says "before start" (or "before push") in place of that fact.

A question that you answer for the operator goes in the ticket's next state comment as `(question N of 3 answered: <one sentence>)`. It adds no state value.

## A worker's record under `cli`

Under `cli` the rule above gives a worker's record two comments:

- Before the start: an `implementing` comment that names the run folder and the file names to be used.
- Right after the start: the next comment holds the record of that process, in the format of `executors/cli.md` (What the ledger records), one per process.

A restarted orchestrator that finds a worker with only the first comment looks in the run folder, at the file with the process id and at the event file, and trusts a process id only after its start time matches (`executors/cli.md`).

## Run-level comments

Run-level comments: `run start`, `paused`, `stopped`, `resumed`, and the morning report. `resumed` clears `paused` or `stopped` inside a run that has no morning report yet.

When the newest run-level comment is a morning report and the operator asks to run again, that is a new run: full preflight, a new queue (older states are history; a ticket that is closed, whose PR has merged, or whose PR is still open is not queued; one whose PR the operator closed unmerged can be queued again), a new `run start`. Anything that parses the ledger must recognise these.

## The ready to merge comment

In `auto`, a `ready to merge` comment holds two values: the result of the `git patch-id --stable` comparison (same or different) and the full 40-character hash of `origin/<default>` that the branch contains (`git rev-parse origin/<default>`; a short hash makes the comment invalid); a comment lacking either is invalid, and that PR does not merge.
