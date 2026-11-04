# Workers without a sandbox

The rules for any worker that runs without a sandbox; every executor file that can start such a worker (`executors/bridge.md`, `executors/cli.md`) points here. Under `cli` only the implementer is such a worker; its review processes run read-only (`executors/cli.md`).

## The check

A worker without a sandbox (for example a Claude worker with bypass permissions, or a worker started with full access) is not held back by a sandbox, so the part of "workers hold no push or merge rights" that a sandbox enforces does not hold on the worker's side. The orchestrator's check is the authority. Before and after every worker turn it compares the remote refs (`git ls-remote`), the local refs outside the worktree (`git for-each-ref`), and the main checkout's `HEAD` and status, and it confirms that nothing was pushed. The expected changes are the orchestrator's own commits and pushes, which it records in the ledger: the baseline of a check is what the orchestrator itself recorded since the previous check, and every other change is unexpected (with `max-active` above 1 the branches of other tickets also move, and they are expected only when the orchestrator moved them). A review worker without a sandbox can also change or commit files in its checkout, so for a review worktree (a detached checkout) the check compares `HEAD` and status before and after every turn as well. Any unexpected change stops that ticket (`blocked`, with the reason written) and is reported to the operator.

Before it closes or replaces a worker without a sandbox, the orchestrator makes the full check first, because a stuck worker's turn never ends and the check after the turn never runs. An unexpected change means no replacement: the ticket stops and the operator is told.

## The guards

Three other layers only guard against accidents. Before using a worker without a sandbox in a repository, the preflight checks that its default branch has branch protection (Preflight).

- A deny list: it blocks the literal command and nothing else; `git -C . push` gets around it (`harness/claude-code.md`).
- The credential placeholders. Under a bridge, the credential mode `none` closes the git credential helper, the `gh` configuration and `GH_TOKEN`. Under `cli`, the orchestrator sets a placeholder token and an empty `gh` configuration folder; on the machine tested that made a `git ls-remote` of a private repository fail, because the machine's credential helper reads the `gh` token. Another helper was not tested. They are not a sandbox.
- Server-side branch protection: an administrator can disable it.

## Preflight

`gh api repos/<owner>/<repo>/rules/branches/<default>` lists a `pull_request` rule for the default branch; an empty list, a 404, a 403 or a failing command all count as no such rule, and without one the preflight refuses a worker without a sandbox and says why (the run can use another kind of worker, or the operator sets up the protection and says "go" again).

A private repository on a free personal account answers 403 with a hint to upgrade or make it public. The protection cannot be confirmed, so the preflight refuses; the way out is to upgrade, make the repository public, or use the `subagent` executor.

The interface returns the rules of the rulesets in force, so a repository with classic branch protection only may not be listed and is then refused; the operator can switch to a ruleset (a guess, not verified).

A repository without a GitHub remote is refused earlier by the general preflight (`SKILL.md`, preflight step 2).

The rule's presence is a guard only: an administrator can disable it, so the per-turn check in Workers without a sandbox stays the authority.
