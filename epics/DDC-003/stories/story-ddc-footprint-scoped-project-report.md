---
type: story
id: "story-ddc-footprint-scoped-project-report"
epic: "DDC-003"
title: "ddc footprint: a scoped project report"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-02
external_refs: {}
tags: [cli, contract, json, agents]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# ddc footprint: a scoped project report

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

*Amended 2026-11-02 (ADR "Agent footprint", §7): the command is `ddc agent footprint [ROOT] [--json]`; read `ddc footprint` below as that. It needs the worktree probes of story "Worktree findings and provenance" (its PR 2) first.*

`ddc footprint [ROOT] [--json]` is the report an agent reads about its own project: the main checkout, every linked worktree with its state, the artifacts inside each, and what is reclaimable now and after the running sessions end. It is §4 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents". It is read-only, takes no lock, writes no `last-scan.json`, and answers in seconds, because it walks only the project's checkouts, not the home folder.

The human output is for a person at a terminal who wonders where a project's gigabytes went; `--json` is the contract the skill and, later, an MCP server read.

## Decomposition

Rough until the plan gate.

- [ ] `ddc footprint [ROOT]`: `ROOT` defaults to the current directory; the main checkout is resolved through `.git` (`gitdir:` + `commondir`) when `ROOT` is a worktree or a subdirectory; outside any git repository the command exits 2 with a message
- [ ] Enumeration: `git -C <main> worktree list --porcelain` for the linked worktrees, plus every directory directly under `<main>/.claude/worktrees/` that git does not list (an orphan, shown as `git.worktree-orphan`)
- [ ] Per checkout: `path`, `origin`, and the `worktree` state of "Worktree findings and provenance" (`branch`, `detached`, `clean`, `ignored_content`, `own_commits`, `submodules`, `locked`, `in_use`, `idle_days`, `probe`), obtained through the same probe module under the same contract, and `findings[]` from a scan of that checkout alone with the ordinary rule visitor (so `.venv`, `node_modules`, `target`, caches found inside it are listed with class, bytes, `private_bytes` where measured, and `selected`)
- [ ] Totals: `reclaimable_now` (selected under the ADR's conditions), `reclaimable_after_sessions` (worktrees failing only the in-use or idle condition, with their contents), `held_back` (worktrees `review` for ignored content, submodules, own commits or a lock, with the reason and bytes, so an agent can tell the person what holds them), `review_bytes`; and `harness: {report_age, findings[]}` with the harness-data findings of the last home scan from `last-scan.json`, or `null` with the reason when there is none or its `schema_version` differs
- [ ] `--json`: `docs/footprint.schema.json`, `schema_version` 1, generated with `schemars` and compared with the committed file by a test, as `docs/scan.schema.json`
- [ ] Human output: one block per checkout, the main first, worktrees by size, state words after the path (`clean · unlocked · idle 3d`, `dirty`, `ignored: .env +2`, `submodules`, `own commits: 1`, `in use by claude (pid)`), artifacts indented, totals last with "run `ddc clean --agent --project <main>` to remove what is selected"
- [ ] Performance: worktree probes in parallel (`rayon`), the walk of each checkout in parallel as `scan` does; a `--timeout` is not added, the probe timeout is the existing 30 s
- [ ] `--worktree-min-age` and `--min-age` accepted, same meaning as on `scan`
- [ ] README: a `footprint` row in the Use table

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (a new `footprint_cli.rs`, temp repository with four worktrees: one clean and idle, one dirty, one with a `.venv` and `uv.lock`, one clean with a unique ignored `.env`): `ddc footprint <worktree path>` resolves to the main checkout and lists all four with the expected `worktree` fields, the last in `held_back` with `ignored_content: [".env"]`, the `.venv` under its worktree with `project.worktree_of`, `reclaimable_now` equal to the clean worktree's bytes, and exit code 0
- [ ] Test: run from a subdirectory of the main checkout, the same output; run outside a repository, exit 2 and no output on stdout
- [ ] Test: a directory under `<main>/.claude/worktrees/` that `git worktree list` does not list is reported as `git.worktree-orphan`
- [ ] Test: `footprint` with a fake walker port opens no directory outside the main checkout and its worktrees; with the data directory's lock held by another process it still runs (no lock taken); afterwards `last-scan.json` is unchanged or absent, every worktree's index bytes and mtime are unchanged, and every git invocation recorded by the fake `CommandRunner` carries the probe contract's flags
- [ ] Test: with `last-scan.json` of the current schema present, `harness.findings` holds its `agent.*` findings and `report_age` is set; with a different `schema_version`, `harness` is `null` with reason `schema-mismatch`; with no file, reason `no-report`
- [ ] Test: `--json` validates against `docs/footprint.schema.json`; the committed schema equals the generated one
- [ ] Test: `--worktree-min-age 0` moves a clean worktree touched now from `reclaimable_after_sessions` to `reclaimable_now`
- [ ] On the reference machine: `ddc footprint` in a project with at least ten worktrees, each with an environment, finishes in under 5 s (median of three runs after a warm-up) and its `reclaimable_now` agrees with `ddc scan <main>` on the same checkouts; numbers in the final summary
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The scan of one checkout reuses `scan::walk` with the rule visitor on a root list of the checkouts; the worktree rule then claims each root's worktrees as regions. Because every root is itself a checkout, the "a rule never claims a scan root" restriction means the worktree directory is not a finding of its own scan; `footprint` builds the worktree entries from the enumeration and attaches the inner findings, rather than scanning `<main>` as one root (which would also claim nested worktrees twice).
- "Reclaimable after sessions end" is an estimate like the scan's "frees": it assumes the worktree stays clean. The human output says "if left as it is".
- No lock: `footprint` never writes, and the shared scan lock exists to keep `clean` from racing a writer; a concurrent `clean` can remove a worktree while `footprint` lists it, and the agent's next `clean --agent` re-checks anyway.
- Full Disk Access is not needed: project checkouts are outside `~/Library`. If a root lies in a TCC-protected region (`~/Documents`), the walk may block on a prompt in a background process, as the safety ADR §8 observed; `footprint` is meant for an interactive session, and the README says so.

## Dependencies

- [[epics/DDC-003/stories/story-worktree-findings-and-provenance|Worktree findings and provenance]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[epics/DDC-001/stories/story-ddc-scan-report|ddc scan report]]: the report model and the schema test pattern

## Attachments

-

---

*Last updated: 2026-10-18*
