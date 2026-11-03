# kelvin-dev-mode

An overnight build run for coding-agent harnesses (see **Where it has run** below). Before bed you start it; an orchestrator dispatches each eligible ticket to a fresh implementer and a fresh reviewer, opens PRs, optionally merges them, and leaves a morning report.

> **Warning: this runs unattended and may merge code.**
> With `merge: auto` it merges PRs into your default branch while you sleep, and a broken merge can reach your deploy. The default is `merge: pr-only`: it opens and reviews PRs and never merges. Read **Permissions** before you start.
>
> **Status:** this version has not had a complete real overnight run. The one real overnight run so far (one project, a Claude orchestrator with Codex workers through a bridge) used an earlier version. The current version has had static reviews and a few small tests: the `subagent` executor in a private toy repository, Codex as orchestrator in a toy repository, and the bridge's tools for a Codex orchestrator with Claude workers on one machine (details: **Where it has run**). Start with a daytime dry run (below): call the skill, read the preflight screen, and only then answer "go" for two small tickets with the default `merge: pr-only`. Use `merge: auto` only with a `verify` check that must pass on the PR head; without `verify` the skill runs as `pr-only`.

## What it does

1. Reads your project's adapter (a file named by a pointer in `AGENTS.md` or `CLAUDE.md`, or an older inline `## Dev mode` section) from the default branch on `origin` after a `git fetch`; if your local copy differs, the preflight screen says so, and the run uses the `origin` version.
2. Builds a queue of eligible tickets and shows it to you. You answer "go" (in any language) or name a change.
3. Per ticket: implementer, PR, two-axis review (Standards and Spec), up to `fix-rounds` fix rounds, then the merge gate.
4. Keeps a **ledger** (a GitHub issue) so a new session can resume from it alone.
5. Stops at `stop-at`, when no dispatchable ticket remains, when you say stop, after a failed deploy, or when the infrastructure fails (`gh` login gone, the tracker unreachable twice in a row, every worker start failing). Each of the last three writes a `stopped` comment so a restart does not resume merging; the run then posts a morning report and does not retry until morning.

The orchestrator never writes or reviews code.

## Which one do I use?

Matt Pocock's `implement-spec` (in the repository named under **Skills for the three roles are optional** below) implements one spec's tickets on a single integration branch: implementer sub-agents each work in a worktree, a merger sub-agent combines them, and a review closes it (per its own SKILL.md). This skill is different: it works an unattended queue of independent tickets, one PR per ticket, keeps a ledger you can pause and stop, and leaves a morning report. Use `implement-spec` for one spec you drive in one go; use this for a queue that should shrink while you are away.

## Quick start

You need a GitHub repository, `gh` installed and logged in, and a label that marks tickets as ready for an agent (the setup skill asks for its name; the `list-tickets` example uses `ready-for-agent`).

1. Install the two skill folders (below). A session that is already open may not refresh its skills; opening a new session is safer (not verified one by one).
2. Run the setup skill once per repository. It writes the adapter file and the pointers: `/kelvin-dev-mode-setup` in Claude Code, `$kelvin-dev-mode-setup` in Codex (the Codex form is untested). Then commit and push the adapter file and the pointers to the default branch: the run reads them from `origin` (`skills/kelvin-dev-mode/SKILL.md`, Adapter).
3. Do a daytime dry run with `merge: pr-only` (below).
4. Run `kelvin-dev-mode` itself the same way (`/kelvin-dev-mode`, `$kelvin-dev-mode`).

## Requirements

### Where it has run

| Orchestrator | Status |
|---|---|
| Claude Code | One overnight run in one project, with the `bridge` executor (a Claude orchestrator, workers in Codex), under an earlier version of this skill. |
| Claude Code, `subagent` executor | Three small scenarios in one private toy repository, in one continuous test: with `merge: pr-only`; with `merge: auto` and a CI `verify` check (three tickets merged); with no role skills (the contract only). |
| Codex | The orchestrator ran once in a toy repository: three tickets, about ten minutes, interactive, Full Access, no wake-up tested; running unattended is not verified. `skills/kelvin-dev-mode/harness/codex.md` separates what was observed from what is still Codex's own statement. |
| Codex, `bridge` executor with Claude workers | The bridge's tools were tested on one machine; one real preflight in a practice repository (2026-11-03: the probe worker wrote a file, the orchestrator committed it, the session was closed) completed, and it corrected one rule (the session's directory). The whole dev-mode flow has not run. |
| Any other harness | Not run. |

Everything below describes intended behavior until a run confirms it.

### What it assumes

Every assumption the skill makes, and what happens when it does not hold. Each row has exactly one of three outcomes: the preflight **refuses** and names the missing piece; the run goes on **degraded** and tells you; **no effect**. What the preflight cannot prove is listed after the table.

| Assumption | If missing |
|---|---|
| A wake-up tool that re-enters the skill (in Claude Code, `ScheduleWakeup`; for others see `harness/`). The preflight looks for such a tool among the tools it has. | **Degraded**: it says "cannot run unattended" and runs only while you are present, one manual tick at a time, telling you "call me again" after each. It does not pretend to run through the night. |
| A sub-agent facility, for the `subagent` executor | **Refuses**, naming it. |
| Tickets are GitHub Issues, the git remote is on github.com, and `gh` is installed, authenticated, and can read and create issues and labels | **Refuses**: "unsupported tracker (GitHub only)" for another host, or the failed `gh` check by name. Other trackers are not supported in v1. |
| `git` with worktrees (`git worktree list` runs), and the preflight probes (a temporary worktree, a push dry-run, a probe worker that writes a file there, which the orchestrator commits: `skills/kelvin-dev-mode/SKILL.md`, preflight step 2) succeed | **Refuses**, naming the failed probe; fix the cause (for example open the worker application) and say "go" again, or run in manual mode. |
| The default branch can be found (`git ls-remote --symref origin HEAD`, else `gh repo view --json defaultBranchRef`); the adapter is read from it, and a `default-branch` field governs later use only | **Refuses**. |
| A skill for each role (`implement`, `review`, `pr`) is optional. A role named in `skills` must resolve to a readable skill file (the preflight checks the orchestrator's own skills only; what a worker can call is unchecked) | **Refuses**, naming the role, for a named skill that does not resolve. **Degraded**, listed on the preflight screen and in the `run start` comment: a role with no skill works from the contract. |
| For the `bridge` executor: a bridge that provides the five required actions in `executors/bridge.md` (*close worker* is optional) | **Refuses**, naming the missing action. Without *close worker*: **degraded**, and the report lists the threads left unclosed. |
| `list-tickets`, `ineligible`, `deploy-check` in the adapter; `list-tickets` prints a ticket number and `READY` or `BLOCKED` on every line | **Refuses**, naming the field. `deploy-check: none` is valid; with `merge: auto` the preflight warns that nothing detects a broken default branch. |
| With the `subagent` executor, the role skills can be called by a worker: a user-only skill (`skills/kelvin-dev-mode/reference/contracts.md`) may not be callable | **Degraded**: the preflight tells you and records it in the `run start` comment; workers then do that role directly from the contract in `skills/kelvin-dev-mode/reference/contracts.md`. |
| A CI check you name as `verify`, for `merge: auto` | **Degraded**: the run is `pr-only` and says so. |
| The merge method you chose (`merge-method`, default `squash`) is allowed by the repository, for `merge: auto` | **Refuses**, naming `merge-method`. With `pr-only`: no effect. |
| Windows, macOS or Linux paths | **No effect**: the skill uses `git` and `gh` and assumes no path. Paths live in your adapter commands (the examples here are bash). |
| A database with migrations | **No effect** by design: the "migration versions" part of the merge gate has nothing to compare. It has not been run on a project without migrations. |

**What the preflight cannot prove.** A command that asks for approval stalls an unattended run, and nothing detects the stall: the run stays where it is, the ledger shows the last state written, and you find it in the morning. Other limits the preflight cannot see (a harness cap on total concurrency, a sandbox that blocks a command only later) show up mid-run as failures; the infrastructure rule in the skill stops the run when every worker start fails. See **Permissions**.

### Skills for the three roles are optional

The run needs three roles done: `implement`, a two-axis `review` (Standards and Spec) and writing the PR body (`pr`). **The contract in `skills/kelvin-dev-mode/reference/contracts.md` is the baseline**, so with no `skills` line the workers do each role directly from it and you install nothing from anyone else. Only one of these paths has been tried for real: in one sandbox run the `implement` role worked from the contract and the result was correct. The `review` and `pr` roles from the contract, and a run with all three, are untested. The contract is a baseline and is thinner than Matt's `implement` plus `tdd` (for example, `tdd` is stricter about slices and about tests that merely restate the code). On the contract path, quality rests on the independent review and on `verify`, and the evidence so far is a few toy tickets in a sandbox.

You can name skills instead. Matt Pocock's public skills, <https://github.com/mattpocock/skills> (MIT), have all three: `implement`, `code-review` (two-axis review) and `pr`. Install them by copying their folders into one of the skills folders above.

- **A named skill must resolve to a readable skill file**, or the preflight refuses and names the role. That also catches a misspelt name.
- **Do not use a name that is also a built-in command of your harness.** Claude Code ships its own `/code-review` (it looks for defects in the current diff, which is different). A built-in with the same name as your skill could make the preflight pass without being the skill you mean. Choose a distinct name (for example install Matt's under another name) and confirm yourself that the review skill is the two-axis one, for example by checking that it splits Standards and Spec into two sub-agents. Which one wins when a built-in and an installed skill share a name is not verified.
- **User-only skills.** A skill that only a user can start (Claude Code: `disable-model-invocation: true`, as Matt's `implement` has; Codex: `agents/openai.yaml`) may not be callable by a worker. The preflight tells you, and the role follows the three steps in `skills/kelvin-dev-mode/reference/contracts.md`. The middle step has been seen only in one manual trial in Codex Desktop, not in a dev-mode run, and not at all in Claude Code.

Any bridge that provides the actions in `skills/kelvin-dev-mode/executors/bridge.md` can back `executor: bridge`, in either direction (Claude or Codex as orchestrator). Bridges depend on the worker application's own interfaces, which may have no public contract and can change.

## Install

Get the files with `git clone https://github.com/ky1717/kelvin-dev-mode`, then copy both folders, `skills/kelvin-dev-mode/` and `skills/kelvin-dev-mode-setup/` (with everything inside them), into a skills folder of your harness. Claude Code: `~/.claude/skills/` (user level) or `<project>/.claude/skills/` (project level). Codex: `~/.codex/skills/` (untested here; see `skills/kelvin-dev-mode/harness/codex.md`). On Windows, `~` is `%USERPROFILE%`, with the same folder names below it. No minimum version of Claude Code or Codex was checked. The setup skill creates the labels `dev-mode-ledger` and the ready label if they are missing, after you agree. Both skills are user-invoked only: Claude Code honours the frontmatter key `disable-model-invocation`, and each folder also ships `agents/openai.yaml` with `allow_implicit_invocation: false` for Codex (as stated by Codex, not tested here).

## Start a run

Open an ordinary interactive session of your harness in the project and call `kelvin-dev-mode` (`/kelvin-dev-mode` in Claude Code, `$kelvin-dev-mode` in Codex). This session is the orchestrator. It must stay open until the report appears, so keep the machine awake. The run needs a harness that can schedule a wake-up and re-enter a skill when it fires; the orchestrator uses it to start its next tick. Without one, the preflight says "cannot run unattended" and you drive the run by hand, calling the skill again after each tick. If the harness announces when a background task finishes, the wake-up is only a long fallback (about 1200 s); if it does not, the orchestrator polls at about the time awaited (about 480 s for CI); while a worker reply is awaited it uses about 600 s. All three numbers are starting guesses. A `stopped` ledger comment, or a posted morning report, ends the wake-ups; `paused` limits what a tick does until `resumed`. A restarted session resumes from the ledger: call the skill again and it skips preflight if a run is open (unless `stop-at` has passed since the run began, in which case it posts the report instead of resuming). The skill does preflight, shows the queue and a one-screen summary, and waits for your "go". After that it does not ask you again.

## Adapter

The setup skill writes `docs/agents/dev-mode.md` and adds a short pointer ("Adapter: `docs/agents/dev-mode.md`. Read by `kelvin-dev-mode`.") to every instruction file that already exists, `AGENTS.md` and `CLAUDE.md`, near the top of the file because a harness may read only the start of a long one, and because different harnesses read different files. The main skill finds the adapter in this order: a pointer to an adapter file; else an inline `## Dev mode` section with fields (older projects); if both files carry adapter content that differs, it refuses and names both; if there is none, it tells you to run the setup skill.

Required fields: `executor`, `list-tickets`, `ineligible`, `deploy-check`. Everything else has a default; the full list with defaults and meanings is `skills/kelvin-dev-mode/reference/adapter-fields.md`, the single source, so this README does not repeat it. A line whose whole value is wrapped in parentheses or angle brackets is a placeholder that the preflight refuses; put your own value there before a run. Example file; the fields from `skills` on are optional. In the example, `skills` names Matt's skills for two roles (leave the line out to work from the contract; see the warnings below). `verify` names CI checks, which have no default (without `verify` the run is `pr-only`). `models` is an example for Claude Code and the `subagent` executor only. With the `bridge` executor `pr-body: orchestrator` is recommended, because the preflight cannot check which skills a worker can call. `max-active` on Codex: a suggested 1 (inferred from its stated four slots, not measured). What each field means: the file named above.

```markdown
# Dev mode adapter

- `executor`: subagent
- `list-tickets`: (a command; see `skills/kelvin-dev-mode/reference/contracts.md` for the format and a runnable example)
- `ineligible`: a ticket labelled `needs-migration` or `acceptance-gate`
- `deploy-check`: none
- `skills`: implement=implement, pr=pr
- `merge`: pr-only
- `merge-method`: squash
- `verify`: build, test
- `local-checks`: your lint, type-check and unit-test commands
- `install`: your dependency install command, run in each fresh worktree
- `models`: implementer=sonnet, reviewer=sonnet
- `pr-body`: worker
- `stuck-after`: 45 min
- `max-active`: 3
- `fix-rounds`: 2
- `stop-at`: 07:00
- `report-language`: English
```

Optional fields not shown here (`worktree-dir`, `forbidden-files`, `check-ticket`, `check-ticket-warnings`, `dispatch-doc`, `executor-notes`, `gh`, `default-branch`) are in `skills/kelvin-dev-mode/reference/adapter-fields.md`. `merge: auto` needs `verify`; without it the run falls back to `pr-only`. `deploy-check: none` with `merge: auto` means nothing detects a broken default branch after a merge, and the preflight warns about it. `models` is for the `subagent` executor only; unset, both workers use the orchestrator's model. `stuck-after` is a starting guess: a worker with no new push, PR comment or review event, or output for that long is replaced by a fresh one.

### One project, several orchestrators

The same project can be run by different orchestrators (for example Claude orchestrating Codex through a bridge, Codex orchestrating Claude workers through a bridge, or Codex orchestrating sub-agents), and their adapters differ in `executor`, the `gh` path or the shell syntax of `list-tickets`. Keep one adapter file per orchestrator. The pointer section in `AGENTS.md` / `CLAUDE.md` can list several named adapters (one called `default`), and a call can name the file to use: `adapter=<path>` after the skill call, for example `/kelvin-dev-mode adapter=docs/agents/dev-mode-codex.md` or `$kelvin-dev-mode adapter=docs/agents/dev-mode-codex.md`. Without it, the `default` adapter is used. The setup skill can add a second adapter without touching the first. Nothing inside an adapter file changes.

### list-tickets and the skill contracts

The `list-tickets` format (one line per ticket: number, `READY` or `BLOCKED`, title, tab-separated), a runnable example, and what the three roles must take, do and return, are in `skills/kelvin-dev-mode/reference/contracts.md`. It is part of the installed skill, so it travels with it. The example uses GitHub's issue dependencies (`blockedBy`); it was run with `gh` 2.103.2 and other versions are untested.

## Terms

- **Ledger**: one GitHub issue, found by the label `dev-mode-ledger` (create the label first if your repo lacks it). Every state change is a comment. The states, the run-level comments, the `ready to merge` proof values and the rule for a restart after a morning report: `skills/kelvin-dev-mode/reference/ledger.md`. Any script that parses the ledger has to recognise all of them.
- **Tick**: one pass of the run: read the ledger, read unread results, check each worker for progress, act, then schedule the next wake-up. Delays: about 1200 s as a long fallback while background tasks run, about 600 s while a worker reply is awaited; starting guesses, not measured values.
- **Evidence only CI can give**: `skills/kelvin-dev-mode/reference/contracts.md`.
- **Spec**: the tickets plus whatever specification your project keeps (docs, ADRs, a glossary). The run never changes it to get out of a problem.
- **Seams**: the public interfaces where tests should attach, named by the orchestrator in the implementer's task.
- **Merge gate**: defined in `skills/kelvin-dev-mode/SKILL.md` (Loop step 6).
- **Executors**: `subagent` (same-vendor sub-agents: the orchestrator opens an implementer and a reviewer per ticket) and `bridge` (workers in another application implement and review through a bridge with five required actions and an optional *close worker*).

## Permissions

An unattended run cannot answer permission prompts, so a prompt it hits stalls the run.

1. In Claude Code, sub-agents inherit the permission mode of the orchestrator session (other harnesses: not verified).
2. Recommended: give the orchestrator session an allowlist of the commands the run needs (in Claude Code, in your project's `.claude/settings.json`, syntax per your version; for Codex see `harness/codex.md`), plus your project's own test, install, `verify` and `deploy-check` commands, and allow nothing else. This list is the only copy of the skill's commands; it was compiled on 2026-11-01 from the commands the skill names and the commands its actions need (the skill text says, for example, "open the PR" without naming `gh pr create`), so update it when an action is added there. `gh`: `auth status`, `api` (only to read `repos/<owner>/<repo>/rules/branches/<default>`), `repo view`, `run view`, `issue list`, `issue view`, `issue comment`, `issue create`, `issue pin`, `label list`, `label create`, `pr create` (with `--draft` for evidence PRs), `pr view`, `pr comment`, `pr diff`, `pr close` (evidence PRs only), `pr merge`. `git`: `fetch`, `ls-remote`, `for-each-ref`, `show`, `ls-files`, `worktree list`, `worktree add`, `worktree remove`, `push`, `add`, `commit`, `reset`, `restore`, `clean`, `rev-parse`, `merge`, `status`, `diff`, `patch-id`. The setup skill also runs `git remote -v`. Narrow three of them: leave `gh pr merge` out under the default `merge: pr-only`; allow `git merge` (it merges the default branch into a PR branch during merge preparation) only with `merge: auto`; allow `git push` only in the forms the skill uses, `git push -u origin <branch>`, `git push origin --delete <branch>` and `git push --dry-run origin HEAD:refs/heads/<name>`. The skill writes `git -C <path> push ...`; whether that form matches a prefix rule depends on your permission system, so try it (not verified). The orchestrator stages and commits under both executors (`skills/kelvin-dev-mode/reference/contracts.md`, The orchestrator's commit), so allow `git add`, `git commit` and `git status` in its session; workers do not commit. `git branch`, `gh pr checks` and `gh run` do not appear in the skill text; allow them if your `verify` or `deploy-check` command uses them.
3. Waiting for CI and deploys runs as background tasks, and the next tick comes from the wake-up tool. Whether your setup asks permission for either is not checked here: try it in the dry run.
4. The authority is the orchestrator's check of every worker turn (`skills/kelvin-dev-mode/executors/bridge.md`, Workers without a sandbox); a deny list, the credential mode `none` and branch protection rules only guard against accidents. A worker that runs with bypass permissions (for example a Claude worker started through a bridge) is a risk the operator accepts. Bypassing permissions entirely makes the run work, but unattended it means no human brake: the only limits are the skill's hard rules, and you take that risk. This README does not recommend it.
5. A Codex orchestrator running unattended will most likely need full access, because `gh`, `git push` and `git worktree add` (which writes inside `.git`) cannot be assumed to work under a restricted sandbox without elevated permission (Codex's own statement, untested here). That carries the same risk as bypassing permissions in Claude Code: no human brake except this skill's hard rules. Workers in a Codex task that a bridge creates leave their changes uncommitted, for the sandbox reason in `skills/kelvin-dev-mode/harness/codex.md`.

## Daytime dry run

1. Label two small, low-risk tickets.
2. Set `merge: pr-only` and `stop-at` to about an hour from now.
3. Run `/kelvin-dev-mode` (`$kelvin-dev-mode` in Codex); check the queue and the one-screen summary; answer "go".
4. Read the ledger, the PRs and the review comments. Confirm that nothing was merged and that the report lists the PRs waiting for you.
5. Only then consider `merge: auto`, with `verify` set.

## Hard rules

- `max-active` bounds the workers that count at once; what counts: `skills/kelvin-dev-mode/reference/adapter-fields.md`.
- At most `fix-rounds` fix rounds per ticket, then the ticket is blocked.
- A failed deploy writes a `stopped` comment: nothing new is dispatched or merged; in-flight work may finish into PRs, unmerged.
- You can pause or stop the run in any wording or language. The first action is a `paused` or `stopped` comment in the ledger. Pause: nothing new is dispatched (fix rounds, new reviews and replacements included); results are still read; under `merge: auto` a PR already in the `ready to merge` state may still merge, and PRs that need a re-review or a fix wait. Stop: nothing is dispatched or merged and no wake-up is scheduled; running workers are closed, pushed branches are kept, no worktree is deleted (the report lists them for you to decide), and one morning report closes the run. If it is unclear which, the run treats it as stop and asks once.
- Nothing is dispatched or merged after `stop-at`.
- No merge without the full merge gate.
- No executor edits a path in `forbidden-files`, and the run does not edit the executors' skills or permissions. The one exception is the evidence Draft PR, which never merges.

## Acknowledgements

Ideas in this skill (two-axis review, seams, vertical slices, a `tdd` baseline) come from Matt Pocock's skills (MIT), <https://github.com/mattpocock/skills>. No endorsement by their author is implied.

## License

MIT. See `LICENSE`.
