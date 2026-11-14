---
type: story
id: "story-clean-agent-bounded-consent"
epic: "DDC-003"
title: "Clean: --agent bounded consent"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-14
external_refs: {}
tags: [cli, safety, cleanup, agents, contract]
code_refs: []
specs: []
started_at: "2026-11-14T07:17:15.793Z"
closed_at: null
plan_updated_at: "2026-11-14T07:17:15.793Z"
status_changed_at: "2026-11-14T07:17:15.793Z"
---

# Clean: --agent bounded consent

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

*Rewritten 2026-11-14 at the plan gate to ADR "Agent footprint" §7 (amendment of 2026-11-02); the `--agent` flag of the first draft is gone, and so is removing `rebuildable` items without a person. The owner split the work three ways (2026-11-14): this story; story "Settings: let coding agents clean safe items" (the switch); story "ddc agent clean --session: clean up after one session".*

`ddc agent clean [ROOT] [--items <file|->] [--dry-run] [--json] [--data-dir DIR]` is the one cleanup entry point for a coding agent. ROOT (default: the current folder) is any folder of a project; it resolves to the main checkout and its worktrees as `ddc agent footprint` does.

- **What it removes.** Only `safe` project artifacts (category `project-artifact`, method `delete`, selected by default) that lie strictly inside the main checkout or a linked worktree the walk recognises as this project's (its `git.worktree` finding, even one hidden by `min_size` or a disabled rule), none at or above the home folder by identity (`(dev, ino)`, as `resolve_main` compares it), and none whose parent is another account's (such a checkout leaves the bound, its items are `not-consented` and the report says why, so it never stops the run), and only when `settings.json` has `agents_clean_safe: true` (off by default). Never a `rebuildable`, `review` or `user-data` item, a tool's command (`native`, `native-only`), a worktree itself, a checkout at or above the home folder, or anything outside the checkouts: no shared cache (`~/.cargo`, uv, DerivedData, editor caches).
- **Without the opt-in, with `--dry-run`, or with a settings file that cannot be read** it removes nothing, takes no lock, writes no journal line and exits 0. The report lists every item it considered (each finding inside the bound's checkouts that `build_plan`'s filters let through: not user data, not ignored, with an action, not a scan root), sorted by path: `safe` ones skipped `agent-opt-in-off`, the rest `not-consented`, and states `agents_clean_safe`. Under `--dry-run` with the opt-in on, what would go is `not-run` and the report says `dry_run: true`. When the settings cannot be read it considers nothing, since it cannot know the skips, and says why (`settings_error`). When the settings read again under the lock say off, the lock is released and the report is the one without the opt-in; a switch off after that second read is seen by the next run, not this one.
- **For the person.** `rebuildable` items selected by default inside the checkouts (a dead harness worktree, an idle `.venv`), and the `safe` ones while the opt-in is off, are listed with the exact command: the text output prints a `ddc clean --items -` command fed by a heredoc holding the list itself, so the person sees item by item what they agree to (the owner, 2026-11-14). The delimiter is quoted (`<<'DDC_ITEMS'`), so the shell expands nothing in the list, a path with `$`, a backquote or a quote included; one item per line, its path first. When an item lies outside the scan roots `settings.json` gives, the command names the checkouts' parents as its folders, so `ddc clean --items` reaches them (its re-walk stays pruned to the list). There is no command, only the Review window, when a path holds a control or bidi character or is not UTF-8, or the list has more than 50 items (then: save `for_person` to a file and run `ddc clean --items FILE`); on Windows, and for a shell without heredocs (fish), the text says the same. `--json` carries the list in `for_person`, an item list of schema 1 that `ddc clean --items` reads, absent when it would be empty; under `--items` it holds only listed items. Nothing inside a listed worktree is listed again.
- **`--items` filters, never widens.** A listed item outside the bound is `not-consented`, whatever its class or `permanent`; one inside the bound that the walk does not confirm is skipped as `clean --items` skips it (`changed-since-scan`, `rule-mismatch`, `kept`). A malformed list exits 2 before anything is touched.
- **A real run** takes the exclusive lock (holder `agent-clean`, so the app's Busy panel names it; the owner, 2026-11-14), reads the settings again under it, walks only the checkouts with the scan's own rules (`Assess::Scan`, so the worktree the agent runs in is in use), and runs the executor with `Consent::Agent`: every re-check of the safety ADR §5 applies unchanged. Every journal line carries `consent: agent` and `project` (the main checkout). It never writes `last-scan.json`, so the app's report shows removed items until the next scan, where they are `missing` (a known cost). The bound holds the worktrees of other live sessions too: their safe items can go while they build, which only the `in-use` check of open files guards; `--session` (story C) narrows it. The bound is the owner's consent rule for an honest agent, not a defence against one that runs `rm` itself.
- `ddc agent status` reports `agents_clean_safe`.
- **Contracts.** Skip reasons `not-consented` and `agent-opt-in-off`: clean report 13, journal 13, progress 11, with DDCKit's decoders; holder `agent-clean`: lock 2, error 5. `agents_clean_safe` is an additive settings key, written only when true; `SETTINGS_SCHEMA_VERSION` stays 1. Once it is on, a `ddc` older than this release refuses the whole file; PR 1 makes that refusal name the key and say to update ddc. Until story B, the key is set by editing the file; the app keeps it (`SettingsFile` decodes it and writes it back as read, never from its own state). `ddc agent status` reports false, with the settings error, when the file cannot be read.
- **Exit codes.** 0 done (nothing removed included); 1 an error, or without `--json` an item that failed; 2 refused before anything (not a repository, the home folder is the checkout, a malformed list, run as root, checked first); 69 platform; 75 lock held (real run only).

Out of scope: switching the opt-in (story "Settings: let coding agents clean safe items"); `--session <id>` (story "ddc agent clean --session: clean up after one session"); the guidance's use of `ddc agent clean` (story "Agent guidance: skill and AGENTS.md snippet").

## Decomposition

- [ ] PR 1, contracts, no command: `SkipReason::{NotConsented, AgentOptInOff}`; the consent kind and optional `consent`/`project` on journal item lines; optional clean report fields `consent`, `project`, `agents_clean_safe`, `dry_run`, `for_person`, `settings_error`; `HolderCommand::AgentClean`; versions clean 13, journal 13, progress 11, lock 2, error 5; `Settings.agents_clean_safe`; schemas regenerated; fuzz regression; `prompt::reason`; DDCKit decoders, wording, `SettingsFile`, `SharedSettings.keys`, fixtures, ContractTests
- [ ] PR 2, core: a scoped re-walk seam in `report::recheck`; `plan::{AgentBound, agent_may_remove, plan_agent}`; `Consent::Agent` in the executor (`allowed`, the worktree re-check's `person`, journal tags); `CleanReport::for_agent`; core tests
- [ ] PR 3, command: `ddc agent clean` (`cli.rs`, `gate.rs`, `cmd/agent_clean.rs`, `render::agent_clean` with the person's command); `agents_clean_safe` in `ddc agent status`; cases, structure, unported, native skips; README; DDCKit ContractTests: the app never runs `agent clean`

## Implementation Plan

Planner run 2026-11-14 against `origin/main` 1ab9404 (after the footprint story's PRs #148 to #150). Three PRs, in the order of the Decomposition.

**PR 1, contracts.** `exec/mod.rs` gains the two skip reasons and a consent kind (`interactive`, `yes`, `listed`, `agent`); `exec/journal.rs` the optional `consent` and `project` (through `json_path`), journal 13; `exec/report.rs` the optional report fields, clean 13; `progress.rs` 11 (the reasons travel in item events); `holders.rs` `AgentClean`, lock 2, and `refusal.rs` error 5 (busy envelopes carry holders); `settings.rs` `agents_clean_safe` (serde default, skipped when false); `prompt::reason`, the one exhaustive Rust match on `SkipReason`; schemas regenerated; the settings fuzz regression; every literal version in tests; DDCKit: `CleanReport`, `CleanWording`, `AfterClean`, `ProgressEvent`, `LockHolders` (`agentCleanup`, no `.clean` token so `cleanHasOneOwner` stays true), `SettingsFile` and `SharedSettings.keys`, fixtures, ContractTests counts.

**PR 2, core.** `report::recheck::scoped(env, opts, reach, progress, statfs, reader)` over `scan_inner` with the scope's reach. `plan.rs`: `AgentBound { project, checkouts }`, built from the walked report and the `Scope`: the main checkout (unless bare) and only the scope's checkouts that the walk found as a `git.worktree` finding at that exact path, none at or above the home folder; `contains` is strictly inside. `agent_may_remove(finding, action, bound)`: `safe` class and rule class, selected, not ignored, category `project-artifact`, `Action::Delete`, inside the bound, not `user.kept`; one function, called by the plan and again by the executor. `plan_agent(report, catalog, bound, listed, opt_in, user_skips) -> AgentPlan { plan, items, for_person }`: `build_plan`'s filters without `contain()` (a selected dead worktree must not swallow its safe items, since the agent never removes the worktree); the matching of a listed item (path, identity, rule, class, derived action) shared with `plan_listed_checked`, not copied. `exec`: `Consent::Agent { bound }`; `allowed` calls `agent_may_remove`; a refusal under it is `Skipped(NotConsented)` and journaled; **`WorktreeRecheck`'s `person` becomes `Interactive | Listed` only**, so `Agent` never waives conditions 1 and 3; journal lines tagged. `CleanReport::for_agent`.

**PR 3, command.** `AgentAction::Clean(AgentCleanArgs { root, items, dry_run, json, data_dir })`; gate `None` with a comment; `cmd/agent_clean.rs`: root refusal, the list parsed before any `Env`, `Scope::resolve`, settings loaded (an error is the `settings_error` report), the exclusive lock only for a real run with the opt-in, the settings read again under it, `scan_options` and `recheck::scoped`, `AgentBound`, `plan_agent`, the executor only on a real run with a non-empty plan, `CleanReport::for_agent`, exit codes as `clean_items.rs`. `render::agent_clean` and `render::person_command` (none when a path holds a control or bidi character, as `clean_command`; then the Review window). `agent status` reads `agents_clean_safe` (agents schema stays 1, the field defaulted). Registries: a cases entry for the refusal outside a repository and `uncovered` for the rest; `unported_cli`; a structure test (`cmd/agent_clean.rs` builds only `Consent::Agent`, calls `execute` once, never `write_last_scan`, `after_clean`, `build_plan` or a prompt); DDCKit ContractTests: no app source names `agent clean`. README: the section and the settings key with its risk and the disclaimer link.

**Safety.** One removal path: `Action::Delete` of a `safe` project artifact through `exec::execute`. Without the opt-in, with unreadable settings or `--dry-run`, `execute` is never called and nothing is locked or journaled. The bound is checked at the plan and again in the executor; the walk is pruned to the checkouts; the plan is built from a walk made under the exclusive lock; the guard's re-checks are unchanged; the agent's own process counts as using its worktree.

Code refs, proposed: `crates/ddc-cli/src/cmd/agent_clean.rs`, `crates/ddc-cli/src/cmd/agent_status.rs`, `crates/ddc-cli/src/{cli,gate,render,prompt}.rs`, `crates/ddc-core/src/plan.rs`, `crates/ddc-core/src/exec/{mod,journal,report}.rs`, `crates/ddc-core/src/report/recheck.rs`, `crates/ddc-core/src/{holders,refusal,progress,settings}.rs`, `crates/ddc-core/src/agents/model.rs`, `docs/{clean,progress,lock,error,settings,agents}.schema.json`, `fuzz/regressions/settings/`, DDCKit `CleanReport`, `CleanWording`, `AfterClean`, `ProgressEvent`, `LockHolders`, `SettingsFile`, `SharedSettings`, ContractTests, `README.md`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (`agent_clean_cli.rs`, temp home: project `proj` with `__pycache__` and `.pytest_cache` in main, an idle `.venv` with `uv.lock`, a dead Claude Code worktree holding a `__pycache__`, a dirty worktree; a second project `other` with `__pycache__`; the npm cache): without `agents_clean_safe`, `ddc agent clean ~/proj --json` exits 0, validates against `docs/clean.schema.json`, removes nothing, writes no journal and no `last-scan.json`, states `agents_clean_safe: false`, reports each safe item of proj's checkouts `agent-opt-in-off` and every other considered item `not-consented`; nothing of `other` or the cache is considered
- [ ] Test: the same with `agents_clean_safe: true`: exactly the safe items inside proj's checkouts are removed; both worktrees, the `.venv`, other's `__pycache__` and the npm cache stay; every journal line has `consent: agent` and `project` = proj; exit 0
- [ ] Test: with the opt-in on, `for_person` lists exactly the dead worktree and the `.venv`; with it off, also main's `__pycache__` and `.pytest_cache`; never anything inside the worktree; `ddc clean --items` accepts it and, passed to `ddc clean --items -`, removes the worktree through git (`git worktree list` no longer names it). The text output prints the same list in the heredoc command, its delimiter quoted
- [ ] Test: ROOT = the dead worktree resolves to proj and covers its sibling worktrees; a folder outside a repository exits 2 (`not-a-repository` under `--json`); a repository at the home folder exits 2 (`home-is-the-checkout`)
- [ ] Test: `--items` listing proj's `__pycache__`, the `.venv`, a review item the catalog deletes with `permanent: true` (the fixture's, picked at PR 3), other's `__pycache__` and the npm cache: only the first is removed, the rest are `not-consented`, in list order; a malformed list exits 2 and touches neither the tree nor the journal
- [ ] Test: `--dry-run` with the opt-in on removes nothing, runs while another exclusive holder holds the lock, writes no journal; what would go is `not-run` and `dry_run` is true. A real run while the lock is held exits 75 with the `busy` envelope, untouched; while one runs, `lock status` names `agent-clean`
- [ ] Test: a malformed `settings.json`: exit 0, nothing removed or considered, `agents_clean_safe: false`, `settings_error` set
- [ ] Test (core): under `Consent::Agent` the executor re-checks as for any consent (an item replaced after the walk is `changed-since-scan`, one a process works in is `in-use`, one under `.ddc-keep` is `kept`); a plan item `agent_may_remove` refuses (rebuildable, outside the bound, a safe cache of another category, a `native` action, a checkout itself) is skipped `not-consented` and journaled; `Consent::Agent` never waives worktree conditions 1 and 3
- [ ] Test (core): a checkout git lists that the walk does not recognise as this project's worktree, one at or above the home folder (a symlinked home included), or one whose parent another account owns, is outside the bound, and the run goes on
- [ ] Test: a path with `$`, a backquote, a single quote and a space survives the printed command: `sh` runs it against a stub `ddc` that records stdin, which equals `for_person`; a project outside the scan roots of `settings.json` gets a command naming the checkouts' parents; a list of more than 50 items, or a non-UTF-8 path, prints no command
- [ ] Test: an older-format refusal: a `settings.json` with an unknown key names the key and says to update ddc
- [ ] Test: `ddc agent status --json` reports `agents_clean_safe` false by default and true with the key
- [ ] Contract: clean 13, journal 13, progress 11, lock 2, error 5 and the settings key, each committed schema equal to the generated one; DDCKit decodes the new values and ContractTests pass, the app never running `agent clean` included; the settings fuzz target round-trips a file with the key
- [ ] `cases.json`, `structure.rs` (the new test), `unported_cli.rs`, `gate.rs`, `readme_cli.rs` green; the goldens unchanged
- [ ] `make gate-fast` passes; the Linux and Windows legs pass on the owner's machines, with their skips listed in `.config/native-skips/`
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): on a project with dead worktrees, `ddc agent clean --dry-run --json` lists them under `for_person`, and a real run with the opt-in removes only the safe items; no timings or project names in this vault

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Status log

- 2026-11-14T07:17:15.793Z status: planned → in-progress — work started (plan gate)

## Technical Notes

- ~~`--yes` today: `safe` by default, `rebuildable` only with `--class rebuildable`, `review` in `--class` refused (`cmd/clean.rs`, exit 2). `--agent` is that policy with `rebuildable` turned on and scoped by path, so `PlanOptions` gains a consent enum rather than a flag soup: `Interactive`, `Yes { classes }`, `Items { list }`, `Agent { root, items: Option<list> }`.~~ *Superseded 2026-11-14: `plan_agent` and `Consent::Agent`; `PlanOptions` is unchanged.*
- The amendment to §7 of the menu bar ADR matters here: `--items` without `--agent` keeps its meaning (a person's list); `--items` with `--agent` is a filter. One `--items` parser, two consent modes.
- ~~A `safe` global cache (`~/.cache/uv`) is removable under `--agent`…~~ *Superseded 2026-11-02 (ADR §7): shared caches are never removed by an agent; it may only propose them to the person.*
- The DDC-002 story's open question (full rescan vs scoped re-walk) decides this story's rescan too; `--agent` is called from inside sessions, so a 40 s rescan is the cost to measure.
- ~~The `not-consented` reason is new… so no `schema_version` bump~~ *Superseded 2026-11-02 (ADR §7): the clean report ships (schema 6, decoded by the app), so `not-consented` and `agent-opt-in-off` bump the clean report and the journal.*

- *2026-11-08, from story "ddc footprint: a scoped project report" (done):* reuse `footprint::Scope::resolve` and a pruned scan over `report::walk_classified` with `scope.reach()` (a new seam beside `recheck::pruned`), keeping `safe`, selected findings inside a checkout. Do not write `last-scan.json` from that scoped walk: its roots are the checkouts' parents, so `same_roots` is false and the app's home report would be replaced by a cut one. The clean assesses in `Scan` mode, so its own process counts as a user of the worktree it runs in (footprint leaves it out): a worktree the agent sits in is never removed, which is the safe side.

- *2026-11-14, plan gate (decisions beyond §7's text, for the ADR's dated note):* (1) the tree bound counts only checkouts the walk recognises as this project's `git.worktree` and none at or above the home folder, since a forged `.git/worktrees/*/gitdir` could otherwise make the home a "checkout"; and only category `project-artifact` with `Action::Delete`, so no shared cache and no tool's command; (2) with an unreadable `settings.json` it considers nothing (the skips are unknown) instead of listing every item; (3) the progress schema bumps too (skip reasons travel in item events); (4) the versions above supersede "schema 6" in the notes before; (5) `PlanOptions` gains no consent: `plan_agent` and `Consent::Agent` instead; (6) the person's command carries the list itself in a heredoc, the POSIX form; PowerShell is a follow-up.

## Dependencies

- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]] (done)
- [[epics/DDC-003/stories/story-ddc-footprint-scoped-project-report|ddc footprint: a scoped project report]] (done): its `Scope` and walk
- [[epics/DDC-003/stories/story-worktree-findings-and-provenance|Worktree findings and provenance]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 and §7

## Attachments

-

---

*Last updated: 2026-11-14*
