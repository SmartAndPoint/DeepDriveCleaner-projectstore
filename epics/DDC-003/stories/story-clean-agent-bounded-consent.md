---
type: story
id: "story-clean-agent-bounded-consent"
epic: "DDC-003"
title: "Clean: --agent bounded consent"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-02
external_refs: {}
tags: [cli, safety, cleanup, agents, contract]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Clean: --agent bounded consent

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

*Amended 2026-11-02 (ADR "Agent footprint", §7): the command is `ddc agent clean [ROOT] [--items <file|->] [--json]`, not a `--agent` flag on `ddc clean`. Without a person it removes only `safe` items, and only with the Settings opt-in `agents_clean_safe` on; `rebuildable` items (a harness worktree included) are listed with the `ddc clean --items` command for the person. Criteria below that say `--agent` removes `rebuildable` items are superseded by §7 and get rewritten at this story's plan gate.*

`ddc clean --agent --project ROOT [--items <file|->] [--json]` is the one cleanup entry point for a caller that is not a person: §5 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents". It behaves like `--yes` with a tree: `safe` items, plus `rebuildable` items selected by default that lie inside `ROOT`'s main checkout or its worktrees (the dead worktrees of "Worktree findings and provenance" included). It never removes `review`, never runs `native-only`, never prompts, and journals every action with `consent: agent`. With `--items`, the list filters; it never widens.

It builds on the `--items`, `--json` and exit-code contracts of the DDC-002 story "CLI: clean an explicit item list" and is blocked until that story closes.

## Decomposition

Rough until the plan gate.

- [ ] Flags: `--agent` requires `--project ROOT` (resolved to the main checkout as `footprint` does); `--agent` with `--class`, `--permanent` or `--yes` exits 2; `--agent` without a terminal is the normal case and is not a refusal
- [ ] Selection: `build_plan` with `PlanOptions { consent: Agent { root } }`: default selection as today for `safe`; for `rebuildable`, only items whose path is inside `ROOT`'s main checkout or one of its worktrees; `review`, `user-data` and `native-only` items never selected; the worktree conditions apply to `git.worktree` items
- [ ] `--items` under `--agent`: the list is intersected with that selection; a listed item outside it is reported with the new skip reason `not-consented` (never a refusal); `permanent: true` on a listed item is ignored and noted `permanent-ignored`
- [ ] Rescan: as every clean, a fresh scan of the project's checkouts and the scan roots of `safe` items; the plan gate decides whether a scoped rescan (the checkouts plus the known-path roots of listed `safe` items) is enough, with the same safety argument the DDC-002 story owes for `--items`
- [ ] Journal: `consent: agent`, `project: ROOT` on every line; the existing fields unchanged
- [ ] `--json`: the DDC-002 clean report, with `consent` and the new skip reasons; `docs/clean.schema.json` regenerated
- [ ] Exit codes as DDC-002: 0 done, 1 error, 2 refused, 75 lock held
- [ ] README: a `--agent` section, including what an agent cannot do with it

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (`clean_cli.rs`, temp home with a project, a dead harness worktree inside it, a dirty worktree, an idle `.venv` with `uv.lock` in another project, a `safe` cache and a `review` model directory): `ddc clean --agent --project <root>` removes the dead worktree and the `safe` cache, leaves the other project's `.venv`, the dirty worktree and the model untouched, exits 0, and every journal line has `consent: agent` and `project`
- [ ] Test: the same with `--items -` listing all five: the worktree and the cache are removed; the other three are reported `not-consented` in the `--json` report, and the exit code is 0
- [ ] Test: a listed `review` item with `permanent: true` is skipped `not-consented`, and a listed `rebuildable` item inside the project with `permanent: true` is removed with its rule's method and the note `permanent-ignored`; nothing is deleted permanently that the rule would have trashed
- [ ] Test: `--agent` with `--class safe`, with `--permanent`, with `--yes`, or without `--project`, exits 2 and touches neither the tree nor the journal; `--agent` with stdin and stdout not a terminal runs
- [ ] Test: `--agent --project <a worktree path>` resolves to the main checkout and the selection covers the sibling worktrees; `--project` outside a repository exits 2
- [ ] Test (`clean_safety.rs`): the executor's re-checks run for `--agent` items as for any other (a worktree made dirty after the scan is skipped `changed-since-scan`; an item in use is skipped `in-use`)
- [ ] Test: `clean --agent --json` validates against `docs/clean.schema.json` and the committed schema equals the generated one; `consent` is `agent` in the report
- [ ] On the reference machine: `ddc clean --agent --project <a project with dead worktrees> --dry-run --json` lists exactly the worktrees a manual check calls dead, and the real run removes them with `git worktree list` confirming; result in the final summary
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `--yes` today: `safe` by default, `rebuildable` only with `--class rebuildable`, `review` in `--class` refused (`cmd/clean.rs`, exit 2). `--agent` is that policy with `rebuildable` turned on and scoped by path, so `PlanOptions` gains a consent enum rather than a flag soup: `Interactive`, `Yes { classes }`, `Items { list }`, `Agent { root, items: Option<list> }`.
- The amendment to §7 of the menu bar ADR matters here: `--items` without `--agent` keeps its meaning (a person's list); `--items` with `--agent` is a filter. One `--items` parser, two consent modes.
- ~~A `safe` global cache (`~/.cache/uv`) is removable under `--agent`…~~ *Superseded 2026-11-02 (ADR §7): shared caches are never removed by an agent; it may only propose them to the person.*
- The DDC-002 story's open question (full rescan vs scoped re-walk) decides this story's rescan too; `--agent` is called from inside sessions, so a 40 s rescan is the cost to measure.
- ~~The `not-consented` reason is new… so no `schema_version` bump~~ *Superseded 2026-11-02 (ADR §7): the clean report ships (schema 6, decoded by the app), so `not-consented` and `agent-opt-in-off` bump the clean report and the journal.*

## Dependencies

- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]: blocking
- [[epics/DDC-003/stories/story-worktree-findings-and-provenance|Worktree findings and provenance]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 and §7

## Attachments

-

---

*Last updated: 2026-10-18*
