# Executor: subagent (orchestrator and workers from the same vendor)

No bridge. You spawn workers with your harness's sub-agent facility (tool names: `harness/`). Spawn every agent yourself, at the first level: do not have a worker spawn sub-agents that must hand a result back, because a sub-agent whose spawner has already finished cannot return its report (seen in a sandbox run; see `harness/claude-code.md`).

## Workers

Each ticket gets an **implementer** (only implements) and, for review, **two independent read-only review agents**, one per axis: **Standards** and **Spec**. All are fresh. No worker serves a second ticket or another role.

- Name them `impl-<ticket>`, `review-<ticket>-standards` and `review-<ticket>-spec` where the harness takes names.
- The model comes from the adapter's `models`. The review agents' tier is at least the implementer's. Pass it explicitly on every spawn.
- Worktrees are created by you as in `SKILL.md` Loop step 1 (baseline `origin/<default>`, a detached checkout of the PR head for the review agents, then the adapter's `install`), and the path goes in the prompt. The implementer works there and does not use the harness's own isolation feature, so the baseline never depends on how the harness chooses one.
- The implementer's prompt line 1 is the `implement` role's line with the ticket (`reference/contracts.md`, Prompt line). A review agent is not asked to call the `review` skill, which would spawn its own two sub-agents and bring back the nesting; its prompt says to review one axis as `reference/contracts.md` describes. `<base>` is the merge-base of the PR branch and the default branch.

## Reviewer independence

Each review agent starts with a clean context. Its prompt carries the diff base, the repo and ticket number, its one axis, and nothing of the implementer's reasoning or Decisions. It is read-only and returns its report to you as its reply. You post the two reports under the PR as two comments headed `Standards` and `Spec`.

## Messages

- Implementer prompt: the worktree path and branch, the ticket number (read it with `gh issue view`), seams (the public interfaces to test), risks, the `forbidden-files` list, "commit in your worktree; do not push and do not open the PR; post your Decisions, the PR body (written with the `pr` skill; with `pr-body: orchestrator`, leave the PR body out) and what you could not verify as a comment on the ticket; do not run commands that print environment variables, and if any output contains a secret, name it in the report and never paste its value; on an ambiguity with no Spec basis your final reply is `QUESTION: <one question>` (`reference/contracts.md`); otherwise return only done, failed, or the comment link".
- Review agent prompt: the detached worktree path, the repo, the PR and ticket numbers, the base, its axis (Standards or Spec), "read-only; do not post anything; spawn no sub-agent; return your report as green or a findings list".
- You push the implementer's branch from its worktree (`SKILL.md` Loop step 3); the implementer never pushes.
- Fix rounds go to the same implementer through the harness's follow-up message facility. When it is mid-task, interrupt it first if the harness has an interrupt or stop action; otherwise wait for its current turn to end, then send the follow-up. When the follow-up fails (the worker is not found, or does not receive it), treat it as a replacement under Tick step 3: the new worker carries the original brief, all later directives, the previous report and the branch, and the replacement counts toward the ticket's limit.

## Limits

- At most `max-active` workers count at once (what counts: `reference/adapter-fields.md`); a ticket's two review agents count as two.
- Close each worker as `SKILL.md` Loop steps 6 and 8 say, not when its first report is in: fix rounds reuse the same implementer. Idle workers hold memory and may be reclaimed.
- Closing a worker is the harness's stop action. A harness without one (Codex has none: `harness/codex.md`): stop sending the worker messages, interrupting a turn in progress where the harness has an interrupt, otherwise waiting for it to end (a degraded procedure, untested).
- A stuck worker (Tick step 3) is closed and replaced by a new worker whose prompt carries the original brief, later directives, and the previous worker's report and branch.
