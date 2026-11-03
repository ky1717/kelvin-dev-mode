# Adapter fields

Read at preflight, from `origin/<remote-default>` after `git fetch` (`SKILL.md`, Adapter), not from the working tree. The adapter is a list of ``- `field`: value`` lines, in `docs/agents/dev-mode.md` (written by the setup skill) or, for older projects, in an inline `## Dev mode` section of the project's instruction file.

Required:

| Field | Meaning |
|---|---|
| `executor` | `bridge` or `subagent`. Read only the matching file in `executors/`. The older names `codex-bridge` and `claude-subagent` are accepted as aliases (deprecated). |
| `list-tickets` | Command that prints one line per open ticket: number, `READY` or `BLOCKED`, title. Format and a runnable example: `contracts.md`. |
| `ineligible` | How to recognise tickets dev mode never touches (acceptance gates, database migrations). |
| `deploy-check` | Command that waits for the post-merge deploy and reports success or failure, and says how to recognise "no deploy ran" (path-filtered). `none` is a valid explicit value: nothing is awaited. |

Optional, with defaults:

| Field | Default | Meaning |
|---|---|---|
| `skills` | none: each role works from the contract | Optional skills for the roles, as `role=name` pairs: `implement=…`, `review=…`, `pr=…`. A role without a name works directly from `contracts.md`, so the run needs no one else's skill. A named skill must resolve to a readable skill file, or the preflight refuses and names the role (this also catches a misspelt name and a name that is a built-in command of your harness, which would pass without being the skill you mean). Preflight checks the orchestrator's own skills only; what a worker can call is unchecked. A user-only skill follows the three steps in `contracts.md`. How a skill is called (`/name`, `$name`) depends on the harness: see `harness/`. |
| `default-branch` | `<remote-default>` (`SKILL.md`, Adapter) | The branch PRs merge into. The field wins when set. In the skill, `<default>` means this branch. The adapter is read before this field is known, so the field governs later use only. |
| `install` | none | Command that prepares a fresh worktree (dependencies, build) before a worker starts. |
| `merge` | `pr-only` | `pr-only`: open and review PRs, the operator merges. `auto`: merge when the gate passes. `auto` needs `verify`. |
| `merge-method` | `squash` | `squash`, `merge` or `rebase`: the flag `gh pr merge` gets. Pick one your repository allows. |
| `verify` | none | The machine check that must be green on the PR head before an automatic merge: CI check names, or a command you run on a clean checkout of the PR head. Every named check must exist on the PR head and be `success`; a missing, skipped, neutral or pending check is not green. Absent: `merge` is forced to `pr-only`. |
| `local-checks` | none | Commands you may run yourself to re-check an implementer's work. Empty: `verify` stands in. Both empty: nothing re-checks the implementer's results, and the morning report says so. |
| `max-active` | `3` | Upper bound on the workers that count at once. A worker counts while it has a turn in progress or a reply you have not read; one that has delivered its result with nothing pending does not, and the preflight probe worker is complete once it has replied. Where the environment limits total concurrency, nested sub-agents count too (under `bridge` each reviewer opens two; under `subagent` the review agents are first-level and already count as two). This counting rule comes from the last overnight run (the orchestrator counted only threads with a turn in progress, and 8 tickets ran without reaching a limit), and one Codex toy test supports it (2026-11-01: nine sub-agents started with completed ones left open; `harness/codex.md`), which makes it no general finding; the first run on this version will show whether the application refuses a new thread because of idle ones. The default 3 assumes no total limit, and none is known for Claude Code (not checked); for Codex see `harness/codex.md`. |
| `fix-rounds` | `2` | Fix rounds allowed per ticket (a fix after a blocking finding, a failed `verify`, a failed `pre-commit` hook or a merge conflict in Prepare is one round). Answering a worker's question is not a fix round. |
| `worktree-dir` | `<repo>/.dev-mode-worktrees` | The folder that holds every worktree the run creates, as `<worktree-dir>/<name>`. `<name>` is the ticket's branch for an implementer, `review-<ticket>` for a reviewer, `probe-<short random string>` for a probe. The default is inside the repository so that a worker with a restricted sandbox can write there (a guess about the sandbox). An absolute path outside the repository is allowed (for example `D:/worktrees`): the operator then makes sure workers can write there, and the preflight probes (a temporary worktree, the probe worker's file write) refuse the run when they cannot. With a Claude worker started through a bridge, the allowed-root rule applies to the session's directory (the repository root), and the message names the worktree's path (`executors/bridge.md`). A reminder, not tested: on Windows a path inside the repository is longer, and npm-style installs can reach the 260-character path limit; a short path outside the repository is safer. |
| `stop-at` | `07:00` | Local clock time after which nothing is dispatched or merged. |
| `stuck-after` | `45 min` | A worker with no new side effect (a push, a PR comment or review event, new output) for this long is replaced (Tick). A starting guess; tune it. |
| `report-language` | `English` | Language of the morning report. |
| `check-ticket` | none | Command run on each ticket before dispatch. It prints `OK`, warnings or errors. The output text decides, not the exit code: kinds named in `check-ticket-warnings` are warnings; any other text that is not `OK` is an error and skips the ticket. |
| `check-ticket-warnings` | none | The kinds of `check-ticket` output that count as warnings. |
| `forbidden-files` | `CLAUDE.md, AGENTS.md`, the adapter file(s) in use, `.github/workflows/**`, `.claude/**` | Paths or globs that no executor may edit during a run (list the Spec files here if they must stay frozen). The default keeps a merged PR from changing the merge gate. A value you give replaces the default, so list every path to freeze, the default paths included. A PR whose diff matches a listed path fails the merge gate. |
| `pr-body` | `worker` | `worker`: the implementer writes the PR body with the PR skill. `orchestrator`: you write it with the PR skill from the implementer's report and the ticket's acceptance items, never from the diff. With `executor: bridge`, recommended: `orchestrator`, because the preflight cannot check which skills a worker can call. |
| `dispatch-doc` | none | Project doc on writing messages, review rounds and merging. |
| `executor-notes` | none | Pointer to the executor's own usage rules (for a bridge: its protocol doc). |
| `models` | orchestrator's own model | `subagent` only: implementer and reviewer models. Unset: both use the orchestrator's current model, so the reviewer is never a lower tier than the implementer. |
| `gh` | `gh` on PATH | Path to the GitHub CLI when it is not on PATH. |
