# Ledger

The rules for the ledger issue. `SKILL.md` (Ledger) keeps the core and points here.

## What the ledger is

One issue per repo, found by label `dev-mode-ledger`, never by title; only when no issue has the label, create one titled `kelvin-dev-mode ledger`.

Every state change appends one comment: ticket, state (exactly one of `queued`, `not dispatched` + reason, `implementing`, `reviewing`, `fixing round N`, `ready to merge`, `held at gate` + reason, `merged`, `cleaned`, `blocked` + reason), worker, PR number, head SHA after any push or commit of yours, UTC time.

Only you write it; append before acting. The newest comment per ticket is its state, so a new session resumes from the ledger alone. Name tickets and PRs the way the instruction file asks.

## Run-level comments

Run-level comments: `run start`, `paused`, `stopped`, `resumed`, and the morning report. `resumed` clears `paused` or `stopped` inside a run that has no morning report yet.

When the newest run-level comment is a morning report and the operator asks to run again, that is a new run: full preflight, a new queue (older states are history; a ticket that is closed, whose PR has merged, or whose PR is still open is not queued; one whose PR the operator closed unmerged can be queued again), a new `run start`. Anything that parses the ledger must recognise these.

## The ready to merge comment

In `auto`, a `ready to merge` comment holds two values: the result of the `git patch-id --stable` comparison (same or different) and the full 40-character hash of `origin/<default>` that the branch contains (`git rev-parse origin/<default>`; a short hash makes the comment invalid); a comment lacking either is invalid, and that PR does not merge.
