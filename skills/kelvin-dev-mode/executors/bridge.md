# Executor: bridge (the orchestrator talks to workers in another application)

This file is an **executor interface**. The orchestrator in `SKILL.md` talks to workers only through the actions below: five required, and *close worker*, which is optional. Any bridge that provides the five required ones can back `executor: bridge`. It works in both directions: a Claude orchestrator with Codex workers, or a Codex orchestrator with Claude workers.

The bridge's own usage rules (how results are announced, delivery failures, sandbox limits) live in its docs, reached through the adapter's `executor-notes`. Read them once per run and follow them where they differ from this file. The project's `dispatch-doc` adds ticket-specific rules.

## The actions

| Action | Contract |
|---|---|
| create worktree | The orchestrator, not the worker, creates it as in `SKILL.md` Loop step 1: baseline `origin/<default>`, a detached checkout of the PR head for review, then the adapter's `install`. A worker may be unable to write `.git`. |
| start worker | Starts a new worker with a name and a one-line greeting in a saved project at the given working directory. Returns a worker id. |
| send task | Sends a message to a worker id. Returns a message id that the ledger records. |
| read result | Given a message id or worker id, returns the worker's reply or its current state. |
| close worker | Optional. Ends the worker; its worktree is removed afterwards. Without it: see Checks and limits, Closing. |
| query status | Says whether a worker is working, waiting or gone, and whether a message was delivered. Also lists the replies that exist but have not been read. A bridge without that list: read each message id recorded in the ledger instead. |

If a bridge lacks one of the five required actions, the run refuses to start and names the action.

## Tools by direction

The tool names, parameters and returns of the bridge used in testing are in `harness/`: `harness/claude-code.md` for a Claude orchestrator with Codex workers, `harness/codex.md` for a Codex orchestrator with Claude workers. Another bridge maps its own tools to the five actions.

## Sending to a Claude worker

Measured on the bridge used in testing, 2026-11-03; the same limits for a Codex worker are not recorded.

- A worker has one open message at a time: while a message awaits its final reply, every further send to that worker is refused. A stuck worker therefore cannot be nudged, and a fix round or the answer to a `QUESTION` waits for the previous final reply.
- A receipt proves only that the message reached the worker's inbox; the final reply is a separate item matched by message id. A message with a receipt and no final reply is delivered and not yet handed in, working or stuck (judge it by side effects: `SKILL.md`, Tick step 3).
- A message that is `not_delivered` (the recipient was not live, the bridge was busy, or a preflight failed) may be judged undelivered and sent again. A message that is `unknown`, or whose id the delivery read does not find, proves nothing about delivery and is never sent again.
- The opening message that starts a worker is a short opener: it states the role only, expects no reply, does not count as an open message, and does not start with `/`. Neither your opener to a worker nor the opening text of the orchestrator's own thread asks anyone to wait for a task that carries a message id. The id is how agents on the bridge match a reply to a task; a task that the operator gives by hand in the orchestrator's thread has no id and is valid authorization all the same. In one complete real run (2026-11-03) the orchestrator stopped and asked for authorization because its opener had promised a task with an id.
- A Claude worker's session opens in the repository root where the orchestrator's thread runs (the allowed-root rule is in Preflight), not in a worktree: the bridge refused to send to a session whose directory was a worktree subdirectory. The message gives the worktree's absolute path and says to work only in that path. An expected working directory, if used, is the session's own directory (the repository root), not the worktree. Address the worker by its exact session id, never by "latest". This rule comes from one real preflight in a practice repository (2026-11-03): an observed behaviour, not interface documentation.
- The bridge limits the number of workers in neither direction; the only cap is the adapter's `max-active`.

## Workers without a sandbox

The orchestrator's check before and after every worker turn, and the guards that only prevent accidents, are in `reference/unsandboxed-workers.md`. Because the session's directory is the repository root, the worker's working path is limited to the worktree named in the task.

## Preflight

A Codex worker: the project is saved in the Codex application (starting a worker accepts saved projects only). A Claude worker: the session's directory (the repository root of the orchestrator's thread) lies inside one of the bridge's allowed roots, and the branch-protection check in `reference/unsandboxed-workers.md` passes. The probe message also asks the worker to read one named ticket and reply with its title, when the reading tool covers the repository's owner (otherwise the orchestrator pastes the ticket text and skips this step); a worker that cannot read it is refused with the reason, because the reading key can expire while the session still lists the tool. The preflight worker probe (`SKILL.md`, preflight step 2) closes its worker with *close worker*, and the probe file is defined in `reference/contracts.md`; with a Claude worker the probe passes only when *close worker* reports that the process exited; a bridge without it: see Checks and limits, Closing.

## Workers

Each ticket gets two workers, both new, both closed as `SKILL.md` Loop steps 6 and 8 say (where the bridge can close): an **implementation worker** (only implements) and a **review worker** (only reviews). A worker never serves a second ticket or the other role.

Start a worker in two steps: *start worker* with a one-line greeting, then the task with *send task*, so the task carries a message id. Name the worker `<project> · <ticket and short name> · implement|review · <date>` in the report language; the bridge may allow naming only at start.

## Implementation message

1. Create the worktree first (above).
2. Line 1: the `implement` role's line with the ticket number (`reference/contracts.md`, Prompt line), written the way the worker's application calls skills (`harness/`; for example `$implement #<N>`).
3. Body: worktree path, branch, migration number to use if the project asks for one, the **seams** (public interfaces to test, one line each), risks specific to the ticket, the `forbidden-files` list. The repo name and ticket number replace ticket text: the worker reads the ticket itself, except where it cannot (Reading tickets).
4. Include: "Read tickets with the tool named in this message (see Reading tickets); if a read fails, report the error text. Change files in the worktree and leave the commit, the push and the PR to the orchestrator (`reference/contracts.md`). Write the PR body with the PR skill and return it, the changed paths, your Decisions, and what you could not verify (with `pr-body: orchestrator`, return the Decisions and the not-verified list only). Run only commands that keep environment variables out of the output. If any output contains a secret, name it in the report and paste nothing of its value. On an ambiguity with no Spec basis your final reply is `QUESTION: <one question>` (`reference/contracts.md`). When the ticket needs a new dependency, your final reply is `QUESTION: needs dependency <name@version>`. Open your report with the ticket title and acceptance criteria you read."
5. Compare the title and acceptance criteria in the report with the live ticket before going on.
6. A worker whose sandbox has no network (a Codex worker in a bridge-created task: `harness/codex.md`) cannot install packages, so a new dependency goes through the orchestrator. The reply `QUESTION: needs dependency <name@version>` is the worker's final reply. Install the package in the worktree with the project's package manager, in the way the adapter's `install` command installs dependencies; write the name, the exact version and the size in the ticket or the report; add the changed paths of the installation (`package.json`, the lockfile) to the worker's returned list, so that the path check expects them and your commit includes them (`reference/path-check.md`); then send the worker a new message in the same session.

## Reading tickets

- A Codex worker reads through the GitHub connector with its read tools only (it needs no token).
- A Claude worker reads with the read-only GitHub tool installed in its session, (its name: `harness/codex.md`). It reads Issues only: pull requests and file contents are not readable, and only the owners the tool was set up for are covered. The message gives the repository name and the ticket number and names the tool (not `gh`); for a pull request it gives a local diff or the worktree path. The ticket of an uncovered repository cannot be read by the worker: paste the ticket text into the message.
- The tool name belongs to the machine's setup. The preflight confirms that a Claude worker session lists its reading tool.

## Review message

One review worker per ticket. Line 1: the `review` skill with the base, written the same way (for example `$<review skill> <base>`), where `<base>` is the merge-base of the PR branch and the default branch. Body: worktree path of the detached checkout, repo and ticket number, and "open two sub-agents, Standards and Spec; return the two reports separately and labelled; read-only: read files and run read-only git; no writes, no database". The review worker only returns the reports; you post them under the PR as two separate comments headed `Standards` and `Spec`.

## Fix rounds

Send the findings to the same implementation worker, then the same review worker re-checks (see `SKILL.md`, Loop step 5). Paste CI log text to the worker verbatim; do not reason out the cause for it.

## Checks and limits

- The Tick in `SKILL.md` decides when to check. Each tick, use *query status* for unread replies and for each worker's progress; probe progress by side effects, not by the status text alone. A bridge's result notifications can fail to arrive, so the tick, not the notice, is what guarantees a reply is read.
- What counts toward `max-active`: `reference/adapter-fields.md`.
- A finding that depends on live environment state is verified with a read-only query before any fix.
- Closing: when `SKILL.md` Loop step 6 or 8 closes a ticket's workers, tell the implementation worker to remove its worktree, or remove it yourself with `git worktree remove`; do this before the freed slot is reused. A worker is closed with *close worker*. Without that action: stop sending it messages, end any turn it has in progress with the bridge's interrupt (where the bridge has one), and remove its worktree when the ticket is closed (a replacement worker keeps the worktree: `SKILL.md`, Tick step 3). An action that stops the whole worker application ends every worker, so it is not one worker's close. The morning report lists each thread left unclosed. When a close reply says the process is still running, treat the worker as still running: it keeps counting toward `max-active`, no replacement worker starts in its worktree (the old process may still write there), and the next tick closes it again. If two tries leave it running, mark the ticket `blocked` with the reason, and list the session and its worktree in the morning report and to the operator.
- A replacement for a stuck worker (Tick step 3) is a new worker, started as Tick step 3 says.
