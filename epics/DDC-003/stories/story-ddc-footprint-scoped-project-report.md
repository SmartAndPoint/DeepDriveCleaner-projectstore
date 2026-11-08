---
type: story
id: "story-ddc-footprint-scoped-project-report"
epic: "DDC-003"
title: "ddc footprint: a scoped project report"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-08
external_refs: {}
tags: [cli, contract, json, agents]
code_refs: []
specs: []
started_at: "2026-11-02T07:27:12.042Z"
closed_at: null
plan_updated_at: "2026-11-08T00:00:00.000Z"
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
- [ ] Human output: one block per checkout, the main first, worktrees by size, state words after the path (`clean · unlocked · idle 3d`, `dirty`, `ignored: .env +2`, `submodules`, `own commits: 1`, `in use by claude (pid)`), artifacts indented, totals last with "run `ddc clean <main> [<worktree outside main>…]` (a person's clean; `ddc agent clean` replaces it once it exists) to remove what is selected" *(superseded by the revision of 2026-11-08: the command is `ddc clean <main>`, and selected findings outside it are listed by path)*
- [ ] Performance: worktree probes in parallel (`rayon`), the walk of each checkout in parallel as `scan` does; a `--timeout` is not added, the probe timeout is the existing 30 s
- [ ] `--worktree-min-age` and `--min-age` accepted, same meaning as on `scan`
- [ ] README: a `footprint` row in the Use table

## Implementation Plan

Planned 2026-11-02 with the planner against `origin/main` 8da91ed. Two PRs.

**Corrections to this story, settled at the plan gate:**
1. Command names follow ADR "Agent footprint" §7: `ddc agent footprint` (renamed in the text above).
2. *(Superseded by the revision of 2026-11-08.)* The suggested command is a person's `ddc clean <main> [<worktree outside main>…]`, never `ddc clean --agent` (no such flag under §7). The cases table records it as shown, not run, since footprint writes no `last-scan.json`.
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

**Revision 2026-11-08, against `origin/main` 3e487eb** (planner re-run; the owner put this story before "Clean: --agent bounded consent", which reuses its scope and walk). What changed since the plan above:

- **Correction 2, replaced.** A worktree outside `<main>` passed as a scan root is never recognised (`Catalog::worktree_region` returns `None` at depth 0), so `ddc clean <main> <worktree>` would not clean it; and naming its parent instead widens the clean to every sibling (`~/Projects/foo` with `../foo-wt` normalises to `ddc clean ~/Projects`, a worktree in the home to `ddc clean ~`). So the suggested command is `ddc clean <main>` alone, through `render::clean_command`. Selected findings outside `<main>` (rare: an outside worktree is almost always `yours` or `not-a-harness`) are listed by path under "outside the main checkout: review them in the app's Review window", with no command that would widen the clean. The cases table records the command as shown, not run, with a pattern narrow to the fixture's path. `ddc agent clean` will offer `--items` lists for the person instead (ADR §7).
- **Correction 4, renamed.** The method is `Catalog::worktree_region`; `Reach` gains `subtrees(&[PathBuf])`, and `subtree` becomes `subtrees(&[p])`.
- **Correction 5, moved.** The lock is a field of footprint's checkout entry, `lock: { pid, live }`, not of the scan's `WorktreeState`: the scan report, `docs/scan.schema.json` and the goldens do not change (no golden holds a worktree finding). The parser of `(pid N` lives in `rules/worktree.rs`; `live` is a process of that pid owned by the current user, from the process table. The `start …` part of the reason is not compared: its format is not confirmed, so `live` is an estimate, and the schema says so. `assess` gets a mode, `Scan` (as today) or `Footprint`, in which a worktree that fails condition 2 or 3 is still probed for 4 to 6; its class stays `review`. The mode is a field of `Probing`, set by a parameter of `walk_classified`; it is not in `ScanOptions`, so neither a scan nor a clean (nor `agent clean`, which reuses `Scope::scan_options`) can carry it, and `scan_inner` always passes `Scan` (a test pins it). A candidate that failed 2 or 3 carries that failure: its probes record their facts, but an all-clear never makes it `rebuildable` and never replaces the hint naming the lock, the process or the idle window.
- **Correction 7, reason kept, cause corrected.** `last-scan.json` is replaced atomically, so footprint never reads half a file; `unreadable` is an I/O error or a report of the right version that does not decode.
- **Correction 9, kept as a guard.** The walk prunes a parent's other entries before any rule hook, so a sibling is never claimed; the leak that remains is the parent's own files reaching `large_files` and the walk's totals, which the report drops. Keeping only findings inside a checkout stays as a guard. `not_scanned` reasons: `skipped` (under a skip), `not-recognized`, `missing` (git lists it, the folder is gone), `other-filesystem`. A worktree under an exclude or holding `.ddc-keep` is not here but `held_back: kept`; it is a claim, not a region, so its entry has no `worktree` and no `origin`, and the schema allows that. Paths git lists are canonicalised before they are matched with the walk, by `(dev, ino)`; one that does not canonicalise is `missing`, and its parent is never a walk root, so a stale `git worktree add /Volumes/…` entry never fails the walk (the scan's root normalisation errors on a missing root).
- **Correction 11, numbers.** The error schema is 3 today (`root-not-owned`, `busy`); `not-a-repository` and `home-is-the-checkout` make it 4, with two constructors beside `root_not_owned` and `busy`, and DDCKit's `BusyReport.errorSchemaVersion`, its fixture and tests in the same PR. `in_use` is absent, not `null`, without a process table, as `WorktreeState` serialises it.
- **Correction 13, more reasons.** `held_back` also takes `immutable`, `foreign-owner`, `kept` (an exclude or a `.ddc-keep` makes the worktree `user.kept`), `hidden-by-settings` (under `min_size`), `activity-unknown` (condition 3 has no signal) and `rule-disabled`.
- **Main checkout resolution.** From the canonical ROOT up to the first ancestor with `.git`: a directory is the main checkout; a file read by `recognize` gives a linked worktree's main, an orphan sends the search higher, and any other `.git` file (a submodule's, one `recognize` cannot read or that names a gitdir without `commondir`) makes that directory the main checkout, after which git decides; a `.git` file naming a gitdir on a network volume runs no git (a 30 s timeout there) and the listing is `probe-unknown`. No ancestor with `.git` is `not-a-repository`; a main checkout that is the home folder, by identity, is `home-is-the-checkout`.
- **Names.** The module stays `crate::footprint`; its types avoid `agents::footprint`'s names (`CheckoutEntry`, `Bucket`, `HeldReason`).
- **Two PRs, rebase-merged.**
  - **PR B1, seams with no change in behaviour** (all existing tests and goldens unchanged): `report::walk_classified` factored out of `scan_inner` (the walk, `Probing`, `classify_probed`, findings, the immutable check, sorting; `scan_inner` keeps totals, Full Disk Access and volumes); `Reach::subtrees`; in `rules/worktree.rs` the `Held` reasons recorded per worktree, the `Assess` mode and the lock parser; `Classified.held`.
  - **PR B2, the command:** `footprint/{mod,scope,harness,model}.rs`; `ddc agent footprint [ROOT] [--json] [--min-age] [--worktree-min-age] [--data-dir]`; `cmd/agent_footprint.rs`; a downcast of `footprint::Refused` in `main.rs` to exit 2 with the envelope; `gate.rs` with no capability, as `disk docker`; `render::footprint`; `docs/footprint.schema.json`; the error schema 4; tests; cases (a fixture repository under `{root}/repo`), `structure.rs`, `unported_cli.rs`, native skips, README.
- **For `ddc agent clean`.** `footprint::Scope { root, main, bare, listing, checkouts, missing }` with `resolve`, `walk_roots`, `reach`, `scan_options` and `checkout_of`, and `footprint::scoped` over `walk_classified`. `agent clean` resolves the same `Scope`, runs a pruned scan over `walk_classified` with `scope.reach()` (a new seam: `recheck::pruned` builds its `Reach` from a list and walks `opts.roots` today), and keeps `safe`, selected findings inside a checkout, so footprint and the clean classify alike by construction. It must not write `last-scan.json` from that scoped walk: its roots are the parents, so `same_roots` is false by construction and the app's home report would be replaced by a cut one.
- **Also.** Footprint keeps `require_own_roots` (it moves into `walk_classified`, and a test pins the refusal under an elevated terminal where that applies); an unreadable `settings.json` stops it as it stops `ddc scan`. The 5 s budget is at risk in the `Footprint` mode, which probes every harness worktree, a locked or fresh one included, and `git status --ignored=matching` lists every file of a `.venv`; the reference-machine run decides, and a cut (probing only worktrees that passed 1) is the fallback.
- **The reference-machine criterion** reports no timings in this public vault (owner's rule on machine measurements): the final summary says whether it finished within 5 s and whether `reclaimable_now` agreed with `ddc scan <main>`.

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
- [ ] On the reference machine: `ddc agent footprint` in a project with at least ten worktrees, each with an environment, at least one of them locked by a live session, finishes in under 5 s (median of three runs after a warm-up) and its `reclaimable_now` agrees with `ddc scan <main>` on the same checkouts; the final summary says whether both held, with no timings (revision 2026-11-08)
- [ ] `make check` passes

- [ ] Test (footprint, correction 5): a worktree locked by a live pid of the current user (fake process table) and otherwise clean counts in `reclaimable_after_sessions` with `locked: true` and `lock: {pid, live: true}`, `in_use` unchanged; the lock parser reads the pid from both `claude agent …` and `claude session …` reasons; one whose lock names a dead pid is `held_back` with the unlock hint; one a process works in (fake cwd) counts in `reclaimable_after_sessions`
- [ ] Test (footprint, corrections 3 and 9): totals are named by class (`safe`, `rebuildable`), a clean idle worktree's bytes count under `rebuildable`, its inner `.venv` counts once, and a large file beside the main checkout is in no total; a worktree under a skip is `not_scanned: skipped`, and one under an exclude or holding `.ddc-keep` is `held_back: kept` (revision 2026-11-08)
- [ ] Test (footprint, correction 11): `--json` outside a repository exits 2 with an error report `not-a-repository`; with `~/.git` it is `home-is-the-checkout`; the error schema is current; the report header carries `schema_version`, `ddc_version`, `generated_at`, `duration_ms`
- [ ] Test (correction 13): each checkout of the fixture is in exactly one of `reclaimable_now`, `reclaimable_after_sessions`, `held_back` and `not_scanned`; a worktree whose probe fails is `held_back` with `probe-unknown`; a `git worktree add ../y` worktree is `held_back` with `yours`
- [ ] Test (revision 2026-11-08): the suggested command is `ddc clean <main>` for a repository with an outside worktree, never its parent nor the home; a worktree git lists whose folder is gone is `not_scanned: missing` and the run exits 0; a bare repository's linked worktrees are walked and the bare directory is not; an orphan with no repository above is `not-a-repository`; a submodule ROOT resolves to the submodule; `scan_inner` assesses in `Scan` mode
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
