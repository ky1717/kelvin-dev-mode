# Morning report

Read when the run ends (no dispatchable ticket remains, `stop-at`, or `stopped`). Post a new comment on the ledger issue, in `report-language`. Sections, in order:

1. Line 1: merged N, blocked M, waiting on the operator for K decisions.
2. **Ready to merge** (`pr-only` runs, and `auto` PRs the gate held): per PR, number, ticket, review verdict, `verify` state. This is what the operator merges first.
3. **Waiting on an unmerged PR**: tickets not dispatched because they depend on, or share a file with, a PR that is not merged (`pr-only`), tickets kept out of the queue by an unwaived warning category, and tickets a new run left out because their PR is still open, each with the reason.
4. Per ticket waiting on the operator (blocked, held at gate, or a skipped forbidden-file item on a `Refs #N` PR): one sentence for the problem, one for what the operator must choose, then the options and a recommendation, in plain words without internal jargon. When there is nothing to choose (a `pr-only` PR with no problem), one sentence is enough: merge.
5. Non-blocking findings.
6. Merged PRs, and any merged PR whose `ready to merge` comment lacked the `patch-id` result or the `origin/<default>` hash.
7. Anything the run could not verify. Say so when `verify` stood in for empty `local-checks`, and when both were empty. List the worktrees left on disk (after `stopped`, and those of blocked tickets). List each worker left unclosed, and each terminal tab of a Claude worker that the close reply did not report as asked to close, or whose session crashed (the operator closes such a tab by hand; tool names: `harness/`).
