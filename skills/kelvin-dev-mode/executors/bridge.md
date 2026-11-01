# Executor: bridge (the orchestrator talks to workers in another application)

This file is an **executor interface**. The orchestrator in `SKILL.md` talks to workers only through the actions below: five required, and *close worker*, which is optional. Any bridge that provides the five required ones can back `executor: bridge`.

The bridge's own usage rules (how results are announced, delivery failures, sandbox limits) live in its docs, reached through the adapter's `executor-notes`. Read them once per run and follow them where they differ from this file. The project's `dispatch-doc` adds ticket-specific rules.

## The actions

| Action | Contract |
|---|---|
| create worktree | The orchestrator, not the worker, creates it as in `SKILL.md` Loop step 1: baseline `origin/<default>`, a detached checkout of the PR head for review, then the adapter's `install`. A worker's sandbox may not write `.git`. |
| start worker | Starts a new worker with a name and a one-line greeting in a saved project at the given working directory. Returns a worker id. |
| send task | Sends a message to a worker id. Returns a message id that the ledger records. |
| read result | Given a message id or worker id, returns the worker's reply or its current state. |
| close worker | Optional. Ends the worker; its worktree is removed afterwards. Without it: see Checks and limits, Closing. |
| query status | Says whether a worker is working, waiting or gone, and whether a message was delivered. Also lists the replies that exist but have not been read. A bridge without that list: read each message id recorded in the ledger instead. |

If a bridge lacks one of the five required actions, the run refuses to start and names the action.

## Preflight

The project must be saved in the worker application (starting a worker accepts saved projects only). The preflight worker probe (`SKILL.md`, preflight step 2) closes its worker with *close worker*; a bridge without it: see Checks and limits, Closing.

## Workers

Each ticket gets two workers, both new, both closed as `SKILL.md` Loop steps 6 and 8 say (where the bridge can close): an **implementation worker** (only implements) and a **review worker** (only reviews). A worker never serves a second ticket or the other role.

Start a worker in two steps: *start worker* with a one-line greeting, then the task with *send task*, so the task carries a message id. Name the worker `<project> · <ticket and short name> · implement|review · <date>` in the report language; the bridge may allow naming only at start.

## Implementation message

1. Create the worktree first (above).
2. Line 1: the `implement` role's line with the ticket number (`reference/contracts.md`, Prompt line), written the way the worker's application calls skills (`harness/`; for example `$implement #<N>`).
3. Body: worktree path, branch, migration number to use if the project asks for one, the **seams** (public interfaces to test, one line each), risks specific to the ticket, the `forbidden-files` list. The repo name and ticket number replace ticket text: the worker reads the ticket itself.
4. Include, adapted to how the worker reaches GitHub (this text assumes a GitHub connector): "Read tickets through the GitHub connector with its read tools only (it needs no token); if a read fails, report the error text. Change files in the worktree and leave the commit, the push and the PR to the orchestrator (`reference/contracts.md`). Write the PR body with the PR skill and return it, the changed paths, your Decisions, and what you could not verify (with `pr-body: orchestrator`, return the Decisions and the not-verified list only). Run only commands that keep environment variables out of the output. If any output contains a secret, name it in the report and paste nothing of its value. On an ambiguity with no Spec basis your final reply is `QUESTION: <one question>` (`reference/contracts.md`). Open your report with the ticket title and acceptance criteria you read."
5. Compare the title and acceptance criteria in the report with the live ticket before going on.

## Review message

One review worker per ticket. Line 1: the `review` skill with the base, written the same way (for example `$<review skill> <base>`), where `<base>` is the merge-base of the PR branch and the default branch. Body: worktree path of the detached checkout, repo and ticket number, and "open two sub-agents, Standards and Spec; return the two reports separately and labelled; read-only: read files and run read-only git; no writes, no database". The review worker only returns the reports; you post them under the PR as two separate comments headed `Standards` and `Spec`.

## Fix rounds

Send the findings to the same implementation worker, then the same review worker re-checks (see `SKILL.md`, Loop step 5). Paste CI log text to the worker verbatim; do not reason out the cause for it.

## Checks and limits

- The Tick in `SKILL.md` decides when to check. Each tick, use *query status* for unread replies and for each worker's progress; probe progress by side effects, not by the status text alone. A bridge's result notifications can fail to arrive, so the tick, not the notice, is what guarantees a reply is read.
- What counts toward `max-active`: `reference/adapter-fields.md`.
- A finding that depends on live environment state is verified with a read-only query before any fix.
- Closing: when `SKILL.md` Loop step 6 or 8 closes a ticket's workers, tell the implementation worker to remove its worktree, or remove it yourself with `git worktree remove`; do this before the freed slot is reused. A worker is closed with *close worker*. Without that action: stop sending it messages, end any turn it has in progress with the bridge's interrupt (where the bridge has one), and remove its worktree. An action that stops the whole worker application ends every worker, so it is not one worker's close. The morning report lists each thread left unclosed.
- A replacement for a stuck worker (Tick step 3) is a new worker, started as Tick step 3 says.
