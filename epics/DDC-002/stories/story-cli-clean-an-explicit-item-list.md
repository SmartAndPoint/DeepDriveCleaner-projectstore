---
type: story
id: "story-cli-clean-an-explicit-item-list"
epic: "DDC-002"
title: "CLI: clean an explicit item list"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [cli, contract, json, cleanup]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# CLI: clean an explicit item list

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The app cleans exactly the items the user confirmed in its sheet. This story gives the CLI that entry point: an explicit item list, a JSON report, a read-only dry run for the sheet's warnings, cancellation, and exit codes the app can act on. It is §6 items 1, 2, 4 and 5 of the ADR "Menu bar app: Swift shell over the ddc CLI", built and tested in Rust without the app.

One change is visible in the terminal: a held lock exits 75 instead of 1. The existing refusals keep exit code 2.

## Decomposition

Rough until the plan gate.

- [ ] `clean --items <file|->`:
  - JSON input `{schema_version, items: [{path, dev, ino, rule, class, action, permanent?}]}`, `action` one of `delete`, `trash`, `native`, `native-only`, and `permanent` only on `review` items;
  - it rescans as every clean does, and runs a listed item only if the fresh plan finds it again with the same path, `(dev, ino)`, rule and class, and derives the same action for the item's `permanent`; anything else is skipped as `changed-since-scan`;
  - the list is the selection: no class filter, no default selection, no prompt (ADR §7); `disabled_rules` never drops a listed item;
  - refused with exit 2: a global `--permanent` together with `--items`, and a malformed list
- [ ] `clean --json`: the execution report on stdout, with `schema_version`: the outcome per item, skip reasons, free space before and after for each volume touched, the journal path. Its schema is generated into `docs/clean.schema.json`
- [ ] `clean --items … --dry-run --json`, for the sheet's warnings, without a rescan (ADR §6 item 2):
  - the executor's per-item re-checks against the listed items only: identity (`(dev, ino)` at the path), class and action derived again from the rule and the project's lock file, kept (`.ddc-keep`, an exclude), in use, owner running, protected;
  - no walk of any unlisted tree;
  - one JSON line per item on stdout as its checks finish, then a summary line;
  - would delete, would trash, would run a command, or would skip with its reason (`owner-running`, `in-use`, `changed-since-scan`, …);
  - no rename, no delete, no Trash, no native command
- [ ] SIGINT and SIGTERM:
  - a scan needs no handler: `last-scan.json` is only ever replaced atomically at the end;
  - a clean installs a handler for both (in the core's `sys` module, the only one allowed `unsafe`, or through `signal-hook`) that sets a cooperative flag `exec::execute` checks between items;
  - a native command runs in its own process group (`CommandExt::process_group`, stable std, no `unsafe`); the handler forwards SIGINT and SIGTERM to that group (`-pgid`), so Ctrl-C in a terminal still reaches it;
  - the clean stops after the current item, and the journal holds every completed action
- [ ] Exit codes: 0 done, 1 error, 2 refused (the existing refusals and the new `--items` ones), 75 another scan or clean holds the lock
- [ ] Plan gate: decide between the full rescan (about 40 s) and a scoped re-walk of the listed findings (ADR, Open Questions); the full rescan stays unless the scoped one is shown to give the same safety

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: `--items` with one valid item and one whose directory was swapped after the scan deletes the first and skips the second as `changed-since-scan`; a listed item of a `user-data` rule is refused (`action-not-allowed`)
- [ ] Test: a listed `review` item with `permanent: false` goes to the Trash, and with `permanent: true` is deleted; an item listed as `review` with action `trash` that the fresh scan classifies `rebuildable` (a lock file appeared) is skipped as `changed-since-scan`, not deleted
- [ ] Test: `--items` together with `--permanent` exits 2, and so does a malformed list; neither touches the tree or the journal
- [ ] Test: `clean --items` runs with stdin and stdout not a terminal (no `no terminal` refusal), as ADR §7 states
- [ ] Test: `--dry-run --json` over a temp tree reports would delete / would trash for its items and an `owner-running` skip for an item whose owner a fake port reports running; afterwards the tree and the journal are unchanged and the fake native runner recorded no call
- [ ] Test: with a fake walker port that counts the directories it opens, a dry run over N listed items in a tree with other, unlisted trees opens no directory outside the listed items; the output is N item lines, each written before the summary line
- [ ] Test: `clean --items --json` output validates against `docs/clean.schema.json`, the committed schema equals the generated one (as for `docs/scan.schema.json`), and it holds free space before and after for each volume touched
- [ ] Unit test: with the cooperative flag set after the first of three items, `exec::execute` returns after that item and reports the other two as not run
- [ ] Test: with a fake native tool on `PATH` that sleeps, SIGTERM to `ddc clean --items` ends the tool and ddc within a second; the journal lists only completed actions
- [ ] Test: Ctrl-C in a terminal (a pseudo-terminal, SIGINT to its foreground process group) while a fake native tool runs during `ddc clean` ends the tool, and the clean stops after that item; the journal lists the items before it
- [ ] Test: SIGTERM during a scan of a temp tree leaves `last-scan.json` unchanged
- [ ] Test: with the lock held, `scan` and `clean` exit 75 (the lock test in `crates/ddc-cli/tests/clean_cli.rs` asserts the code); the existing refusal tests still assert 2
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Consent: §7 of the ADR "Menu bar app: Swift shell over the ddc CLI" makes an explicit item list interactive consent, recorded as an amendment of §5 of the safety ADR. The executor's refusals and re-checks are unchanged.
- Today's codes: refusals return `ExitCode::from(2)` (`crates/ddc-cli/src/cmd/clean.rs`), errors `ExitCode::FAILURE` (`main.rs`), and a held lock surfaces as an error (1).
- The clean report and its skip reasons are new, so `changed-since-scan` adds nothing to an existing schema.
- Today a native command is spawned in the terminal's foreground process group (`crates/ddc-core/src/exec/system.rs`) and gets Ctrl-C itself. Its own process group takes it out of that group, so without forwarding Ctrl-C would stop reaching it: the forwarding of SIGINT is what prevents that regression.
- The dry run predicts; it never replaces the clean's own rescan and re-checks.
- The DDC-001 story "Progress output while ddc clean runs" left Ctrl-C between items out of scope; the SIGTERM handler here is the same mechanism and can serve SIGINT too.

## Dependencies

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-18*
