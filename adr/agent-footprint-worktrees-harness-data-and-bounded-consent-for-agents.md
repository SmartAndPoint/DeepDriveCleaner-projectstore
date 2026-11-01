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

Coding agents (Claude Code, Codex, opencode, Cursor) now run many sessions a day on a developer's machine, and each session leaves something on the disk. The owner asked on 2026-10-18 for two things: agents should be able to use `ddc` to see what their own work leaves behind (checkouts, environments, worktrees, caches), and `ddc` should clean up after them, because that residue accumulates quietly and eats the disk.

What the residue is, and what is known about it:

1. **Worktrees and the environments inside them.** The research note [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] already saw the pattern before agents multiplied it: ten git worktrees of one project, each with its own `.venv` of about 1.1 GB, eleven gigabytes spent on copies of one environment. Claude Code creates a worktree per isolated session under `.claude/worktrees/<name>/` at the repository root, on a branch `worktree-<name>`; subagents with `isolation: worktree` get one each. Every such checkout is fresh, so the first thing a session does is install dependencies into it. Claude Code removes the worktree of an *unnamed, clean, interactive* session when the session exits. It keeps the rest: a named session's worktree (the person is asked and may keep it), a worktree with changes or new commits, every worktree of a non-interactive `-p` run (no exit prompt, so no cleanup), and a subagent worktree that holds changes. A periodic sweep removes subagent and background-session worktrees older than `cleanupPeriodDays` (30 days by default) only when they hold no work; it never touches a `--worktree` session that was not backgrounded, whatever its age, and never a worktree created by hand with `git worktree add`. While a session runs, Claude Code holds a `git worktree lock` on its worktree, and writes a marker into the worktree's git metadata so the sweep can tell its own worktrees from the person's.
2. **The safety model does not fit this residue.** [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]] selects a `rebuildable` artifact by default only when its project has been idle for `--min-age` days (14). An agent's worktree is hours old when it is already dead: the session ended, the branch was merged or abandoned, nothing will ever touch it again. By the activity rule it is an active project, and its `.venv` stays out of every plan for two weeks. A `git.worktree` finding does not exist at all: a directory with `.git` at its top level is never claimed by a rule, and the executor refuses to delete one. That refusal is right for a checkout a person works in; it is also why `ddc` cannot remove a worktree today.
3. **Harness data.** Claude Code writes what passes through a session under `~/.claude`: transcripts (`projects/<project>/<session>.jsonl`), spilled tool results and subagent transcripts (`projects/<project>/<session>/`), pre-edit snapshots (`file-history/`), `paste-cache/`, `debug/`, `shell-snapshots/`, `usage-data/`, and legacy `todos/`, `statsig/`, `logs/`. Its own sweep deletes most of these after `cleanupPeriodDays`, and `claude purge` removes a project's data on request; `projects/<project>/memory/` is auto memory and is never swept. Codex keeps sessions and logs under `~/.codex`, and the research note measured `~/.cache/codex-runtimes` at 1.6 GB on the reference machine. opencode keeps its data under `~/.local/share/opencode`, its provider credentials (`auth.json`) among it. Cursor's agent keeps chats and tracking under `~/.cursor`. None of this is in the rule catalog, so a scan counts it as "outside findings".
4. **Agents work on a disk they cannot see.** When `uv sync` fails with `ENOSPC` inside a worktree, the agent has no way to tell what is free to remove and what belongs to the person. `ddc scan ~` takes about 40 s and answers a different question (the whole home folder); the agent needs its own project's worktrees, their environments and their state, in seconds.
5. **Consent.** The only non-interactive cleanup today is `--yes`: `safe` items, plus `rebuildable` only with an explicit `--class rebuildable`, and only items selected by default. [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §7 adds `clean --items`, an explicit list that counts as interactive consent **because a person confirmed it**. An agent is not a person. A list an agent writes cannot carry a person's consent, and treating it as one would let a prompt-injected or simply mistaken agent move `review` items to the Trash.
6. **Isolation.** Inside a worktree, Claude Code blocks commands whose working directory is the main checkout and git commands redirected into it. `ddc` runs with absolute paths and its own data directory, so it is not affected; but a `git worktree remove` that `ddc` runs on the agent's behalf acts on the shared `.git`, which is the point of removing a worktree, and it must be `ddc`, not the agent's shell, that runs it.

## Decision

### 1. Agent residue is a provenance, not a new risk class

The four classes stay. A finding gains an optional `origin` field: `{harness: "claude-code" | "codex" | "opencode" | "cursor" | "git", kind: "worktree" | "session-data" | "runtime-cache"}`, set when the rule recognizes a harness's own structure (item 2 and item 3). A `git.worktree` finding also gains an optional `worktree` field that carries its state (item 2): `{branch, detached, clean, ignored_content, own_commits, submodules, locked, in_use, idle_days, probe}`. The report, the plan and the app group by `origin`; the classes and the methods decide what happens, as before.

Both are new optional fields, so they do not raise `schema_version` under the versioning policy of [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]. The `notes` enum is **not** extended: that policy names `notes` among the enums whose new values bump the version, so every reason a worktree is not selected lives in `worktree`, never in `notes`. A reader that ignores unknown fields keeps working (the bumps of §3 are separate), and a fixture test proves it.

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

### 3. Harness data: known-path rules, a liveness check, and a protected memory

*Revised 2026-10-30, before acceptance, for the DDC-002 story "Review window: coding agents' data as a group". The owner chose on 2026-10-30 to keep one design for agents here (way A of that story) and to let a stale session's transcript go to the Trash after a liveness check. The changes against the 2026-10-18 text: transcripts become a `review` item instead of report only; Cursor's agent data joins; sessions become entries of their folders; Trash moves get the liveness check below; `shell-snapshots` leaves the catalog; the rules form one "Coding agents" group. The research note [[research/coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session|Coding-agent harnesses on macOS: where their data lives and how to tell a live session]] holds the paths and the live signals, which `ddc disk agents` (`agents::HARNESSES`, `agents/claude.rs`, `agents/codex.rs`, `agents/running.rs`) already reads.*

`~/.claude`, `~/.codex` and `~/.cursor` are harness folders. Rules inside them, all in one group, "Coding agents":

| id | Paths (from `~`) | Class | Method | Liveness | Hint |
|---|---|---|---|---|---|
| `agent.claude-code-scratch` | `.claude/debug`, `.claude/todos`, `.claude/statsig`, `.claude/logs` | `safe` | `delete` | §5's in-use check, as today | debug logs and legacy folders the harness no longer writes; a live session's open log makes the item `in-use` |
| `agent.claude-code-file-history` | `.claude/file-history`, by entry | `review` | `trash`, per entry | the session | "Checkpoints of this session. /rewind cannot use them once moved." |
| `agent.claude-code-transcripts` | `.claude/projects/<project>`, by entry | `review` | `trash`, per entry | the session | "This session's transcript. --resume of it fails until it is put back from the Trash, before it is resumed." |
| `agent.claude-code-history` | `.claude/paste-cache`, `.claude/usage-data` | `review` | `trash` | none (see below) | "A recalled prompt loses its pasted text; past /insights reports are gone." |
| `agent.codex-sessions` | `.codex/sessions`, `.codex/log` | `review` | `trash` | the harness | "Codex's past threads; Codex cannot reopen them once moved. Quit Codex, and its app-server, to move them." |
| `agent.codex-runtimes` | `.cache/codex-runtimes` | `review` | `trash` | the harness | "Downloaded runtimes; Codex fetches them again on demand. Quit Codex to move them." |
| `agent.opencode-data` | `.local/share/opencode/storage`, `.local/share/opencode/snapshot`, `.local/share/opencode/log`, `.cache/opencode` | `review` | `trash` | the harness | "opencode's stored sessions, snapshots and logs; opencode cannot show those sessions once moved. Quit opencode to move them." |
| `agent.cursor-agent-data` | `.cursor/projects`, `.cursor/ai-tracking` | `review` | `trash` | the harness | "Cursor's agent transcripts per project, and its AI-code tracking; Cursor cannot show them once moved. Quit Cursor to move them." (the plan gate confirms what `projects` holds) |

**Never claimed, at any depth:** `.claude/projects/<project>/memory/` (auto memory, the one thing there that is not reproducible) and every other child of a project folder that is not a session; `.claude/sessions/` (the live records themselves); `.claude/plugins` (what the person installed); `.claude/shell-snapshots` (a live session sources its snapshot before every Bash command, without holding it open, so neither §5's in-use check nor a session check sees the use; Claude Code removes it on a clean exit, and a crash leaves a few kilobytes); the newer per-session folders (`image-cache`, `uploads`, `tasks` and the like) until the plan gate classifies them; `.cursor/extensions`, `.cursor/plugins` (what the user installed); `.cursor/plans`, `.cursor/chats` (the person's plans and chats, beside the agent's transcripts); `.local/share/opencode` itself, `auth.json` (provider credentials) and `opencode.db` with its `-wal` and `-shm` (opencode's live index; the folder is never claimed whole); `.cursor/worktrees` and `.codex/worktrees` (agent work on the user's repositories, which §2's worktree rules cover). The Cursor desktop app's caches under `~/Library/Application Support/Cursor` stay `ide.cursor-cache`; these rules do not claim them again.

**Granularity: sessions are entries.** A per-session item is an *entry* of its folder, in the per-entry model of the safety ADR's user-data amendment (2026-10-20), which this section extends to `review` rules.
- `agent.claude-code-transcripts` matches each `.claude/projects/<project>` as a known child: one finding per project folder, `review`, method `none` for the folder itself. Its entries are the sessions, matched positively by a session-id (UUID) name: `<session>.jsonl`, with `<session>/`, `<session>.jsonl.superseded-*` and `<session>.orphaned-*.jsonl` as companions moved with it in one action. `memory/` and every other child that is not a session are neither entries nor claimed. A companion without its `<session>.jsonl` is never listed, so `ddc` never removes it; Claude Code's own sweep does.
- **An entry with companions.** Its identity is the `<session>.jsonl`'s `(dev, ino)`, as for any entry. Its companions are not taken from the list: the plan and the guard re-derive them by name under the same parent directory descriptor, each checked as a direct child of a matching name. The moves run companions first, the transcript last, so a failure leaves the resumable file in place. If any move fails, the rest stop and the item is `failed`; the journal writes one line per path moved, a companion's line names the entry's `<session>.jsonl` in a new optional `entry` field (the journal has no item field; see "The journal" in the plan-gate list), so each can be put back. The entry's `bytes` is the sum of its files.
- `agent.claude-code-file-history` uses the same model over `.claude/file-history`, whose direct children are the sessions.
- These folders list every entry, with no size floor (the user-data folders list entries of 10 MiB or more); the report's cap of the 100 largest entries per folder still applies, the rest summed.
- An entry goes only to the Trash: `--permanent` or `action: delete` on it is `action-not-allowed`, as for a user-data entry.
- The Review window shows the project folder as the row and its sessions as its entries, as it shows a Downloads folder.

**Liveness.** A move to the Trash never counts open files (safety ADR, the 0.2.1 amendment), so these rules carry their own check. **This amends safety ADR §5 and its 0.2.1 amendment, dated 2026-10-30:** for the rules of this table a Trash move carries a liveness check, and it runs in the plan (the pruned walk of `--items` and the dry run) as well as before each action. The scan records the state as of the scan, as an entry lock, so the app can grey a live session; the dry run and the executor decide.

- **The session** (Claude Code's transcripts and file history). A session is live when:
  - a `sessions/<pid>.json` names it, and its process has the user's uid and started no later than `startedAt`, or its UTC start equals `procStart` (`agents/claude.rs`). A reused pid starts later and fails both;
  - or the transcript's or a companion's own modification time is within the last 10 minutes (`entries::RECENT`; a write deep inside `<session>/` is not seen). This covers a session that has not written its record yet (the record trails the process start by up to a minute) and a version without records;
  - or a process of the user holds one of its files open.

  A `sessions/` folder that is missing or unreadable, any record that cannot be parsed, a record that cannot be decided and names no session, or a live record that names no session refuses every Claude Code entry as `unknown`; an undecidable record that names a session refuses that session only. Today `agents::claude::sessions` drops a single unparsable record and reports `Unparsed` only when every record fails; a strict reader beside it reports each parse failure and is the one the guard uses, while `sessions` keeps its lenient result for `disk agents` (`codex::threads` and `running::of` become `pub(crate)` for the guard).

  A `--resume` of a stale session started in the seconds before its move is caught only by its record, which can trail the process by up to a minute; a resume that only reads the transcript changes no time and opens no file for long. The Trash and the hint are the recovery (see Open Questions). A record whose process is gone (a crash leftover, which Claude Code clears on its next launch) does not block. Other sessions' entries stay eligible while one runs.
- **The harness** (Codex, opencode, Cursor). An item is refused while the harness runs:
  - Codex: a `thread-writer-locks/<id>.lock` is held open by a process of the user, matched by `(dev, ino)` (`agents/codex.rs`), **or** a Codex process runs (`running::of(Codex)`: the CLI or the app-server; a Codex process of kind `Other` (inside another app's bundle, such as ChatGPT's) counts as `unknown`). The app-server stays up on many machines, so `agent.codex-sessions` and `agent.codex-runtimes` are refused until the person quits it; the hints say so. Matching a lock to its rollout file would allow per-thread items, which the rollouts' dated folders do not fit as entries; it is left for later.
  - opencode: an `opencode` process runs.
  - Cursor: the Cursor app or a `cursor-agent` process runs (`agents/running.rs` learns `cursor-agent`).
- **None** for `agent.claude-code-history`. Its entries are keyed by content hash and report time, not by session. The loss of a Trash move (a recalled prompt without its paste, until it is put back) is small, restorable and named in the hint. A harness-level check would refuse it whenever Claude Code runs, which on the reference machine is always. §5's path-based in-use check still applies.
- **The refusal**, by item kind. A session entry is refused as `entry-locked`, the plan's and the guard's existing path for a locked entry, with two new lock values: `live-session {session, by: record | recent | open-file}` and `liveness-unknown {cause}` (the payload names settled at the plan gate). For these rules they replace `recent`: a change within 10 minutes is `live-session` with `by: recent`. The scan sets the same lock on the entry, so the Review window greys a live session; a `liveness-unknown` entry's hint says the harness's live records could not be read (an older Claude Code without `sessions/` greys every session). A whole-folder item (Codex, opencode, Cursor) is offered by the scan and refused at plan time and before its move with a new skip reason, `agent-live {harness, by}`, `by` being `running`, `lock-held` or `unknown`; the Review window shows the refusal with the hint. The new lock values raise the scan-report schema version; the new lock values and `agent-live` raise the clean-report schema version. The checks cost one record read per Claude Code session and, for Codex, a sweep of the user's processes' open files (a few hundred milliseconds), done once per clean and re-read before each Codex item.
- **A transcript Claude Code's own sweep removes** between the scan and the clean is skipped `missing`, as any vanished item.

**Settled at the plan gate, 2026-10-30.** The reference machine was read by names, sizes and times only. These points refine the section above; where they differ, they hold.

- **Companions by name.** A project folder there holds `<session>.jsonl`, `<session>/` (with `subagents/`, `tool-results/`, `custom-title.json`), `<session>.superseded-<n>` (without `.jsonl`), `<session>.desktop-released.json`, `memory/`, `sessions-index.json` and a stray `.claude/`. So a companion is any sibling named `<session>` or `<session>.<anything>` other than the transcript itself; a sibling whose name does not start with a session id is never an entry. A companion without its `<session>.jsonl` (one `desktop-released` file there) is never listed.
- **Field names.** `EntryLock` and `SkipReason` are tagged by a field named `reason`, so the payloads use other names: `live-session {session, by: record | recent | open-file}`, `liveness-unknown {cause}`, `agent-live {harness, by: running | lock-held | unknown}`.
- **The journal.** It has no item field today. Each moved path is its own line, as above; a companion's line carries a new optional `entry`, the transcript's path. An optional field does not move the version (the Rust core ADR's policy; `needed_by`, 2026-10-29, did not). The journal moves because an `item` line flattens the outcome, which carries `SkipReason` and `EntryLock`, the enums of the previous point, as `tool-not-ready` moved it 2 → 3.
- **Every version that embeds these enums moves once, in one change:** the scan report (2 → 3), the clean report (4 → 5), the progress events (2 → 3) and the journal (4 → 5, assuming `ddc keep remove`, which takes 3 → 4, lands first), with the app's decoders in the same change.
- **`origin`** does not exist yet. The "Harness data rules" story introduces `Origin {harness, kind}` with §1's full value set; the worktree story fills it for worktrees.
- **Worktree folders.** `.cursor/worktrees` and `.codex/worktrees` are never a finding themselves and get no rule of their own (no rule matches a plain folder of that name); the rules keep working inside them, as §2 says of any worktree.
- **Auto memory** needs no rule of its own: the project folder is a claim, so no rule runs inside it, and only a session id is an entry.
- **`.claude/plugins`** (130 MB there) is sealed like Cursor's extensions and plugins: what the person installed, which Claude Code's documentation says not to delete. Nothing inside it is a finding, a build output included.
- **Codex locks.** A missing `thread-writer-locks` folder means no lock is held, and a running Codex process alone decides; an unreadable one is `agent-live` with `by: unknown`.
- **"Only a person" is per rule.** `--yes` with `--class review` is already refused (`review-with-yes`), and `--yes` never selects `review`. As defence in depth, the rules of this section carry a `listed_only` flag, checked when the plan is built and again when an action is allowed, so they stay out of any non-listed plan whatever their class.
- **The switch.** A rule whose method is `none` but whose entries go to the Trash can be switched off in Settings, as one with an action can, **unless its class is `user-data`**: Downloads, Pictures, Movies, Music and the iOS backups stay without a switch, as today. "Coding agents" is a switchable group.
- **Absent here:** `.claude/todos`, `statsig`, `logs` and `usage-data`, `.cursor/plans` and `.cursor/chats`, opencode's `auth.json`. The rules keep them (a rule whose path is gone finds nothing), and fixtures cover them.

Only a person removes the `review` items: picked in the Review window or listed in `--items`, never inside `--yes` or `--agent`. The `safe` scratch is the one exception: selected by default and inside `--yes`; under `--agent` only behind the Settings opt-in the owner required on 2026-10-28 for agent cleans of the `safe` class, which §5 must adopt as a dated amendment (see Open Questions). Session scratchpads under the system temp directory (`CLAUDE_CODE_TMPDIR`) lie outside `$HOME` and outside the default scan roots; the OS clears them, and `ddc` does not look for them.

*Amendment, 2026-11-01 (reading only), replacing "`ddc` does not look for them" for reading:* `ddc disk agents` now reads the current user's Claude Code temporary base (`<tmp>/claude-<uid>`, or under `CLAUDE_CODE_TMPDIR`), read-only: entry names, sizes and the stat of a session's own files, never their content. It reports each session's temporary folder with that session's liveness, and the base's remainder. Removal stays out of scope until story "Coding agents' temporary folders outside the home" amends this section; story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees" carries the reading half.

### 4. `ddc footprint`: a scoped report for one project, in seconds

`ddc footprint [ROOT] [--json]` answers an agent's question about one project, not the home folder:

- `ROOT` defaults to the current directory; the main checkout is found through `gitdir:` and `commondir` when `ROOT` is a worktree.
- It lists the main checkout and every linked worktree from `git worktree list --porcelain`, plus every directory under `<main>/.claude/worktrees/` that git does not list (a worktree removed from git but left on disk is a `git.worktree-orphan`).
- For each: path, `origin`, and the `worktree` state of item 2 (branch or detached, clean or dirty, ignored content, own commits, submodules, locked, in use, idle days, probe result), obtained under the same probe contract, and the findings inside it with their class and size, found by walking only those directories with the ordinary rule visitor.
- Totals: what is reclaimable now (selected by default under this ADR's conditions), what would be reclaimable once the running sessions end (worktrees that fail only condition 2 or 3), and the harness folders of item 3 from `last-scan.json` when a report exists, with its age.
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
6. **Transcripts and history: the person's call, never while live.** They have a retention setting and a purge command, and they hold the resume state, so they are never removed by default or by an agent. Revised 2026-10-30: a person may move a stale session's transcript to the Trash, which keeps it restorable; the liveness check keeps the live session's out of reach.
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

**Rejected because**: the harness has the retention policy; `ddc` reports the size and names the setting. Revised 2026-10-30: transcripts and `file-history` are `review` to the Trash, after the liveness check, which is neither `safe` nor report only.

## Consequences

**Positive**:
- Dead agent worktrees and the environments inside them are found, grouped under their project, and can go without a person, within two days instead of two weeks.
- An agent sees its own project's footprint in seconds and can act on it through one bounded command.
- Harness data stops being "outside findings": its size has a name and a knob, and a person can move a stale session's data to the Trash from the Review window (revised 2026-10-30).
- The safety model is unchanged where it matters: `review` stays a person's choice, the executor's re-checks apply to every `--agent` action, and worktree removal goes through git.

**Negative / trade-offs**:
- A scan runs git probes on candidate worktrees; on a worktree with a huge tree `status --ignored` and the content comparison can take seconds, bounded by the 30 s timeout and `probe: unknown`, which keeps the worktree `review`.
- A harness worktree with any unique ignored file, an initialized submodule or a detached `HEAD` on a unique commit is never selected by default; the report says why, and the inner artifacts are still cleaned. Fewer automatic removals than the first draft promised; none that lose data.
- A person's named worktree, kept clean on purpose and untouched for two days, is `rebuildable` and selected by default if it was created by a harness; `.ddc-keep` in it, or `--worktree-min-age`, keeps it. The README says so.
- `--agent` lets a script without a person remove `rebuildable` items of one project tree. That is the cost of cleaning after agents; the journal records every such action with `consent: agent`.
- Two new CLI contracts with a `schema_version`: the footprint report and the `--agent` clean report (a variant of the DDC-002 clean report).
- A `--resume` begun in the seconds before a move, before its record is written, can lose its transcript to the Trash mid-session; the record check, the 10-minute rule when the resume writes, and the hint narrow it, the Trash puts it back.
- Four harnesses' private layouts are now in the catalog and can change under `ddc`; the plan gate confirms paths on the reference machine, and a rule whose path is gone simply finds nothing.

**What changes in code / process**:
- `ddc-core`: `origin`, `worktree` and `Project.worktree_of` in the model; the `git.worktree` and `git.worktree-orphan` rules and their matcher in `rules/`; the six conditions in `rules/classify.rs` with the git probes behind one guarded invocation in a new `git` module (the probe contract, the ignored-content accounting, the surviving-ref set); `contained` in `plan.rs`; the `native` worktree removal with the location check in `exec/native.rs`, no `prune`; the harness known-path rules in `rules/catalog.rs` with the never-claimed list (`memory/` and the others of §3) in the visitor.
- `ddc-cli`: `footprint` command and `docs/footprint.schema.json`; `clean --agent --project` with its refusals, the `not-consented` skip reason and `consent` in the journal; `--worktree-min-age`; render groups by worktree.
- Repository: `agents/` with the skill and the `AGENTS.md` snippet; README sections for `footprint` and `--agent`.
- Safety ADR §5 and menu bar ADR §7: dated amendments pointing here. Safety ADR §5 and its 0.2.1 amendment also take the liveness check on Trash moves of §3 (2026-10-30).
- For §3 (revised 2026-10-30): the liveness guard in `exec/guard.rs`, reusing `agents::{claude, codex, running}`; a strict reader of Claude Code's records beside `agents::claude::sessions`, reporting each parse failure, and `pub(crate)` for `codex::threads` and `running::of`; `SkipReason::AgentLive` for whole-folder items; the entry model extended from `ENTRY_RULES` to a per-rule entry policy, with `EntryLock` `live-session` and `liveness-unknown` (replacing `recent` for these rules), no size floor, companion files re-derived under the parent descriptor and moved transcript-last, and the four schema bumps of the plan-gate list (scan, clean, progress, journal); a "Coding agents" group in `GROUPS` (`rules/listing.rs`); `running::classify` learns `cursor-agent`.
- Depends on the DDC-002 stories "CLI: clean an explicit item list" (`--items`, `--json`, exit codes) for the `--agent` entry point; the worktree rules and `footprint` do not.

## Open Questions

- The exact marker Claude Code writes into a worktree's git metadata, and whether Codex and opencode mark theirs. Until confirmed, `<main>/.claude/worktrees/` is the only Claude Code signal, and the other two harnesses have no worktree signal.
- The minimum git version the probe contract is pinned to, and whether `--ignored=matching` and `--porcelain=v1` behave the same on the oldest version Homebrew ships on macOS 26.
- Whether `--worktree-min-age` should default to 2 days or to the harness's own idea of "old" (`cleanupPeriodDays` is 30). Two days is the proposal; the first weeks of use on the reference machine decide.
- `claude purge <path> --yes` is non-interactive and per project, but it also deletes `projects/<project>/memory/`, so it can never be the method here. Closed 2026-10-30.
- Whether Claude Code writes `sessions/<pid>.json` before it opens a transcript on `--resume`. If not, a resume started between the plan and the move is caught only once the record lands (Consequences). The plan gate checks it with a throwaway session: names and times of `sessions/` and the transcript, never contents.
- Whether a record's `sessionId` follows `/clear`, which starts a new session id in the same process. If it does not, the live session's current transcript is protected only by the 10-minute rule; the plan gate checks it the same way.
- Whether a resumed session whose transcript was moved recreates `<session>.jsonl`, so that putting it back conflicts. The hint already says "before it is resumed".
- The Settings opt-in for agent cleans of the `safe` class (owner, 2026-10-28) is not yet in §5; a dated amendment adds it before `--agent` ships.
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

*Last updated: 2026-10-30*
