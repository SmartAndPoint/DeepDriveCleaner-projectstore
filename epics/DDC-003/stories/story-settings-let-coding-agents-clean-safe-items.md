---
type: story
id: "story-settings-let-coding-agents-clean-safe-items"
epic: "DDC-003"
title: "Settings: let coding agents clean safe items"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-14
updated: 2026-11-14
external_refs: {}
tags: [cli, app, settings, agents, consent]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# Settings: let coding agents clean safe items

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The person's switch for ADR "Agent footprint" §7: `ddc agent clean` removes `safe` items only while `settings.json` has `agents_clean_safe: true`, and story "Clean: --agent bounded consent" reads the key but gives no way to switch it except editing the file. This story adds the switch the ADR names.

- `ddc settings set agents_clean_safe on|off`: the one CLI writer of the key, through the settings store's read-modify-write; it records the change in the action journal (a `settings-set` line, journal schema bump) and prints the new state.
- The app's Settings shows "Let coding agents clean safe items", off by default, with the risk in words and a link to the disclaimer; switching it runs the CLI command, never writes the file itself. While it is on, the popover names it in one line, and Settings switches it off in one place.
- The menu bar ADR §6 item 6 ("The CLI writes [settings.json] only to clear a stale scheduler record") gets a dated amendment for this second writer.

## Decomposition

- [ ] CLI `ddc settings set agents_clean_safe on|off`, journal line `settings-set`, schema bump, README
- [ ] DDCKit: run the command, the Settings toggle, the popover line; ContractTests: the toggle is the one caller
- [ ] Menu bar ADR §6 item 6: the dated amendment

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (CLI): `ddc settings set agents_clean_safe on` writes the key, keeps every other key and the file's mode, journals one `settings-set` line; `off` removes the key; an unreadable `settings.json` exits non-zero and writes nothing
- [ ] Test (DDCKit): switching the toggle runs the command once and shows the CLI's state; the popover line shows only while the key is on; nothing scheduled runs the command
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): the toggle's words and the disclaimer link read right; with it on, an agent's `ddc agent clean` removes the safe items, and with it off removes nothing

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Split out of story "Clean: --agent bounded consent" at its plan gate (2026-11-14, the owner's pick): that story reads `agents_clean_safe`; this one writes it.
- Once the key is on, a `ddc` older than story A's release refuses the whole file; the toggle's text says so.
- Whether `ddc settings set agents_clean_safe off` waits for the exclusive lock (so a running `ddc agent clean` finishes first) is this story's plan gate.
- Two writers of `settings.json` then exist (the app's `SettingsStore.update` and this command): both read, change one key and replace the file atomically; the race and its handling are this story's plan gate.

## Dependencies

- [[epics/DDC-003/stories/story-clean-agent-bounded-consent|Clean: --agent bounded consent]]: blocking (the key, `SettingsFile`)

## Attachments

-

---

*Last updated: 2026-11-14*
