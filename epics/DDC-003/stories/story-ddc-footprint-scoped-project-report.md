---
type: story
id: "story-ddc-footprint-scoped-project-report"
epic: "DDC-003"
title: "ddc footprint: a scoped project report"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-02
external_refs: {}
tags: [cli, contract, json, agents]
code_refs: []
specs: []
started_at: "2026-11-02T07:27:12.042Z"
closed_at: null
plan_updated_at: "2026-11-02T07:27:12.042Z"
---

# ddc footprint: a scoped project report

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

*Amended 2026-11-02 (ADR "Agent footprint", §7): the command is `ddc agent footprint [ROOT] [--json]`; read `ddc agent footprint` below as that. It needs the worktree probes of story "Worktree findings and provenance" (its PR 2) first.*

`ddc agent footprint [ROOT] [--json]` is the report an agent reads about its own project: the main checkout, every linked worktree with its state, the artifacts inside each, and what is reclaimable now and after the running sessions end. It is §4 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents". It is read-only, takes no lock, writes no `last-scan.json`, and answers in seconds, because it walks only the project's checkouts, not the home folder.

The human output is for a person at a terminal who wonders where a project's gigabytes went; `--json` is the contract the skill and, later, an MCP server read.

## Decomposition

Rough until the plan gate.

- [ ] `ddc agent footprint [ROOT]`: `ROOT` defaults to the current directory; the main checkout is resolved through `.git` (`gitdir:` + `commondir`) when `ROOT` is a worktree or a subdirectory; outside any git repository the command exits 2 with a message
- [ ] Enumeration: `git -C <main> worktree list --porcelain` for the linked worktrees, plus the orphans the walk recognises (see correction 6)
- [ ] Per checkout: `path`, `origin`, and the `worktree` state of "Worktree findings and provenance" (`branch`, `detached`, `clean`, `ignored_content`, `own_commits`, `submodules`, `locked`, `in_use`, `idle_days`, `probe`), obtained through the same probe module under the same contract, and `findings[]` from a scan of that checkout alone with the ordinary rule visitor (so `.venv`, `node_modules`, `target`, caches found inside it are listed with class, bytes, `private_bytes` where measured, and `selected`)
- [ ] Totals: `reclaimable_now` (selected under the ADR's conditions), `reclaimable_after_sessions` (worktrees failing only the in-use or idle condition, with their contents), `held_back` (every other worktree, with its reason and bytes, so an agent can tell the person what holds it; the reasons are listed in correction 13), `review_bytes`; and `harness: {report_age, findings[]}` with the harness-data findings of the last home scan from `last-scan.json`, or `null` with the reason when there is none or its `schema_version` differs
- [ ] `--json`: `docs/footprint.schema.json`, `schema_version` 1, generated with `schemars` and compared with the committed file by a test, as `docs/scan.schema.json`
- [ ] Human output: one block per checkout, the main first, worktrees by size, state words after the path (`clean · unlocked · idle 3d`, `dirty`, `ignored: .env +2`, `submodules`, `own commits: 1`, `in use by claude (pid)`), artifacts indented, totals last with "run `ddc clean <main> [<worktree outside main>…]` (a person's clean; `ddc agent clean` replaces it once it exists) to remove what is selected"
- [ ] Performance: worktree probes in parallel (`rayon`), the walk of each checkout in parallel as `scan` does; a `--timeout` is not added, the probe timeout is the existing 30 s
- [ ] `--worktree-min-age` and `--min-age` accepted, same meaning as on `scan`
- [ ] README: a `footprint` row in the Use table

## Implementation Plan

Planned 2026-11-02 with the planner against `origin/main` 8da91ed. Two PRs.

**Corrections to this story, settled at the plan gate:**
1. Command names follow ADR "Agent footprint" §7: `ddc agent footprint` (renamed in the text above).
2. The suggested command is a person's `ddc clean <main> [<worktree outside main>…]`, never `ddc clean --agent` (no such flag under §7). The cases table records it as shown, not run, since footprint writes no `last-scan.json`.
3. Totals by class, as §7 needs: `safe` bytes inside the tree (an agent may clean them once the person's opt-in is on) apart from `rebuildable` bytes (the person's).
4. **Shape:** checkouts are not scan roots. A scan root is never recognised as a worktree (`RuleVisitor::worktree_region` returns `None` at depth 0), so the walk starts at the checkouts' parents, pruned to the checkouts (a `Reach` over several paths, extending PR 3's `Reach::subtree`), through the walk-and-classify step factored out of `report::scan_inner` (correction 10). Each worktree is then a region at depth ≥ 1, and `classify_probed`/`assess` give the same six conditions, `origin`, `worktree_of` and downgrades as `ddc scan`. *Considered and rejected:* letting the walk take each checkout as a root and seeding it as a region at depth 0. That breaks the invariant that no rule claims a scan root, which the plan and the executor rely on (a scan root is never an item); reading each parent once and descending only into the checkouts costs one directory listing per parent.
5. `reclaimable_after_sessions` needs `assess` to run conditions 4–6 for **every** worktree that failed only condition 2 or 3, a lock included: Claude Code locks a live session's worktree (`locked` = `claude agent <name> (pid N start …)`), and `assess` returns at the lock before the in-use check, so without this the total is about zero while sessions run. A footprint-only `Probing.probe_kept`; the class stays `review`. `in_use` keeps its meaning (a process of the current user works inside the worktree). The lock gets its own field, `WorktreeState.lock: Option<Lock { pid: Option<u32>, live: Option<bool> }>`, filled by a parser in `rules/worktree.rs` that takes `(pid N` from the lock reason whatever comes before it (Claude Code writes `claude agent <name> (pid N start …)` for an agent's worktree, read on the reference machine 2026-11-02, and `claude session <name> (pid N …)`); `live` is whether that pid runs as the current user, `null` without a process table. A lock with `live: true` counts toward `reclaimable_after_sessions`; a lock with no pid or a dead one is `held_back` with the `git worktree unlock` hint. Probing a locked or in-use worktree is safe: every probe is a read under the contract (no optional locks, no helpers), and a lock binds only `remove`/`prune`.
6. Orphans are what `recognize` finds: a `.git` file into a missing `worktrees/<n>` gitdir. A plain folder under `.claude/worktrees/` without a `.git` file is not one. git's list only adds worktrees that lie outside `<main>`. ADR §4's "every directory under `<main>/.claude/worktrees/` that git does not list" carries a dated note.
7. `harness.reason` is `no-report`, `schema-mismatch` or `unreadable` (footprint takes no lock, so it can meet a `last-scan.json` a concurrent scan is writing): a small reader of `last-scan.json`'s `schema_version`.
8. Refuse (exit 2) when `main` is the home folder (`~/.git`): walking it is a home scan. The message says "the main checkout is your home folder; run `ddc scan`", not "outside a git repository". A bare dotfiles repository (`GIT_DIR=~/.dotfiles`) is unaffected. When `commondir` resolves to a bare repository (the common directory is not named `.git` and holds no worktree beside it, read from disk, no git call), there is no main checkout: the report's main entry has no findings and only the linked worktrees are walked, never the bare directory's objects.
9. **Leaks of the parent walk:** the parents are read, and their own files reach the walk's `large_files` and byte totals; the report takes every total from the findings inside the checkouts and drops the walk's totals. A worktree git lists but the walk does not recognise (under a skip or an exclude, `git.worktree` disabled, an unreadable `.git` file) is reported as `not_scanned` with its reason, never dropped silently. When git cannot list worktrees (`worktree_paths` unknown), the report says so instead of showing the main checkout alone.
10. **No Full Disk Access probe:** `scan_inner` reads `TCC.db` (`full_disk_access`) and the volumes; footprint calls a walk-and-classify step factored out of it, and the read-only structure test also forbids `TCC.db` and `full_disk_access`.
11. **JSON contract:** the report's header is `schema_version`, `ddc_version`, `generated_at`, `duration_ms`, as `AgentsReport`'s; `prints_json` gets the `Footprint` arm. Exit 2 under `--json` prints an error report with the new codes `not-a-repository` and `home-is-the-checkout` (`ERROR_SCHEMA_VERSION` bumps; DDCKit's decoder follows). `ErrorReport::new` takes a `&Refusal`, so these errors need a second constructor. Without an in-use check (a platform without the process table), `in_use` is `null`, not `false`.
12. *(Merged into 6.)*
13. **Every checkout lands in exactly one total.** `reclaimable_now`: selected under the six conditions. `reclaimable_after_sessions`: fails only condition 2 (a live lock, or in use) or 3 (idle), the rest probed clean (correction 5). `held_back`, with the reason: the first failed condition of 4–6 (`submodules`, `dirty`, `ignored-content`, `own-commits`), a dead or pid-less lock (`stale-lock`), `probe-unknown`, `not-linked` (copied or moved, `linked: false`), `not-a-harness` (a harness `ddc` does not trust, such as Codex), `orphan` (a `git.worktree-orphan`: `review`, method `trash`, ADR §2), `yours` (`origin.harness: git`: listed with its state, removed only when a person lists it). `not_scanned`: correction 9. The main checkout is in none of them; its inner findings count by class. The schema's description of `reclaimable_after_sessions` says it is an estimate that assumes the worktree stays as it is.

**PR A: `ddc agent status`, `disk agents` hidden (minor).** No dependency on PR 3.
- `cli.rs`: `Command::Agent { action: AgentAction }` ("coding agents"), `AgentAction::Status(AgentsArgs)`; `DiskAction::Agents` hidden; `prints_json`; the LaunchAgent help strings say "LaunchAgent".
- `git mv cmd/disk_agents.rs cmd/agent_status.rs`; both variants dispatch to it.
- `gate.rs`: `Agent{Status}` → `CodingAgents`; `EVERY` and a refusal test.
- cases: an `agent status` case beside the kept `disk agents` case, `COMMANDS`, the system-root arm, `refusal()`, `data_dir_commands`; `unported_cli.rs`; a test that both names give the same report and that neither `--help` lists `agents` under `disk`; native-skips where the `disk_agents_cli` names are.
- README: a `### ddc agent` section; `disk agents` lines leave `### ddc disk`.
- Guidance: skill and snippet call `ddc agent status`, naming `ddc disk agents` as the fallback for an older binary; `structure.rs` lists move.
- The app keeps calling `disk agents` (the alias).

**PR B: `ddc agent footprint` (minor), after PR 3 merges** (it reuses `Git::worktree_paths`, `Reach::subtree`, `removable_inside`).
- New core module `crates/ddc-core/src/footprint/` (`mod.rs`, `model.rs`), beside `disk/` and `agents/`.
- Resolve ROOT → `main`; enumerate checkouts through `git::Git` (probe contract, `GIT_*` cleared, never `/usr/bin/git` on macOS); walk as in correction 4 with settings' skips, excludes and disabled rules; no lock, no `write_last_scan`, no notification.
- `FootprintReport` reuses `model::Finding`; totals by class, `reclaimable_now`, `reclaimable_after_sessions`, `harness` with its reason; paths through `json_path`; `docs/footprint.schema.json` with `committed_footprint_schema_is_current`. `WorktreeState.lock` (correction 5) changes the scan report too: optional with `skip_serializing_if`, so the scan's `SCHEMA_VERSION` stays, but `docs/scan.schema.json` is regenerated (`DDC_UPDATE_SCHEMA=1`) and `bench/golden` re-recorded (`make golden-record`, a diff from this field only).
- CLI `AgentAction::Footprint`, `cmd/agent_footprint.rs`, `render::footprint`; no gate capability, as `scan`.
- A structure test keeps `footprint/*` and `agent_footprint.rs` read-only (no `Executor`, `plan::`, `trash`, `remove_`, `try_lock`, `write_last_scan`, `Command::new`).

**Pitfalls:** no gate input names a Rust-free path; no public doc links a `pub(crate)` item; `platform::Canonical` and `meta::stat`, never std `canonicalize`/`file_id`; real-git fixtures clear `GIT_*`; Windows skips for git tests, none for Linux; coverage floors (ddc-cli 82.0, classify.rs 95.6: new code goes in `worktree.rs`); builds for Windows and Linux run on the Windows machine, not this Mac.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (a new `footprint_cli.rs`, temp repository with four harness worktrees under `<main>/.claude/worktrees/`, unlocked, run with `--worktree-min-age 0`: A clean, B dirty, C clean with an ignored `.venv` holding `pyvenv.cfg` and `uv.lock`, D clean with a unique ignored `.env`): `ddc agent footprint <path of A>` resolves to the main checkout and lists all four with the expected `worktree` fields; `reclaimable_now` = bytes of A + bytes of C (C's `.venv` is a claimed `delete` finding, counted once); B is `held_back` with `dirty`, D with `ignored-content` and `ignored_content: [".env"]`; the `.venv` is listed under C with `project.worktree_of`; exit code 0
- [ ] Test: run from a subdirectory of the main checkout, the same output; run outside a repository, exit 2 and no output on stdout
- [ ] Test: a directory under `<main>/.claude/worktrees/` whose `.git` file names a missing `<main>/.git/worktrees/<n>` is reported as `git.worktree-orphan`; a plain folder there without a `.git` file is not
- [ ] Test: `footprint` with a fake walker port opens no directory outside the main checkout, its worktrees and their parents (read, never descended into beyond the checkouts); with the data directory's lock held by another process it still runs (no lock taken); afterwards `last-scan.json` is unchanged or absent, every worktree's index bytes and mtime are unchanged, and every git invocation recorded by the fake `CommandRunner` carries the probe contract's flags
- [ ] Test: with `last-scan.json` of the current schema present, `harness.findings` holds its `agent.*` findings and `report_age` is set; with a different `schema_version`, `harness` is `null` with reason `schema-mismatch`; with no file, reason `no-report`
- [ ] Test: `--json` validates against `docs/footprint.schema.json`; the committed schema equals the generated one
- [ ] Test: `--worktree-min-age 0` moves a clean worktree touched now from `reclaimable_after_sessions` to `reclaimable_now`
- [ ] On the reference machine: `ddc agent footprint` in a project with at least ten worktrees, each with an environment, at least one of them locked by a live session, finishes in under 5 s (median of three runs after a warm-up) and its `reclaimable_now` agrees with `ddc scan <main>` on the same checkouts; numbers in the final summary
- [ ] `make check` passes

- [ ] Test (footprint, correction 5): a worktree locked by a live pid of the current user (fake process table) and otherwise clean counts in `reclaimable_after_sessions` with `locked: true` and `lock: {pid, live: true}`, `in_use` unchanged; the lock parser reads the pid from both `claude agent …` and `claude session …` reasons; one whose lock names a dead pid is `held_back` with the unlock hint; one a process works in (fake cwd) counts in `reclaimable_after_sessions`
- [ ] Test (footprint, corrections 3 and 9): totals are named by class (`safe`, `rebuildable`), a clean idle worktree's bytes count under `rebuildable`, its inner `.venv` counts once, and a large file beside the main checkout is in no total; a worktree under an exclude is `not_scanned` with the reason
- [ ] Test (footprint, correction 11): `--json` outside a repository exits 2 with an error report `not-a-repository`; with `~/.git` it is `home-is-the-checkout`; the error schema is current; the report header carries `schema_version`, `ddc_version`, `generated_at`, `duration_ms`
- [ ] Test (correction 13): each checkout of the fixture is in exactly one of `reclaimable_now`, `reclaimable_after_sessions`, `held_back` and `not_scanned`; a worktree whose probe fails is `held_back` with `probe-unknown`; a `git worktree add ../y` worktree is `held_back` with `yours`
- [ ] Structure test: `footprint/*` and `agent_footprint.rs` mention none of `Executor`, `plan::`, `build_plan`, `trash`, `remove_`, `try_lock`, `write_last_scan`, `Command::new`, `TCC.db`, `full_disk_access`

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- *Superseded at the plan gate (correction 4):* the first draft scanned the checkouts as roots and built the worktree entries from git's list. The walk now starts at the checkouts' parents, so each worktree is a region and gets its findings from the same rules as `ddc scan`.
- "Reclaimable after sessions end" is an estimate like the scan's "frees": it assumes the worktree stays clean. The human output says "if left as it is".
- No lock: `footprint` never writes, and the shared scan lock exists to keep `clean` from racing a writer; a concurrent `clean` can remove a worktree while `footprint` lists it. Nothing acts on the footprint directly: an agent never removes a worktree (§7), and a person's clean re-checks every condition before it runs.
- Full Disk Access is not needed: project checkouts are outside `~/Library`. If a root lies in a TCC-protected region (`~/Documents`), the walk may block on a prompt in a background process, as the safety ADR §8 observed; `footprint` is meant for an interactive session, and the README says so.

## Dependencies

- [[epics/DDC-003/stories/story-worktree-findings-and-provenance|Worktree findings and provenance]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[epics/DDC-001/stories/story-ddc-scan-report|ddc scan report]]: the report model and the schema test pattern

## Attachments

-

---

*Last updated: 2026-11-02*
