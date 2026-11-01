# Dev mode adapter

The shape of the file the setup skill writes: one line per field, ``- `field`: value``. The values below are only examples of the format. Write the values collected from the user, and leave out any field that was not collected (its default applies). Fields, defaults and meanings: `reference/adapter-fields.md` in the `kelvin-dev-mode` skill folder.

```markdown
# Dev mode adapter

- `executor`: subagent
- `list-tickets`: gh issue list --state open --label ready-for-agent --limit 200 --json number,title,blockedBy --jq '...'
- `ineligible`: a ticket labelled needs-migration or acceptance-gate
- `deploy-check`: none
- `verify`: the CI checks `build` and `test`
- `merge`: pr-only
```

Optional fields that are written only when the user gave a value: `skills` (for example `implement=implement, pr=pr`; without it every role works from the contract), `merge-method`, `local-checks`, `install`, `worktree-dir`, `forbidden-files`, `check-ticket`, `check-ticket-warnings`, `dispatch-doc`, `executor-notes`, `default-branch`, `gh`, `pr-body`, `models`, `report-language`, `max-active`, `fix-rounds`, `stop-at`, `stuck-after`.
