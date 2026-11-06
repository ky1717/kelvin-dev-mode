# Executor: cli (the orchestrator starts the other vendor's command-line program)

This file is an **executor interface**. Under `executor: cli` the orchestrator starts each worker turn as an operating-system process of the other vendor's command-line program, and resumes the same session by its id for the next turn. There is no bridge and no opener. The program names, options, paths and Windows details are in `harness/` (`harness/codex.md`, "Command-line route", for Codex workers started by a Claude orchestrator).

## The actions

| Action | Contract |
|---|---|
| create worktree | The orchestrator creates it as in `SKILL.md` Loop step 1: baseline `origin/<default>`, a detached checkout of the PR head for review, then the adapter's `install`; it records the baseline and `HEAD` (`reference/contracts.md`, The orchestrator's commit). |
| start worker | Starts a headless process whose working directory is the worktree. The task is the prompt and goes in through the process's standard input, because a long prompt does not fit on a command line. The orchestrator sets the permissions when it starts the process (the implementer: full access, `reference/unsandboxed-workers.md`; a review process: read-only) and gives the process placeholders for credentials. With `worker-speed: fast` it also hands the program the Fast tier setting. The start returns at once (see What the ledger records). It records what "What the ledger records" below lists. |
| send task | Starts a new process that resumes the same session with the next message. The process is started with the worktree as its working directory, because a resumed process runs its commands in the folder it was started in, not in the session's old directory (`harness/codex.md`). Check the session id first (see Checks and limits). A resumed session keeps its tier, so handing the Fast tier setting again is harmless; do it to be safe (`harness/codex.md`). A resumed session does not keep full access, so every permission setting is passed again on every resume, read-only for a review process as well (`harness/codex.md` says why). |
| read result | Waits for the process to exit, then reads the result file (the worker's final message) and the exit code. A killed or crashed process leaves no result file; the event output it wrote keeps the session id. |
| query status | A recorded process is alive only when its process id and its start time both match the running process (`harness/codex.md` says how to read them); any difference means gone, because the system may have given the id to another program. The result file exists or not; the event output grows or not (a side effect: `SKILL.md`, Tick step 3). A long command can leave the output unchanged for a while, so the stuck time is generous and is judged together with other side effects. |
| close worker | After the same match of process id and start time, kills the process and everything it started, and checks that nothing is left; a process that has already exited by itself is only recorded; a recorded process whose id and start time do not match counts as gone, is only recorded, and is never killed. A worker between turns has no process, so there is nothing to close; its session stays resumable. |

## What the ledger records

Every process gets one record in the worker field of a ledger comment, written right after the process starts: each implementer turn, each reviewer, and the probe. The format is an example, not a field:

`worker: <role> session <id> pid <n> start <ISO time> result <path> events <path>`

The start returns at once, and the process id and start time of the record are those of the root process that the start returns. A start that waits until the process ends returns neither, so it is not used; the orchestrator waits by polling for the result file or for the process to exit (`harness/codex.md` has the tested start).

Right after the start the orchestrator also writes the process id and the start time to a file in that folder. Both files live in one folder per run, chosen by the orchestrator outside `worktree-dir` and outside every worktree: a file inside a worktree changes its status every turn, and the orchestrator's check reports that as an unexpected change. The folder follows from the recorded paths.

## Workers

Each ticket gets an **implementer** (only implements) and, for review, **two independent review processes**, one per axis: **Standards** and **Spec**. The implementer runs with full access; a review process starts read-only, so it cannot change files, and passes the read-only setting again on every resume. All are fresh, and you start each of them. No worker serves a second ticket or another role.

- **Model.** It comes from the adapter's `models`; unset, the preflight resolves the program's default model and pins it for the run. The pinned id is passed on every start and every resume, because a resumed session otherwise takes whatever the default is then (`harness/codex.md`). The reasoning level is the program's own default: the start leaves its option out (`harness/codex.md`).
- **One open message per worker** holds by construction: you start a worker's next process only after its previous process has exited, and never two at once.
- **Questions.** A final message that starts with `QUESTION:` is read from the result file; you answer by resuming the session with a new message. The limit of three questions and how they are counted: `reference/contracts.md`. With full access the worker installs its own dependencies, so the dependency route of `executors/bridge.md` (Implementation message, item 6) does not apply.
- **Implementation message.** It is that of `executors/bridge.md` (Implementation message) with these differences, item by item:
  - The changed paths are returned as `reference/contracts.md` says (plain relative paths), in the form of the example under item 4 of `executors/bridge.md`.
  - Items 1 (create the worktree), 3 (body) and 5 (compare the report with the ticket): unchanged.
  - Item 2 (line 1): the role's line is written the way the Codex application calls skills (`harness/codex.md`).
  - Item 4: leave out the sentence on `QUESTION: needs dependency`, and read tickets as the Reading tickets bullet below says, not "with the tool named in this message".
  - Item 6 (the dependency route): does not apply.
  - The worker returns its results in the final message and you post what belongs on GitHub, because its token is a placeholder; an instruction to post a comment on the ticket (as in `executors/subagent.md`) does not apply.
- **Review prompt.** That of `executors/subagent.md` (Messages, Reviewer independence); it also says that the process is read-only. The review skill's `gh pr diff` (`reference/contracts.md`) cannot run with a placeholder token, so the prompt says: read the change with `git diff <base> HEAD` and the committed content with `git show <ref>:<path>`.
- **Reading tickets.** The worker reads through the GitHub connector of its application. The connector's tool name and parameters are in `harness/codex.md`; the read has worked in a command-line process. The message gives the repository name and the ticket number and says: if the read fails, report the error text. After a failed read, paste the ticket text into the next message. You push the branch and post the reports yourself, as in the other executors.

## Preflight

- The worker program is found by the selection rule in `harness/codex.md` and runs; the screen shows its path and version. With none, the run refuses.
- With `worker-speed: fast`, the preflight reads whether the pinned model offers the Fast tier (`harness/codex.md` says where) and shows the answer on the screen. A model without it runs at standard speed and gives no error, so the absence of an error proves nothing; the run goes on at standard speed.
- The model is pinned in the preflight:
  - The probe starts with the model the run will use: the written id when `models` has one; with `models` unset, without a model, so that the program shows its default.
  - The preflight reads which model the probe used and pins that id. If it cannot be read, it falls back as `harness/codex.md` says.
  - The `run start` comment records the pinned id of each role and where it came from, the adapter or the default.
  - A `models` value that is a series name and not a full id is refused, and the refusal lists the visible ids of that series.
  - A value that is not in the program's model list gets a warning, and the probe decides: if the program accepts it, the list is stale; if it rejects it, the run is refused with the rejection text word for word.
  - A value that is hidden in the list is passed with a warning.
  - The screen shows one line for the model: `Codex model: <id> (program default, pinned for this run)` or `Codex model: <id> (adapter; ...)`, with the list's fetch time and age, the client version, and the program's default reasoning level for that id. The part after `adapter;` is one of: `equals the program default`; `the program default is <default id> and is not used`; `not in the program list fetched <time>`; `hidden in the program list`. A refusal reads `Refused: models value "<value>" is not a full model id; visible: <ids>`.
- A worker without a sandbox needs the branch-protection check of `reference/unsandboxed-workers.md` (Preflight). No login file is copied and no command rules file is written by default.
- The probe is that of `SKILL.md`, preflight step 2, with these differences: there is no greeting (the first process carries the task); the process ends by itself, so there is no close action; the probe's process gets a record like any other (What the ledger records), and the probe passes only when that process has exited and nothing of it remains, judged from the recorded process id and its descendants, not from process names (the machine runs Codex Desktop's own `codex.exe` processes); the message also asks the worker to read the first ticket of the queue and reply with its title, and with an empty queue that part is skipped.

## Checks and limits

- A worker counts toward `max-active` while a turn is in progress or its result is unread, as for the other executors (`reference/adapter-fields.md`).
- The orchestrator's check before and after every worker turn is the check at the start and at the exit of a process (`reference/unsandboxed-workers.md`).
- Before a resume, check that the session id is a non-empty UUID. An empty id starts a new session with no task. If it is empty, stop, record it in the ledger, and start nothing.
- After every process, read the model that it used and look in its event stream for the program's warning that a resumed session changed model (`harness/codex.md`). A model other than the pinned one is an unexpected change: the ticket is `blocked`, and the program's text is recorded word for word.
- Closing: when `SKILL.md` Loop step 6 or 8 closes a ticket's workers, close each process still alive as the close worker row says (the match of process id and start time comes first), and remove the worktree. A replacement for a stuck worker is a new session in the same worktree, started as Tick step 3 says, after the old process is killed.
- The orchestrator's own wake-up tool comes from its harness (`SKILL.md`, preflight step 3); it is not part of this file.
