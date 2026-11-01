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
| Follow-up message to a worker | `SendMessage`. Stop a worker first when it is mid-task. |
| Permissions | A command that asks for approval stalls an unattended run. Put the commands the run needs on an allowlist in your project's `.claude/settings.json` (syntax per your Claude Code version). Bypassing permissions makes the run work but removes the human brake. |

A skill that can only be started by a user (`disable-model-invocation: true`, as Matt's `implement` has) is a user-only skill. A sandbox run of the `subagent` executor found the implementer, whose prompt the orchestrator wrote, unable to start it. Such a role follows the three steps in `reference/contracts.md`. Whether a Claude sub-agent can read a user-only skill's file and follow it was not tested (it waits for a sandbox run); the sandbox run that found the limit used the contract only. The `bridge` executor, whose workers receive the call as a user message, did not hit this in the one overnight run.

A sub-agent cannot hand its report back once the agent that spawned it has finished: in a sandbox run of the `subagent` executor, the review sub-agents' hand-back failed with "spawner no longer running" four times (the mechanism name, `SubagentHandback`, is what the orchestrator's transcript showed; the cause was not examined further), because the reviewer finished first. The results arrived only because the sub-agents improvised a message to the orchestrator. So the `subagent` executor spawns every review agent from the orchestrator, at the first level.

Names that collide: Claude Code ships its own `/code-review`, which finds defects in the current diff and is not a two-axis review. Do not use that name for the `review` role unless you have confirmed which one runs. Which wins when a built-in and an installed skill share a name is not verified here.

Untested option: `claude -p` in a shell opens a separate Claude worker per ticket, and a budget flag can cap spend. Unattended permission prompts and model selection with it are unverified.
