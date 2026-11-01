---
type: epic
id: "DDC-003"
title: "Agent footprint"
status: in-progress
priority: p2
created: 2026-10-18
updated: 2026-11-01
external_refs: {}
tags: [agents, worktrees, cli, rules]
code_refs: ["crates/ddc-core/src/agents/", "crates/ddc-core/src/rules/entries.rs", "crates/ddc-core/src/rules/catalog/macos.rs", "crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/src/report/contents.rs", "crates/ddc-core/src/report/recheck.rs", "crates/ddc-core/src/exec/", "crates/ddc-core/src/plan.rs", "app/Packages/DDCKit/"]
review_status: pending
reviewed_at: null
---

# DDC-003: Agent footprint

| Field | Value |
|---|---|
| **Status** | planned |
| **Priority** | p2 |
| **Created** | 2026-10-18 |
| **Updated** | 2026-10-18 |

---

## Goal

`ddc` cleans up after coding agents, and coding agents use `ddc` to see what their work leaves behind.

- **Dead worktrees go.** A worktree a harness created, unlocked, idle for two days, without submodules, clean down to its ignored files, and with no commits of its own, is found with the environments inside it, grouped under its project, and removed through `git worktree remove` without a person. Anything less is `review` with the reason named.
- **Harness data has a name.** What Claude Code, Codex and opencode keep under the home folder is a finding with a class and a hint, not "outside findings".
- **An agent sees its project in seconds.** `ddc footprint` lists the main checkout and every worktree with its state, its artifacts and what is reclaimable now.
- **An agent cleans within bounds.** `ddc clean --agent --project ROOT` removes what `--yes` would, plus the dead worktrees of that one project, and nothing a person would have to think about.
- **The guidance ships with the tool.** An agent that reads the repository's `agents/` folder knows when to run `footprint`, what `--agent` may do, and that worktree removal goes through `ddc`.

## Context

The research note [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] found ten worktrees of one project with a 1.1 GB `.venv` each. Agents turn that from an occasional pattern into a daily one: Claude Code creates a worktree per isolated session under `.claude/worktrees/`, each session installs dependencies into it, and the harness keeps every worktree of a non-interactive run, every named session's worktree the person kept, and every worktree with changes. Its own sweep removes subagent and background worktrees after 30 days, only when they hold no work.

The safety model of DDC-001 cannot act on this: a worktree is a directory with `.git` at its top level, which no rule claims and the executor never deletes, and its environments count as active for 14 days. The owner asked on 2026-10-18 for `ddc` to clean after agents and to give agents a view of their own footprint. The how is decided in the ADR [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]:

- worktrees are findings (`git.worktree`, `git.worktree-orphan`) with a provenance (`origin`) and a state (`worktree`), `review` by default, `rebuildable` under six conditions (known harness, unlocked and not in use, idle, no submodules, clean including ignored content, no own commits), each checked through a guarded git probe contract, removed with `git worktree remove` and nothing else;
- harness data are known-path rules, with Claude Code's auto memory never claimed, transcripts reported, not removed, and `paste-cache`/`usage-data` as `review` because they are history;
- `ddc footprint` is a scoped, read-only, fast report for one project;
- `ddc clean --agent` is bounded consent, a variant of `--yes`, never of `--items`;
- a skill and an `AGENTS.md` snippet ship in the code repository; an MCP server is deferred.

This ADR and epic were drafted without a critic pass, as a test of the vault as the handoff between an agent that plans and an agent that implements. The first critic pass (PR #1 review, 2026-10-18, seven findings reproduced on git 2.54) is answered in this revision; the ADR's §2 probe contract, ignored-content and submodule conditions, surviving-ref check and the removal of `prune`, and §3's `review` for `paste-cache`/`usage-data`, come from it.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-worktree-findings-and-provenance](./stories/story-worktree-findings-and-provenance.md) | planned | `git.worktree` and `git.worktree-orphan` rules; `origin`, `worktree` state and `Project.worktree_of`; the git probe contract; six eligibility conditions with `--worktree-min-age`; `contained` in the plan; `native` removal through `git worktree remove` alone |
| [story-harness-data-rules](./stories/story-harness-data-rules.md) | planned | Known-path rules for `~/.claude` (debug and legacy `safe`; `file-history`, `paste-cache`, `usage-data` `review`; transcripts report-only; `memory/` never claimed), `~/.codex`, `~/.cache/codex-runtimes`, opencode; paths confirmed on the reference machine |
| [story-ddc-footprint-scoped-project-report](./stories/story-ddc-footprint-scoped-project-report.md) | planned | `ddc footprint [ROOT] --json`: main checkout and worktrees with state, artifacts and totals; `docs/footprint.schema.json`; read-only, no lock, under 5 s for ten worktrees |
| [story-clean-agent-bounded-consent](./stories/story-clean-agent-bounded-consent.md) | planned | `ddc clean --agent --project ROOT [--items -] --json`: `--yes` semantics plus the project's dead worktrees; `not-consented` skips; refusals; `consent: agent` in the journal; needs DDC-002 "CLI: clean an explicit item list" |
| [story-agent-guidance-skill-and-agents-snippet](./stories/story-agent-guidance-skill-and-agents-snippet.md) | planned | `agents/` in the code repository: a Claude Code skill and an `AGENTS.md` snippet that cite the JSON contracts; README sections for `footprint` and `--agent` |
| [story-per-session-footprint-transcripts-temporary-folders-and-worktrees](./stories/story-per-session-footprint-transcripts-temporary-folders-and-worktrees.md) | in-progress | `ddc disk agents` per session: transcript, checkpoints, temporary folder and worktrees, live or dead, sorted by size, in text and JSON; the report half by the DEV deadline (DDC-009) |
| [story-agent-skill-v0-ask-ddc-before-guessing-about-the-disk](./stories/story-agent-skill-v0-ask-ddc-before-guessing-about-the-disk.md) | planned | A read-only Claude Code skill and `AGENTS.md` snippet on today's commands (`disk agents`, `scan`, `rules`); agents propose, people clean; replaced later by the full guidance story |
| [story-coding-agents-temporary-folders-outside-the-home](./stories/story-coding-agents-temporary-folders-outside-the-home.md) | planned | The cleanup half: an `agent.claude-code-temp` rule for dead sessions' folders under `/private/tmp/claude-<uid>` (macOS) or `<tmpdir>/claude-<uid>` (Linux), by amendments of the agent and safety ADRs; after the DEV deadline |

## Expected Results

- [ ] On the reference machine, a project with several Claude Code worktrees under `.claude/worktrees/`: `ddc scan` lists each as a `git.worktree` finding grouped under the project, with its `.venv` or `node_modules` under it, and selects by default exactly those that pass all six conditions; the others name the condition that held them
- [ ] `ddc clean --yes --class rebuildable` on that project removes a selected worktree through `git worktree remove` alone, `git worktree list` no longer shows it, `.git/worktrees/<name>` is gone, the branch still exists, and no other worktree's metadata changed
- [ ] A dirty worktree, a locked worktree, a worktree with one unmerged commit (named or detached), a clean worktree with a unique ignored `.env`, a clean worktree with an initialized submodule, and a worktree created by hand with `git worktree add` are `review` and never in a default plan; a scan of them runs no repository-configured git helper and leaves their index untouched
- [ ] `ddc scan ~` reports `~/.claude`, `~/.codex` and opencode data as findings; `~/.claude/projects/<project>/memory/` is inside no finding; `paste-cache` and `usage-data` are `review`, never in a `--yes` plan
- [ ] `ddc footprint` in a worktree of that project returns in under 5 s and names the main checkout, every worktree with clean/dirty, locked, in-use and own-commits state, and the reclaimable totals now and after sessions end
- [ ] `ddc clean --agent --project <root> --items -` with a list that names a `review` item, a `rebuildable` item outside the project and a dead worktree inside it removes the worktree, skips the other two as `not-consented`, and journals `consent: agent`
- [ ] A Claude Code session that loads the shipped skill runs `ddc footprint` before installing dependencies into a new worktree (checked by reading the session transcript once, on the reference machine)

## Dependencies

- [[epics/DDC-001/epic|DDC-001: Disk space cleaner CLI for developers]]: the rule catalog, the executor and the native-command location check
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]: `--items`, `--json` and the exit codes that "Clean: `--agent` bounded consent" builds on
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]

## Open Questions

- [ ] The marker Claude Code writes into its worktrees' git metadata, and whether Codex and opencode mark theirs (decided at the plan gate of "Worktree findings and provenance")
- [ ] Whether `--worktree-min-age` stays at 2 days after the first weeks on the reference machine
- [ ] Whether `claude purge` runs non-interactively per project, which would turn transcripts from report-only into `native-only`
- [ ] Whether Codex and opencode session data can be mapped to a project for `footprint`, or only totalled

## Related

- Claude Code documentation: worktrees (https://code.claude.com/docs/en/worktrees), the `.claude` directory (https://code.claude.com/docs/en/claude-directory)
- ProjectStore, the sibling project whose `agents/` and MCP layout the guidance story follows

---

*Last updated: 2026-10-18*
