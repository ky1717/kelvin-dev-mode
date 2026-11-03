# Harness notes: Claude Code

Read when the orchestrator is a Claude Code session (or a Claude Code worker is used). Tool names change between releases; check them against your session's tool list.

| Verb in the skill | In Claude Code |
|---|---|
| Call a skill | `/name`, for example `/implement #12`. |
| User-only skill marker | The frontmatter key `disable-model-invocation: true` (the main and the setup skill both set it). |
| Wake-up tool | `ScheduleWakeup`. Its prompt is run again when it fires; for this skill the prompt re-enters `/kelvin-dev-mode`. The tool's own description clamps the delay to a range of 60 to 3600 seconds and advises a long fallback, not short polling, for work the harness tracks. |
| Background task | The shell tool's background mode. Its description says you are notified when the command exits, so a CI or deploy wait can end the turn and be woken by the notice. |
| Spawn worker / sub-agent | The `Agent` tool (parameters include a name and a model). A general-purpose sub-agent has the `Agent` tool itself, so it can spawn sub-agents; a second-level agent could not take a name (observed in an earlier session, not rechecked). |
| Close worker | `TaskStop`. |
| Background task stopped by the system | A task that the system stops for lack of memory comes with a notification that says not to restart it. Leave it stopped: end the turn, and at the next wake-up confirm the awaited condition with a one-off query (for example `gh run view`). Memory is tight in practice: on the machine that ran the overnight run of 2026-11-01, Codex Desktop used about 5.6 GB and the system reclaimed one background task. |
| Follow-up message to a worker | `SendMessage`. Stop a worker first when it is mid-task. |
| Permissions | A command that asks for approval stalls an unattended run. Put the commands the run needs on an allowlist in your project's `.claude/settings.json` (syntax per your Claude Code version). Bypassing permissions makes the run work but removes the human brake. |

A skill that can only be started by a user (`disable-model-invocation: true`, as Matt's `implement` has) is a user-only skill. A sandbox run of the `subagent` executor found the implementer, whose prompt the orchestrator wrote, unable to start it. Such a role follows the three steps in `reference/contracts.md`. Whether a Claude sub-agent can read a user-only skill's file and follow it was not tested (it waits for a sandbox run); the sandbox run that found the limit used the contract only. The `bridge` executor, whose workers receive the call as a user message, did not hit this in the one overnight run.

## Bridge tools for Codex workers

Source: the tool descriptions of the bridge used in testing, read on 2026-11-01. These are interface facts, not measurements; the earlier overnight run used this direction (a Claude orchestrator with Codex workers) under an earlier version of this skill, and the whole flow of the current version has not run overnight.

A Claude orchestrator reaches Codex workers through the bridge used in testing (`executors/bridge.md` has the protocol). Per action: start worker, `start_codex_thread`; send task, `send_to_codex_thread`; read result, `read_bridge_message`, `list_unread_results` and `acknowledge_bridge_message` (`read_codex_delivery` reads a receipt); query status, `list_codex_threads` and `read_codex_thread`. There is no close action: `interrupt_codex_turn` ends one turn, and `stop_codex_app_server` ends every worker, so it is not one worker's close.

Command-line workers on Windows (tested on 2026-11-01 with Claude Code 2.1.316; one test). A worker can commit by itself only when its allowlist names both tool families: with an ordinary profile and a git allowlist, write `Bash(git add:*)` and `PowerShell(git add:*)` (the same for each git command); writing one family is refused. A launcher preset that starts with `--restricted` disables every shell, so its worker cannot run git. With `-p --permission-mode dontAsk`, `Write` must be in the allowlist. This skill still has the orchestrator commit (`reference/contracts.md`, The orchestrator's commit); these lines are facts, not a rule.

Without a sandbox, a deny list such as `--disallowedTools "Bash(git push:*)"` blocks only the literal command: `git -C . push` gets around it (tested in a toy repository against a local bare remote, where the remote's refs showed the push). The orchestrator's check is therefore the authority.

A sub-agent cannot hand its report back once the agent that spawned it has finished: in a sandbox run of the `subagent` executor, the review sub-agents' hand-back failed with "spawner no longer running" four times (the mechanism name, `SubagentHandback`, is what the orchestrator's transcript showed; the cause was not examined further), because the reviewer finished first. The results arrived only because the sub-agents improvised a message to the orchestrator. So the `subagent` executor spawns every review agent from the orchestrator, at the first level.

Names that collide: Claude Code ships its own `/code-review`, which finds defects in the current diff and is not a two-axis review. Do not use that name for the `review` role unless you have confirmed which one runs. Which wins when a built-in and an installed skill share a name is not verified here.

Untested option: `claude -p` in a shell opens a separate Claude worker per ticket, and a budget flag can cap spend. Unattended permission prompts and model selection with it are unverified.
