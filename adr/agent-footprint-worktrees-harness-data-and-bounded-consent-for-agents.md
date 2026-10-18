---
type: adr
id: "agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents"
title: "Agent footprint: worktrees, harness data and bounded consent for agents"
status: proposed
date: 2026-10-18
authors: ["Evgenii Konev"]
tags: [agents, worktrees, safety, cli, contract]
external_refs: {}
supersedes: null
superseded_by: null
review_status: pending
reviewed_at: null
---

# Agent footprint: worktrees, harness data and bounded consent for agents

| Field | Value |
|---|---|
| **Status** | proposed |
| **Date** | 2026-10-18 |
| **Authors** | Evgenii Konev |

---

## Context

Coding agents (Claude Code, Codex, opencode) now run many sessions a day on a developer's machine, and each session leaves something on the disk. The owner asked on 2026-10-18 for two things: agents should be able to use `ddc` to see what their own work leaves behind (checkouts, environments, worktrees, caches), and `ddc` should clean up after them, because that residue accumulates quietly and eats the disk.

What the residue is, and what is known about it:

1. **Worktrees and the environments inside them.** The research note [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] already saw the pattern before agents multiplied it: ten git worktrees of one project, each with its own `.venv` of about 1.1 GB, eleven gigabytes spent on copies of one environment. Claude Code creates a worktree per isolated session under `.claude/worktrees/<name>/` at the repository root, on a branch `worktree-<name>`; subagents with `isolation: worktree` get one each. Every such checkout is fresh, so the first thing a session does is install dependencies into it. Claude Code removes the worktree of an *unnamed, clean, interactive* session when the session exits. It keeps the rest: a named session's worktree (the person is asked and may keep it), a worktree with changes or new commits, every worktree of a non-interactive `-p` run (no exit prompt, so no cleanup), and a subagent worktree that holds changes. A periodic sweep removes subagent and background-session worktrees older than `cleanupPeriodDays` (30 days by default) only when they hold no work; it never touches a `--worktree` session that was not backgrounded, whatever its age, and never a worktree created by hand with `git worktree add`. While a session runs, Claude Code holds a `git worktree lock` on its worktree, and writes a marker into the worktree's git metadata so the sweep can tell its own worktrees from the person's.
2. **The safety model does not fit this residue.** [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]] selects a `rebuildable` artifact by default only when its project has been idle for `--min-age` days (14). An agent's worktree is hours old when it is already dead: the session ended, the branch was merged or abandoned, nothing will ever touch it again. By the activity rule it is an active project, and its `.venv` stays out of every plan for two weeks. A `git.worktree` finding does not exist at all: a directory with `.git` at its top level is never claimed by a rule, and the executor refuses to delete one. That refusal is right for a checkout a person works in; it is also why `ddc` cannot remove a worktree today.
3. **Harness data.** Claude Code writes what passes through a session under `~/.claude`: transcripts (`projects/<project>/<session>.jsonl`), spilled tool results and subagent transcripts (`projects/<project>/<session>/`), pre-edit snapshots (`file-history/`), `paste-cache/`, `debug/`, `shell-snapshots/`, `usage-data/`, and legacy `todos/`, `statsig/`, `logs/`. Its own sweep deletes most of these after `cleanupPeriodDays`, and `claude purge` removes a project's data on request; `projects/<project>/memory/` is auto memory and is never swept. Codex keeps sessions and logs under `~/.codex`, and the research note measured `~/.cache/codex-runtimes` at 1.6 GB on the reference machine. opencode keeps its data under `~/.local/share/opencode`. None of this is in the rule catalog, so a scan counts it as "outside findings".
4. **Agents work on a disk they cannot see.** When `uv sync` fails with `ENOSPC` inside a worktree, the agent has no way to tell what is free to remove and what belongs to the person. `ddc scan ~` takes about 40 s and answers a different question (the whole home folder); the agent needs its own project's worktrees, their environments and their state, in seconds.
5. **Consent.** The only non-interactive cleanup today is `--yes`: `safe` items, plus `rebuildable` only with an explicit `--class rebuildable`, and only items selected by default. [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §7 adds `clean --items`, an explicit list that counts as interactive consent **because a person confirmed it**. An agent is not a person. A list an agent writes cannot carry a person's consent, and treating it as one would let a prompt-injected or simply mistaken agent move `review` items to the Trash.
6. **Isolation.** Inside a worktree, Claude Code blocks commands whose working directory is the main checkout and git commands redirected into it. `ddc` runs with absolute paths and its own data directory, so it is not affected; but a `git worktree remove` that `ddc` runs on the agent's behalf acts on the shared `.git`, which is the point of removing a worktree, and it must be `ddc`, not the agent's shell, that runs it.

## Decision

### 1. Agent residue is a provenance, not a new risk class

The four classes stay. A finding gains an optional `origin` field: `{harness: "claude-code" | "codex" | "opencode" | "git", kind: "worktree" | "session-data" | "runtime-cache"}`, set when the rule recognizes a harness's own structure (item 2 and item 3). A `git.worktree` finding also gains an optional `worktree` field that carries its state (item 2): `{branch, detached, clean, ignored_content, own_commits, submodules, locked, in_use, idle_days, probe}`. The report, the plan and the app group by `origin`; the classes and the methods decide what happens, as before.

Both are new optional fields, so `schema_version` stays 1 under the versioning policy of [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]. The `notes` enum is **not** extended: that policy names `notes` among the enums whose new values bump the version, so every reason a worktree is not selected lives in `worktree`, never in `notes`. A reader of version 1 that ignores unknown fields keeps working, and a fixture test proves it.

### 2. Worktrees are findings: `git.worktree` and `git.worktree-orphan`

**Recognition.** A linked worktree is a directory whose `.git` is a **file** with a `gitdir:` line pointing into `<main>/.git/worktrees/<name>`, and that gitdir exists. `report::project::git_dir` already resolves this. The main checkout is `<main>`, read from `<gitdir>/commondir`. A directory whose `.git` file points to a gitdir that does not exist is `git.worktree-orphan`: the main repository is gone, and the directory is a plain copy of files.

The rule fires for every linked worktree inside the scan roots, not only for agents' worktrees; `origin` says which is which. `origin.harness` is `claude-code` when the worktree lies under `<main>/.claude/worktrees/` **or** its gitdir carries the marker Claude Code writes (the marker's exact form is confirmed at the plan gate of the story "Worktree findings and provenance"; until then the path is the signal). Other harnesses are added the same way when their layout is known. A worktree with neither is `origin.harness: git`, the person's own.

**A worktree is also a region.** Rules keep working inside it, as inside `~/Downloads`: its `.venv` is a `python.venv` finding whose `project.root` is the worktree, with the activity signals `report::project::last_activity` already reads through `gitdir:`. `Project` gains an optional `worktree_of: <main root>`, so the report groups a worktree's artifacts under the main project. In a plan, an item that lies inside a selected `git.worktree` item is dropped with the reason `contained`: removing the worktree removes it.

**The git probe contract.** Every git command `ddc` runs — at scan, in `footprint`, in the executor's re-check and for the removal itself — is a probe under one contract, because an ordinary `git status` is not a read: it runs the helper named by `core.fsmonitor`, and its background refresh rewrites the index, whose mtime is the activity signal of condition 5. The contract:

- the program is `git` found on `PATH` through the `CommandRunner`, with `--no-optional-locks` and `GIT_OPTIONAL_LOCKS=0`, so no probe writes the index or any other optional lock;
- every repository- or user-configured helper that a probe could run is switched off on the command line, which outranks every config file: `-c core.fsmonitor=false -c core.untrackedCache=false -c core.hooksPath=/dev/null -c status.submoduleSummary=false -c core.pager=cat -c core.askPass=` and `GIT_TERMINAL_PROMPT=0`; the exact list is a constant with a test per entry, and the plan gate checks it against `git-config(1)` for the pinned minimum git version;
- a probe that exits non-zero, times out (30 s), or prints something the parser does not expect **fails closed**: the worktree is `review` with `probe: unknown`, and the executor skips it. `ddc` never retries a failed probe with fewer restrictions;
- two consecutive scans of an untouched worktree give the same state and the same eligibility, and leave the index bytes and mtime unchanged; a test asserts this and a fixture with a `core.fsmonitor` helper asserts the helper was never executed.

**Class.** `git.worktree` is `review` by default. It is `rebuildable` only when all of the following hold, checked at classification and again by the executor's re-check, in this order, stopping at the first failure, which is recorded in `worktree`:

1. **Known harness.** `origin.harness` is a known harness, not `git`. The person's own worktrees are never selected by default: a person who made one by hand decides about it by hand.
2. **Unlocked and not in use.** `<gitdir>/locked` is absent, and no process of the current user has its working directory, executable or an argument inside the worktree (the `in-use` check of the safety ADR §5). A lock is a lock whoever set it: `ddc` cannot tell a `git worktree lock` a person set from one a killed session left, so a locked worktree is `review` with `locked: true` and the hint `git worktree unlock`. Claude Code's own sweep releases the locks of its sessions whose process has exited, after which `ddc` sees the worktree unlocked.
3. **Idle.** The worktree's own activity (`<gitdir>/index`, `<gitdir>/HEAD`, and the manifests and lock files in the worktree, the signals of safety ADR §4 applied to the worktree) is older than `--worktree-min-age`, **2 days** by default. This is a separate knob from `--min-age` (14 days), because an agent's worktree is dead within hours and a person's abandoned project within weeks. Two days leaves a weekend for "I kept it to come back on Monday".
4. **No submodules.** No submodule is initialized in the worktree (`git submodule status` reports none, or none with a checkout; equivalently no `<worktree>/<path>/.git` for a path in `.gitmodules`). `git worktree remove` refuses a worktree with an initialized submodule even when it is clean, and `ddc` never passes `--force`; promising a cleanup the executor cannot perform is wrong, so such a worktree is `review` with `submodules: true`. Its inner findings are still cleaned on their own.
5. **Clean, including ignored content.** `git status --porcelain=v1 --untracked-files=all --ignored=matching` under the probe contract prints no tracked change and no untracked file (`dirty` otherwise). Ignored entries are the point of a worktree cleanup, and the trap: `status` does not list them by default, and `git worktree remove` without `--force` deletes them without a word, which is how a `.env` or a local data directory would be lost. So every ignored path must be accounted for before the worktree is eligible:
   - it lies inside a finding the rule visitor claimed in the worktree whose class is `safe` or `rebuildable` with method `delete` — an artifact `ddc` would remove on its own; **or**
   - it is byte-identical to the file at the same relative path in the main checkout, which is what `.worktreeinclude` produces (`.env`, `.env.local` copied into every new worktree); a copy carries no unique content, an edited one does.

   Any other ignored path — a unique file, an ignored directory no rule claims, or a nested artifact whose own rule gives `review` or `user-data` (a venv without a lock file, an ignored `node_modules` without a lock file) — makes the worktree `review` with `ignored_content: [paths]` (at most 20, then a count). A person sees what is in it and decides; the inner artifacts that are eligible on their own are still cleaned.
6. **No commits of its own.** The set of references that survive the removal is computed explicitly: `refs/heads/*` except the candidate branch, `refs/remotes/*`, `refs/tags/*`, and the `HEAD` commit of every other worktree from `git worktree list --porcelain`. The candidate's `HEAD` commit, taken from `git rev-parse` in the worktree, must be reachable from that set: `git rev-list --count <head-commit> --not <survivors…>` is 0. Neither `--all` nor `--branches` is used: `--all` includes `HEAD`, and traversal includes other worktrees' `HEAD`s, which is how the sketch in the first draft counted a unique commit as preserved. A detached `HEAD` is handled by the same rule, and matters more: when the worktree goes, nothing else holds its commit. A merged branch whose remote was deleted passes; a branch or a detached `HEAD` with one unique commit does not (`own_commits: n`). The check gives the same answer run from the main checkout and from a linked worktree, because it names commits, not `HEAD`.

The probes of conditions 4, 5 and 6 run only for candidates that passed 1 to 3 first, in parallel across candidates, so a scan does not run git across every worktree on the machine.

**Removal.** The method is `native`: `git -C <main> worktree remove <path>` under the probe contract, run from the main checkout with the 10-minute timeout of every native command. It removes the worktree's own administrative directory `<main>/.git/worktrees/<name>` by itself; `ddc` runs **no** `git worktree prune`, because `prune` acts on every worktree whose directory is missing at the moment — one on an unmounted volume, one moved by hand — and would drop their `HEAD`, index and reflog without their being the consented finding. The location check of safety ADR §6 applies: the command runs only if `git -C <main> worktree list --porcelain` lists the finding's path; otherwise the fallback is `none` (report only). `git worktree remove` without `--force` refuses a dirty or locked worktree, a third net under conditions 2 and 5, but not a net for ignored content, which it deletes; `ddc` never passes `--force`. `delete` and `trash` are never used for a `git.worktree` finding: the executor's rule that a directory with `.git` at its top level is not deleted stays, and a test asserts the method.

`ddc` deletes no branch. The branch `worktree-<name>` is a few bytes and a person's decision; the hint says `git branch -d <branch>`.

`git.worktree-orphan` is `review`, method `trash`, `--permanent` gives `delete`. Rules work inside it too.

### 3. Harness data: known-path rules with a protected memory

`~/.claude` is a harness region. Rules inside it:

| id | Paths (from `~`) | Class | Method | Note |
|---|---|---|---|---|
| `agent.claude-code-file-history` | `.claude/file-history` | `review` | `trash` | checkpoint restore of past sessions; swept by Claude Code after `cleanupPeriodDays` |
| `agent.claude-code-scratch` | `.claude/debug`, `.claude/shell-snapshots`, `.claude/todos`, `.claude/statsig`, `.claude/logs` | `safe` | `delete` | owner: `claude` processes; `shell-snapshots` are removed on clean exit and left by crashes; the other four are debug logs and legacy directories the harness no longer writes |
| `agent.claude-code-history` | `.claude/paste-cache`, `.claude/usage-data` | `review` | `trash` | not reproducible: `paste-cache` holds the pasted text of recalled prompts, `usage-data` the past `/insights` reports and their cached analysis; the hint names the loss |
| `agent.claude-code-transcripts` | `.claude/projects` (region) | `review` | `none` | report only with the hint: lower `cleanupPeriodDays`, or `claude purge` per project. `projects/<project>/memory/` is **never** claimed, at any depth: it is auto memory, the one thing under `~/.claude` that is not reproducible |
| `agent.codex-sessions` | `.codex/sessions`, `.codex/log` | `review` | `trash` | paths confirmed at the plan gate on the reference machine |
| `agent.codex-runtimes` | `.cache/codex-runtimes` | `review` | `trash` | downloaded runtimes, restored by Codex on demand |
| `agent.opencode-data` | `.local/share/opencode`, `.cache/opencode` | `review` | `trash` | paths confirmed at the plan gate |

Transcripts are reported, not removed, because the harness already has a retention sweep and a purge command for them, and a transcript is what `--resume` and `/rewind` read: a person decides its lifetime through `cleanupPeriodDays`, not through a cleaner. `paste-cache` and `usage-data` are history too — a recalled prompt loses its pasted text, a past `/insights` report is gone — so they are `review`, a per-item decision with the loss named, never inside `--yes` or `--agent`. Only the debug logs, the crash leftovers and the legacy directories are `safe`. Session scratchpads under the system temp directory (`CLAUDE_CODE_TMPDIR`) lie outside `$HOME` and outside the default scan roots; the OS clears them, and `ddc` does not look for them.

### 4. `ddc footprint`: a scoped report for one project, in seconds

`ddc footprint [ROOT] [--json]` answers an agent's question about one project, not the home folder:

- `ROOT` defaults to the current directory; the main checkout is found through `gitdir:` and `commondir` when `ROOT` is a worktree.
- It lists the main checkout and every linked worktree from `git worktree list --porcelain`, plus every directory under `<main>/.claude/worktrees/` that git does not list (a worktree removed from git but left on disk is a `git.worktree-orphan`).
- For each: path, `origin`, and the `worktree` state of item 2 (branch or detached, clean or dirty, ignored content, own commits, submodules, locked, in use, idle days, probe result), obtained under the same probe contract, and the findings inside it with their class and size, found by walking only those directories with the ordinary rule visitor.
- Totals: what is reclaimable now (selected by default under this ADR's conditions), what would be reclaimable once the running sessions end (worktrees that fail only condition 2 or 3), and the harness regions of item 3 from `last-scan.json` when a report exists, with its age.
- `--json` is a versioned schema, `docs/footprint.schema.json`, generated and compared like `docs/scan.schema.json`.

It is read-only. It does not write `last-scan.json`, does not take the scan lock, and does not notify. On the reference machine it must finish within 5 s for a project with ten worktrees; a worktree's `git status` is the slow part and runs in parallel across worktrees.

### 5. `ddc clean --agent`: bounded consent

This **amends** §5 of the safety ADR and §7 of the menu bar ADR, and both record it as a dated amendment.

**What an agent may do.** `ddc clean --agent --project ROOT [--items <file|->] [--json]` is the only cleanup entry point for an agent. `--agent` means: the caller is not a person, and nothing in the call is interactive consent. It is bounded like `--yes`, not like `--items`:

- it runs without a terminal and never prompts;
- it removes `safe` items, and `rebuildable` items that are **selected by default** and lie inside `ROOT`'s main checkout or one of its worktrees. For `git.worktree` items, "selected by default" means the conditions of item 2;
- it never removes a `review` item, never runs a `native-only` command, and never removes an item outside `ROOT`'s tree except `safe` global caches, the same set `--yes` already allows a script;
- with `--items`, the list is a **filter**, not a selection: an item that is not selected by default under the rules above is skipped as `not-consented`, however it is listed; a `review` item in the list is skipped, not refused, so an agent that lists the whole footprint gets everything it is allowed to and a reason for the rest. `permanent` on an item is ignored under `--agent` and noted;
- `--class`, `--permanent` and `--yes` together with `--agent` are refused with exit code 2;
- every action is journaled with `consent: agent` and the `ROOT`, next to the fields of safety ADR §7. The existing re-checks, the tombstone, the `in-use` check and the refusals apply unchanged.

**What this changes.** Until now the only non-interactive channel that removed `rebuildable` items without a 14-day idle project was none. With `--agent`, a script can remove a clean, unlocked, two-day-idle harness worktree and the environments inside it, and a running agent can remove its own dead worktrees and the `safe` caches, without a person present. It is bounded by the class rule (nothing a person would have to think about), by the tree (the agent's own project), by the executor's re-checks, and by the journal.

§8 of the safety ADR is unchanged: the background scan still never calls the executor. A scheduled scan finds worktrees and harness data like anything else, and the notification threshold counts them.

### 6. Guidance ships with the tool, the server comes later

The code repository gets an `agents/` folder with the guidance an agent needs to use `ddc` well: a Claude Code skill and an `AGENTS.md` snippet that say when to run `ddc footprint` (at the start of a session in a worktree, before installing dependencies into a new one, on `ENOSPC`, at the end of a session), what `ddc clean --agent` may and may not do, and that worktree removal goes through `ddc`, not through the agent's own `git worktree remove`. The guidance is text that cites the JSON contracts; it decides nothing on its own.

An MCP server over the same contracts is deferred (Alternative C): the CLI's `--json` is the interface, and a server wraps it once a second harness asks for it.

## Rationale

1. **Provenance, not a fifth class.** The classes describe how costly a loss is, and that does not change because an agent made the artifact. What changes is the evidence `ddc` can read: a harness's worktree carries its layout, its lock and its git state, which is more than any `node_modules` offers. `origin` records the evidence; the class stays the judgment.
2. **The clean-worktree conditions are the ones the harness itself applies before removing a worktree**: no changed or untracked files, no commits of its own, not locked. `ddc` adds a short idle window and its own `in-use` check. A person's own worktrees are left to the person because `ddc` cannot tell "kept for later" from "forgotten".
3. **`native` through git, never `delete`.** Removing a worktree is a git operation on the shared `.git`: `git worktree remove` updates the metadata and refuses a dirty tree on its own. Deleting the directory would leave a stale entry in `.git/worktrees` and would mean giving up the executor's "no directory with `.git` at its top level" rule, which protects every checkout the person has. The removal is scoped to its own entry; repository-wide maintenance (`prune`) is not a cleanup of one finding.
4. **Ignored content is accounted for, not assumed.** The one thing git's own refusal does not protect is exactly what a worktree cleanup is about. The two admissible cases, an artifact `ddc` would remove anyway and a verbatim copy of the main checkout's file, are checkable; everything else is a person's call.
5. **Probes under a contract.** A git command is a program the repository can configure; a scanner that runs it unguarded runs repository-chosen code and rewrites the evidence it then reads. One guarded invocation, fail-closed, is cheaper than reasoning about each.
6. **Transcripts and history are the harness's to retire.** They have a retention setting and a purge command, and they hold the resume state; a cleaner that deletes them competes with the harness's own policy. Reporting their size and the knob is what helps.
7. **Agent consent is `--yes` with a tree, not `--items` with a signature.** `--yes` was designed for a caller that cannot be asked; the agent is that caller. Treating an agent's list as a person's confirmation would hand the `review` class to whoever can influence the agent's prompt.
8. **A scoped report because the question is scoped.** The agent asks about its project; a 40-second home scan under a lock is the wrong tool for that, and running it from every session would make the lock contention of the safety ADR §5 a daily event.

## Alternatives Considered

### Alternative A: Treat agent worktrees as ordinary projects and shorten `--min-age`

**Pros**:
- No new rule; one knob.

**Cons**:
- `--min-age` is global: two days would select every idle person's project environment after a weekend.
- The worktree directory itself still could not be removed (the `.git` refusal), so the checkout and its branch would stay while its `.venv` goes.

**Rejected because**: it changes the default for every project to fix one kind of residue, and removes the environment without the checkout.

### Alternative B: Let `--items` carry agent consent as it carries a person's

**Pros**:
- Nothing new in the CLI; the app and the agent use one channel.

**Cons**:
- An agent's list could name `review` items; the executor would move them to the Trash on an agent's say-so. A prompt injection in a repository file would then reach the Trash of the person's models and toolchains.

**Rejected because**: §7 of the menu bar ADR rests on a person having looked at the sheet. No one looks at an agent's list.

### Alternative C: An MCP server first

**Pros**:
- Harness-neutral; tool descriptions double as guidance; the pattern ProjectStore itself uses.

**Cons**:
- A second interface to keep stable before the first one (the JSON contracts) exists.
- Each harness still needs its own registration and guidance file.

**Rejected for now**: the CLI contracts come first; a server over them is a later story once the contracts have been used from two harnesses.

### Alternative D: Delete harness worktrees with `delete`, like any `rebuildable` directory

**Pros**:
- No git invocation; the tombstone and the journal cover it.

**Cons**:
- `.git/worktrees/<name>` stays behind until someone runs `git worktree prune`; the branch stays checked out in git's eyes.
- Requires an exception to the executor's rule that protects directories with `.git` at their top level, for every rule, forever.

**Rejected because**: git owns the metadata; the exception would weaken the protection of every checkout.

### Alternative E: Remove transcripts and `file-history` as `safe`

**Pros**:
- The largest part of `~/.claude` on a busy machine.

**Cons**:
- `--resume`, `--continue`, `/rewind` and the session picker read them; a cleaner that removes them silently breaks a workflow the person did not know depended on them.
- Claude Code already deletes them after `cleanupPeriodDays`.

**Rejected because**: the harness has the retention policy; `ddc` reports the size and names the setting.

## Consequences

**Positive**:
- Dead agent worktrees and the environments inside them are found, grouped under their project, and can go without a person, within two days instead of two weeks.
- An agent sees its own project's footprint in seconds and can act on it through one bounded command.
- Harness data stops being "outside findings": its size has a name and a knob.
- The safety model is unchanged where it matters: `review` stays a person's choice, the executor's re-checks apply to every `--agent` action, and worktree removal goes through git.

**Negative / trade-offs**:
- A scan runs git probes on candidate worktrees; on a worktree with a huge tree `status --ignored` and the content comparison can take seconds, bounded by the 30 s timeout and `probe: unknown`, which keeps the worktree `review`.
- A harness worktree with any unique ignored file, an initialized submodule or a detached `HEAD` on a unique commit is never selected by default; the report says why, and the inner artifacts are still cleaned. Fewer automatic removals than the first draft promised; none that lose data.
- A person's named worktree, kept clean on purpose and untouched for two days, is `rebuildable` and selected by default if it was created by a harness; `.ddc-keep` in it, or `--worktree-min-age`, keeps it. The README says so.
- `--agent` lets a script without a person remove `rebuildable` items of one project tree. That is the cost of cleaning after agents; the journal records every such action with `consent: agent`.
- Two new CLI contracts with a `schema_version`: the footprint report and the `--agent` clean report (a variant of the DDC-002 clean report).
- Three harnesses' private layouts are now in the catalog and can change under `ddc`; the plan gate confirms paths on the reference machine, and a rule whose path is gone simply finds nothing.

**What changes in code / process**:
- `ddc-core`: `origin`, `worktree` and `Project.worktree_of` in the model; the `git.worktree` and `git.worktree-orphan` rules and their matcher in `rules/`; the six conditions in `rules/classify.rs` with the git probes behind one guarded invocation in a new `git` module (the probe contract, the ignored-content accounting, the surviving-ref set); `contained` in `plan.rs`; the `native` worktree removal with the location check in `exec/native.rs`, no `prune`; the harness known-path rules in `rules/catalog.rs` with the `memory/` protection in the visitor.
- `ddc-cli`: `footprint` command and `docs/footprint.schema.json`; `clean --agent --project` with its refusals, the `not-consented` skip reason and `consent` in the journal; `--worktree-min-age`; render groups by worktree.
- Repository: `agents/` with the skill and the `AGENTS.md` snippet; README sections for `footprint` and `--agent`.
- Safety ADR §5 and menu bar ADR §7: dated amendments pointing here.
- Depends on the DDC-002 stories "CLI: clean an explicit item list" (`--items`, `--json`, exit codes) for the `--agent` entry point; the worktree rules and `footprint` do not.

## Open Questions

- The exact marker Claude Code writes into a worktree's git metadata, and whether Codex and opencode mark theirs. Until confirmed, `<main>/.claude/worktrees/` is the only Claude Code signal, and the other two harnesses have no worktree signal.
- The minimum git version the probe contract is pinned to, and whether `--ignored=matching` and `--porcelain=v1` behave the same on the oldest version Homebrew ships on macOS 26.
- Whether `--worktree-min-age` should default to 2 days or to the harness's own idea of "old" (`cleanupPeriodDays` is 30). Two days is the proposal; the first weeks of use on the reference machine decide.
- Whether `claude purge` can be run non-interactively and per project, which would make `agent.claude-code-transcripts` a `native-only` rule instead of report-only.
- Whether `footprint` should also read `~/.codex` and `~/.local/share/opencode` session data per project, or only report their totals; the per-project mapping of those harnesses is not known.

## References

- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]: the ten-worktree case and `~/.cache/codex-runtimes`
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §3 selection conditions, §4 activity, §5 consent and re-checks, §6 native commands, §8 background mode
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 `clean --items`, `--json`, exit codes; §7 consent
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]: schema versioning policy
- [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]]: rule ids, matchers, the `.git` and region rules
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]: the `--items` and `--json` contracts `--agent` builds on
- Claude Code documentation: Run parallel sessions with worktrees (https://code.claude.com/docs/en/worktrees); Explore the .claude directory, "Application data" and "Cleaned up automatically" (https://code.claude.com/docs/en/claude-directory)
- `git-worktree(1)`: `remove`, `lock`, `list --porcelain`; `git-status(1)`: background refresh, `--ignored`; `git-config(1)`: `core.fsmonitor`, `core.untrackedCache`; `git-rev-list(1)`: `--all` includes `HEAD`
- PR #1 review (2026-10-18): the seven findings this revision answers (ignored content, probe side effects, history under `safe`, repository-wide `prune`, `--all` in the own-commit check, `notes` versioning, clean initialized submodules)

---

*Last updated: 2026-10-18*
