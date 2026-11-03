# Harness notes: Codex

Read when the orchestrator is a Codex session. The facts come in three kinds, and each section names its source: an interactive run, small toy tests, and Codex's own statement (untested; nobody here has checked it). Nothing here shows that an unattended Codex run works.

| Verb in the skill | In Codex |
|---|---|
| Call a skill | `$name`; observed with `$kelvin-dev-mode`. Whether `/name` works is unknown (below). |
| User-only skill marker | `agents/openai.yaml` (untested; below). |
| Wake-up tool | The automation heartbeat (untested; below). |
| Background task | A long command polled with `write_stdin` (untested; below). |
| Spawn worker / sub-agent | `collaboration.spawn_agent` (observed). |
| Wait for a worker | `collaboration.wait_agent` (observed). |
| Follow-up message to a worker | `collaboration.followup_task` (observed). |
| List workers | `collaboration.list_agents` (name seen in the toy test below). |
| Interrupt | `collaboration.interrupt_agent`: ends the agent's current turn only (toy test below). |
| Close worker | No such action exists (toy test below): see `executors/subagent.md` (Limits). |

## Observed (one run, 2026-11-01)

One interactive run in a toy repository: Codex as orchestrator, the `subagent` executor, `merge: auto`, three tickets, about ten minutes, Full Access, an operator present.

- Calling the skill as `$kelvin-dev-mode` worked. The frontmatter key `disable-model-invocation: true`, which is a Claude Code setting, caused no error.
- The adapter was found through the pointer in `AGENTS.md`; `CLAUDE.md` was checked as well.
- Under the default sandbox the preflight failed closed: no usable credentials and no connection to the GitHub API, so the run refused. After the operator switched to Full Access the preflight passed (connection, ledger, a temporary worktree, the push probe). Under Full Access `gh`, `git push` and `git worktree add` all worked. Full Access decided what the orchestrator may do; the orchestrator still showed the queue and settings and waited for the operator's "go".
- Sub-agents worked: the tools it used were `collaboration.spawn_agent`, `collaboration.wait_agent` and `collaboration.followup_task`. Per ticket there was one implementer and two first-level review agents (Standards and Spec). A fix round was sent to the same implementer with `followup_task`, and it caught a real defect (numbers and punctuation inside words were treated as letters). With four slots, `max-active: 1` worked. The default `max-active: 3` was not tried.
- The run set no wake-up. It finished inside a single turn by blocking on `wait_agent`. No automation tool and no `write_stdin` polling was used.
- Merged work branches were deleted from the remote after the merge. Branch names were given a `codex/` prefix by the orchestrator.
- The preflight screen did not say whether the run could be unattended or which wake-up tool it had found, which the skill's preflight asks for.
- A separate manual trial by the operator in Codex Desktop (not a dev-mode run; Codex's own report as relayed, not re-checked here): a sub-agent had no Skill-call entry, so an explicit `$implement` call was not verified. Reading `implement`'s SKILL.md and following it worked: the test failed first, then the feature was implemented on its own branch; the `tdd` and review skill files were read and their applicable parts followed; two parallel read-only sub-agents then reviewed Standards and Spec with no findings, so a sub-agent opened sub-agents in this trial (how many slots that used was not tested); the work was committed to a local branch and not pushed; the full suite (40 tests) passed and the main agent re-ran it to confirm. The type check was skipped because no checker was installed. Codex also noted that `implement` asks for its own two-axis review, which must be reconciled with the independent review (the contract's three steps skip such a self-review).
- Windows: passing a `--jq` expression through a `.cmd` wrapper script split the quotes (the error read `'{body' is not recognized`). Write the output to a file, or parse the JSON in the shell, instead of passing the expression through the wrapper.

## Observed (toy test: questions and interrupt, 2026-11-01)

Codex as orchestrator in a toy directory, one test; the facts below are what that test showed, and nothing is inferred from them.

- Tool names: `collaboration.spawn_agent`, `collaboration.wait_agent`, `collaboration.followup_task`, `collaboration.interrupt_agent`, `collaboration.list_agents`.
- Question, then continue: a sub-agent's first turn returned `QUESTION: ...` as agreed and wrote no file. After `followup_task` sent the answer to the same sub-agent, it finished in the same context and replied `DONE`. No new agent was started.
- There is no action that closes or deletes an agent. `interrupt_agent` ends only the current turn; called on an agent that had completed, it still showed `completed` afterwards and the agent still accepted messages and further tasks. Whether a completed agent still takes one of Codex's thread slots: see the section on many sub-agents below.
- The sandbox mode was observed as `workspace-write` and the approval policy as `auto_review` (what that value means was not checked). Codex stated both values directly when asked. The test triggered no approval prompt and no sandbox denial.

## Observed (toy test: many sub-agents, 2026-11-01)

Codex as orchestrator in a toy directory; it started 8 sub-agents one after another, then a ninth, and closed none. The facts below are what that test showed.

- All nine sub-agents started without an error, a sandbox denial or an approval request. A completed agent that was still open did not stop a new agent from starting.
- After each completion `collaboration.list_agents` showed 4 entries in total, the orchestrator included: the root (running) and the 3 most recently completed agents. Earlier completed agents dropped out of the list one by one.
- `collaboration.interrupt_agent` on the first agent, after it had dropped out of the list, returned `{"previous_status":"not_found"}`.
- A further toy test the same day (one test, no general finding): agent A dropped out of the list after the fourth sub-agent completed. `collaboration.followup_task` to A raised no error, and A replied `alpha-7`, which its earlier task had written. An agent that has dropped out of the list still takes follow-ups and keeps its context.

## Observed (bridge-created and hand-created tasks, 2026-11-01)

Source: two probe tasks created through a bridge and two created by hand in the Codex Desktop window.

- A task that a bridge creates runs `workspace-write` with `on-request` approval. The bridge's thread-creation call has no permission parameter, so the bridge cannot ask for more.
- In that task the main repository's `.git` is read-only: `git add` in a worktree failed with `index.lock: Permission denied` under `.git/worktrees/<name>/`, in both bridge-created tasks.
- A task that a person creates by hand in the Codex Desktop window can run `danger-full-access`.
- Consequence for this skill: workers do not commit; the orchestrator does (`reference/contracts.md`, The orchestrator's commit).
- Not measured: whether tests that write temporary files also need an elevated-permission request. One session reported it; the cause was not isolated.

## Observed (one overnight run, Codex as worker, 2026-11-01)

Source: one overnight run in a real project; the bridge's receipt timestamps and the CI and deploy queries. This is one run's observation, not a guarantee: CI and deploy times depend on the project. Use: starting points for the wake-up interval while the orchestrator waits for a reply and while it waits for CI (`SKILL.md`, Tick).

- An implementation reply took about 1.5 to 5 minutes (1.5 minutes for one first reply, about 4.7 minutes for another).
- A review took about 1 minute (two cases).
- One CI round took about 9 to 12 minutes.
- The deploy to staging was still running after 11 to 14 minutes in at least some observations.

## Command-line route (`codex exec`)

Tested on 2026-11-01 on Windows with codex-cli 0.166.0-alpha.5.1; one test, not a guarantee.

- Call: `codex exec --json --skip-git-repo-check --model <id> -c model_reasoning_effort="<level>" -s workspace-write -c approval_policy="never" -C <dir> -` (the prompt arrives on standard input through the final `-`). Take `<id>` from the local `~/.codex/models_cache.json`, not from documents that name older models.
- `codex exec resume <thread id> --json ... -` returns to the same session, which remembered its earlier context; the QUESTION-and-answer flow works through it.
- Three workers in parallel finished in about 22 seconds, and memory peaked at about 1.9 GB in total. One minimal worker used about 0.6 GB. Codex Desktop alone used about 5.6 GB. This is one observation; summed working sets count shared memory more than once, so only the order of magnitude counts.
- The GitHub connector still worked (a ticket was read) while the Codex Desktop process was completely stopped. Putting `--disable apps --disable plugins` in the start arguments removes the connector.
- A command-line session does not appear in the Codex Desktop task list.
- Closing the Codex Desktop window does not stop its processes (110 stayed until they were killed).

## Writing `.git` from a Codex worker (command-line route, 2026-11-01, Windows)

Setup: `-s workspace-write` with `approval_policy="never"`.

| Layout | `git add` |
|---|---|
| a worktree outside the main repository | fails: `Unable to create '.../index.lock': Permission denied` |
| a worktree inside the main repository | fails with the same error |
| an independent clone (its own `.git` is inside the workspace) | fails with the same error |

- With the worktree's git directory plus the shared `objects`, `refs` and `logs` added as writable roots, `git add` and `git commit` succeed; the worker can then change shared branches and object data, so this is not recommended.
- `-s danger-full-access` succeeds; not recommended.
- Cause: git keeps a worktree's index under the main repository's `.git/worktrees/<name>/`, which the Codex sandbox protects. This is why workers do not commit (`reference/contracts.md`, The orchestrator's commit).
- A Windows gotcha: under `-s workspace-write`, even creating a file in a directory under an 8.3 short path (a name containing `~1`) was refused; the same work under a normal long path succeeded.

## Windows: `bash` on PATH (checked 2026-11-01)

- Typing `bash` in PowerShell or cmd runs `C:/Windows/System32/bash.exe`, the WSL launcher. When the machine's only WSL distribution is the internal one that Docker Desktop creates, that distribution has no `/bin/bash`, and the call fails with `execvpe(/bin/bash) failed: No such file or directory`.
- Git Bash (`C:/Program Files/Git/bin/bash.exe`) works, and git hooks run in it, so they are not affected.
- Write the full path to `bash` in worker prompts and project scripts; leave the user's PATH alone.

## Codex as orchestrator with the `bridge` executor (Claude workers)

Source: tests of the bridge's tools on one machine, 2026-11-03; the whole dev-mode flow has not run.

- Tools for a Claude worker, per action:

| Action | Tool |
|---|---|
| start worker | `start_claude_session(cwd, name, firstMessage, disallowedTools?, credentialMode?)`: returns `sessionId`, `pid`, process start time, `name`, `cwd`. `credentialMode` is `none` (the default) or `inherit`. |
| send task | `send_to_claude_session(target, message, expectedCwd?, expectedTaskId?, messageId?, waitFor?, waitSec?)`. `messageId` must be a UUID; `waitFor` is `receipt` (the default, inbox only) or `reply`. |
| read result | `read_bridge_message(messageId, waitSec?)`, `list_unread_results(limit?, after?)`, then `acknowledge_bridge_message(messageId, phase="reply_received")`; `read_claude_delivery(msgId)` reads a receipt. |
| query status | `list_claude_sessions(includeDead?, expectedCwd?)`; `read_claude_transcript(target, limit?)` reads without sending. |
| close worker | `close_claude_session(sessionId or pid)`: closes only sessions the calling thread started, forces the close if the session hangs, returns whether the process exited and `structuredContent.tabCloseRequested`, refuses a reused pid. |

- A Claude worker has no interrupt tool: close it directly (`close_claude_session` ends the process by force; its uncommitted changes stay in the worktree).
- The read-only ticket tool of a Claude worker is named like `mcp__github-ro-<owner>__issue_read` (one per covered owner); the rules for its use are in `executors/bridge.md`, Reading tickets.
- The orchestrator thread must be one that the operator opened by hand with Full Access: it runs `git worktree add`, staging, commits and pushes, and a thread that a bridge creates is `workspace-write` with a read-only main `.git` (the section on bridge-created and hand-created tasks, above).
- The orchestrator keeps its git and `gh` credentials; its workers start with the credential mode `none`.
- A Codex thread starts Claude sessions only after the operator has asked for it in that thread. That is a written rule for the Codex orchestrator, not a check by the bridge.
- When a worker has delivered and the orchestrator has checked the result, close the session with `close_claude_session`; an idle session still holds memory. Read `structuredContent.tabCloseRequested` in the reply: `false`, or a worker that crashed, leaves the terminal tab open for the operator to close by hand, and the morning report lists such a tab.

## Still Codex's own statement, untested

These come from a capability probe in which Codex described itself.

- Wake-up tool: Codex Desktop has an automation "heartbeat" (`automation_update`) that sends an instruction back into the current local chat on a schedule. Whether it reloads a named skill each time, its exact timing, and its shortest interval are unknown. For the Codex CLI: unknown. Without a confirmed wake-up that re-enters this skill, the preflight says so and the run is manual or must fit in one turn.
- Background task: a long command returns a session id and is polled with `write_stdin`. Whether completion is announced automatically is not known, so the skill's polling branch applies (poll at about the time awaited).
- User-only skill marker: read `agents/openai.yaml` in the skill's folder, not the `SKILL.md` header (the header key `disable-model-invocation` is Claude Code's). The marker is `agents/openai.yaml` with `policy.allow_implicit_invocation: false` (as stated by a reference file of Codex's own skill-creator, read by the author; both skills in this repo ship that file). Whether it really prevents implicit invocation was not tested.
- Slots: four concurrent slots including the main agent, and sub-agents may open sub-agents (its cited docs: `learn.chatgpt.com/docs/agent-configuration/subagents`). Nested sub-agents, and more than four slots, were not tried.
- Whether `/name` calls a skill: unknown. Whether a Codex worker can call a skill that only a user may start from a model-written prompt: the one trial above found no Skill-call entry in a sub-agent at all, so an explicit call was not verified either way; do not assume the limit that exists in Claude Code.
- Preflight under the default sandbox: since the observed preflight failed closed (above), the temporary worktree, `git fetch`, `git push --dry-run` and the probe worker's file write are very likely refused too (the author's inference for all but the push probe; untested). Expect to need Full Access or an equivalent grant for the orchestrator; for a worker in a bridge-created task see the section on bridge-created and hand-created tasks above. Show the sandbox mode and the approval policy on the preflight screen (`SKILL.md`, preflight step 6). Where to read them (a guess, not verified): the permission prompt at the start of the session, or `sandbox_mode` and `approval_policy` in `~/.codex/config.toml`; failing that, write "unknown". When the approval policy asks a human to approve commands, the run is not unattended; give that as the basis on the first screen line. `workspace-write` probably limits writes to the workspace, which is why the default `worktree-dir` is inside the repository (`reference/adapter-fields.md`) (inference; untested).
- Paths: write the `adapter=` path with forward slashes, relative to the repository root, and give `git show origin/<branch>:<path>` the same form. In Windows PowerShell, write `git show` output to a file and read it as UTF-8 (a guess; not verified).
- Permissions: the Desktop session it described had the network restricted (its mode and review values: toy test above); the CLI modes it listed are `read-only`, `workspace-write`, `danger-full-access`, with the approval policy a separate setting. The observed run needed Full Access to get past the preflight (above); the sandbox of a worker task that a bridge creates is in the section on bridge-created and hand-created tasks. An unattended Codex run would carry the same risk as bypassing permissions elsewhere: no human brake except this skill's hard rules.
- Instruction files: Codex reads `AGENTS.md`, not `CLAUDE.md`. A large `AGENTS.md` can be cut off at a read limit (its stated default is 32 KiB), which could silently drop an inline adapter. That may be why this skill keeps the adapter in its own file with a pointer, and why the setup skill puts the pointer near the top of the file (the author's inference; not tested).
- Keep-alive and restarts: unknown. Context auto-compaction does not guarantee every detail, so run state must live outside the chat (the ledger does). The CLI has `codex resume`. Desktop scheduled tasks need the computer and the app running. Long runs and restart recovery were not tried.
- Its stated obstacles: GitHub network access, credentials, `.git` writes and approvals; scheduled follow-up depends on the app running, and restart recovery is unverified.
