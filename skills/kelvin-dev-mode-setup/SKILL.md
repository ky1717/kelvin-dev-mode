---
name: kelvin-dev-mode-setup
description: Configure this repository for kelvin-dev-mode (the overnight autonomous build run): write the adapter file and the pointers to it. Run once before first use of kelvin-dev-mode.
disable-model-invocation: true
---

# Setup for kelvin-dev-mode

Writes the per-repo configuration that `kelvin-dev-mode` reads: an adapter file, `docs/agents/dev-mode.md` (plus one named file for each further way of reaching workers), and a short pointer in each instruction file (`AGENTS.md`, `CLAUDE.md`) that already exists. This is a prompt-driven skill, not a script: explore, present what you found, ask one question per section, show a draft, then write.

The main skill's folder, `kelvin-dev-mode`, sits next to this one. Its `reference/adapter-fields.md` defines every field, `reference/contracts.md` holds the `list-tickets` example and the skill contracts, `executors/` the executor interfaces, and `harness/` the tool names for your harness. Read them as you need them.

## 1. Explore (read-only)

Look at the repo; do not assume.

- `git remote -v`: github.com? Anything else (GitHub Enterprise, another host, several remotes, none): stop and say "unsupported tracker (GitHub only)".
- `AGENTS.md` and `CLAUDE.md`: which exist (if one is a link to the other, treat them as one file); whether either has a pointer to `docs/agents/dev-mode.md`; whether either has an inline `## Dev mode` section with fields (an older adapter), and what it holds.
- `docs/agents/issue-tracker.md`: if present, it tells you which tracker the project uses (never ask the tracker question again). `docs/agents/triage-labels.md`: if present, read the real name of the "ready for an agent" label from it; otherwise the default is `ready-for-agent`. `docs/agents/dev-mode.md`: if present, you are updating it.
- The default branch (`gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`); whether `gh` is on PATH (if not, ask for its path later), is authenticated, and can read and create issues and labels; whether the labels `dev-mode-ledger` and the ready label exist.
- Which installed skills could fill the three roles `implement`, `review` (two-axis: Standards and Spec) and `pr`. They are optional: a role without a skill works from the contract. Flag any candidate whose name is also a built-in command of this harness (see `harness/`).
- What this harness offers: a wake-up tool that re-enters a skill, a sub-agent tool, the actions in `executors/bridge.md` (five required, *close worker* optional), and whether the worker program that `executors/cli.md` starts is found and runs (the check is named in `executors/cli.md`, Preflight). Write down what you could not confirm.
- CI workflows (candidates for `verify`), deploy workflows and their path filters, and the merge methods the repo allows (`gh repo view --json squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed`).

**Already configured.** If an instruction file has an inline `## Dev mode` section that holds every required field (`executor`, `list-tickets`, `ineligible`, `deploy-check`) with real values, say "this repository is already configured; nothing to do", show what you found, and offer three choices: leave it (then add no pointer to any file and stop), move it (section 4), or add another way of reaching workers (move it first, because the pointer list needs a file for `default`; then section 2 A and the second-adapter path in section 4). Do not run the other sections.

## 2. Present findings and ask

Summarise what is present and missing. Then take the sections in order, **one question per section**, recommended answer first so the user can accept it in a word, with a one-line explainer only where the choice genuinely branches. Skip a section that exploration already settled, for example B when the adapter already has a working `list-tickets`, or D when the repo has no CI at all (then say it stays `pr-only`).

- **A. Executor.** List the ways this machine offers, from exploration, with one line of reason each, and recommend one:
  - `cli`: when `executors/cli.md` covers this orchestrator and its worker program is found and runs. It needs a `pull_request` rule on the default branch; the preflight refuses without one. A worker is not stopped by a permission question at night; it rests on the orchestrator's check and branch protection (`reference/unsandboxed-workers.md`).
  - `bridge`: when all five required actions are present. The same reason applies to a Claude worker.
  - `subagent`: always; it needs no setup.

  When `docs/agents/dev-mode.md` already exists, first ask: change its way, or add another way as a new file. Recommend the first of the three that is present. The user may take several: the most used is `default`, each other way gets its own named file (section 4). Say that a run keeps the way it started with; to use another way, start a new run between two runs with `adapter=<path>`.
- **B. list-tickets.** Which label marks a ticket as ready for an agent (recommend the name you found). Build the GitHub default from `reference/contracts.md` with it, run it once and show the output.
- **C. ineligible.** Recommend the conservative default: tickets labelled as acceptance gates or needing a migration, plus keyword rules. The question: how does this project mark a "gate", and what counts as a "migration"?
- **D. verify.** What machine check must be green before a merge? Offer both forms: CI check names found in exploration, or a command run on a clean checkout of the PR head; or "none". Recommend the CI check names if any were found. With none, the adapter gets no `verify` and no `merge: auto`.
- **E. deploy-check.** Recommend `none` when no deploy workflow was found, else the command that waits for the deploy run; either way say how "no deploy ran" looks (path-filtered workflows). With `none`, say plainly that nothing detects a broken default branch after a merge.
- **F. skills.** Recommend "none yet: each role works from the contract in `reference/contracts.md`". Say that the contract is the baseline, and that Matt Pocock's skills, <https://github.com/mattpocock/skills> (`implement`, a two-axis review, `pr`; `implement` and `tdd` are the stronger part), are an optional enhancement. If the user names skills or you found installed candidates, resolve each to its file and show the path. A colliding name (also a built-in command): say so and recommend a distinct one. A user-only skill: say the role follows the three steps in `reference/contracts.md`. With `executor: bridge` or `cli`, recommend `pr-body: orchestrator` (`reference/adapter-fields.md` gives the reason). Never invent a name. With no names, the adapter gets no `skills` line.
- **G. The rest.** Everything else stays at its default unless the user names a change: `merge: pr-only` (`auto` only if D named a `verify`, and never without), `merge-method`, `forbidden-files` (default, and the rule that a value replaces it: `reference/adapter-fields.md`), `report-language`, `max-active`, `fix-rounds`, `stop-at`, `stuck-after`, `local-checks`, `install`, `check-ticket`, `gh` (a path, if it is not on PATH). If the user insists on `merge: auto` without `verify`, refuse to write it and explain that an automatic merge needs a named machine check. Ask: "keep these as they are, or name what to change?" Then list the labels you would create and the files you would write.

## 3. Confirm and edit

Show the user the full draft of every adapter file and of the pointer block. Let them edit before writing. Before writing, check the draft for placeholders: any value wrapped in parentheses or angle brackets, or a field whose value was never collected. Do not write such a line; ask again, or leave the field out.

## 4. Write

Order: the adapter file first, then the pointers, then the labels.

- **Adapter files.** One file per chosen way. They differ only in the lines that depend on the executor: `executor`, `executor-notes`, `pr-body`, and `max-active` or `worktree-dir` where the way needs another value; every other field has the same value in all of them. Write only the fields whose value was collected; a field you did not collect is left out, and its default applies (use `dev-mode.md` beside this file as the shape, not as text to copy). Create `docs/agents/` if it is missing. If the file exists, change only the lines of fields this skill owns and leave every other line and all surrounding text as the user wrote it.
- **A second adapter.** If the project already has an adapter and the user wants another way of reaching workers (or an adapter for a different orchestrator), write a new named adapter file, `docs/agents/dev-mode-<name>.md`, and never overwrite an existing file. `<name>` is the executor's name (`cli`, `bridge`, `subagent`); when that name is taken, ask the user for another. List every adapter in the pointer section as `name: path` lines (the first one `default`) and tell the user to start a run with another way by `adapter=<path>`:

  ```markdown
  Adapters (`default` is used without `adapter=<path>`):
  - default: `docs/agents/dev-mode.md`
  - cli: `docs/agents/dev-mode-cli.md`
  ```
- **Pointer.** Add this section to **every** instruction file that already exists among `AGENTS.md` and `CLAUDE.md` (different harnesses read different files), near the top of the file, not at the end, because a harness may read only the start of a long file:

  ```markdown
  ## Dev mode

  Adapter: `docs/agents/dev-mode.md`. Read by `kelvin-dev-mode`.
  ```

  With more than one adapter file, the pointer is the list form shown under "A second adapter". A pointer already there stays (a list gains the new line). If neither file exists, ask which to create, or both; do not choose. A file must never hold both a pointer and inline adapter fields.
- **Inline adapter that already exists.** Show it and ask: *move* or *keep*. Move: show the difference first, put the fields into `docs/agents/dev-mode.md`, and replace the inline section in that file with the pointer. Keep: add no pointer to any file and write no adapter file; the main skill reads the inline section as it is. If both files carry different adapter content, say so before writing anything and ask which one is authoritative; then write that one's content and replace the other's with the pointer.
- **Labels.** Check each label exists (`gh label list`) and create only the missing ones the user agreed to (`dev-mode-ledger`, the ready label) with `gh label create`.

## 5. Done

Say what was written, and that the adapter file and the pointers must be committed and pushed to the default branch: the main skill reads them from the default branch on `origin`. Point to the main skill (`kelvin-dev-mode`; how it is called depends on the harness, see `harness/`), and recommend a daytime dry run with `merge: pr-only` first. Running this skill again updates the file in place; it does not start over unless the user asks for that.
