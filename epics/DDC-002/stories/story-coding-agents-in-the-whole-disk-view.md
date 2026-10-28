---
type: story
id: "story-coding-agents-in-the-whole-disk-view"
epic: "DDC-002"
title: "Coding agents in the Whole disk view"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [cli, app, agents, whole-disk]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Coding agents in the Whole disk view

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner asked, in a comment on artboard 13 of the design canvas (2026-10-28), whether the Whole disk view could single out what coding-agent harnesses keep, list their sessions, and say which session is still active. On the reference machine the harnesses hold about 11 GB under the home folder. The Whole disk view shows that only as part of "Rest of the home folder".

This story adds a **Coding agents** section to the Whole disk view (story "Whole disk in the app", DDC-002). Per harness it shows:
- the folders it keeps, with their sizes;
- its sessions that are live now, where the harness leaves a signal for that.

The research note [[research/coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session|Coding-agent harnesses on macOS: where their data lives and how to tell a live session]] found the signals:
- **Claude Code:** `~/.claude/sessions/<pid>.json`. The session is live when a process of the user has that pid and started no later than `startedAt`.
- **Codex:** `~/.codex/thread-writer-locks/<thread-id>.lock`. The thread is open while a process of the user holds that file.
- **Cursor and opencode:** only whether the app or the process runs.

**Read-only.** The section removes nothing and offers no button that does. Which harness data may be cleaned, and how, belongs to the epic "Agent footprint" (DDC-003): its story "Harness data rules" and its ADR "Agent footprint: worktrees, harness data and bounded consent for agents". This story never adds a catalog rule and never changes a finding.

**What it reads:**
- the Claude Code session records, and of them only the fields listed below;
- the names of the Codex lock files;
- the process list and the open files of the user's processes;
- allocated sizes.

**What it never reads:**
- transcript content;
- prompts;
- the hidden `.key` files beside the session records;
- any harness database.

It takes no lock.

**The section is a breakdown, not a new area.** Its bytes are part of "Rest of the home folder". It changes neither the container bar nor the home row.

## Decomposition

Rough until the plan gate.

- [ ] **Amendment to the menu bar ADR, "Menu bar app: Swift shell over the ddc CLI"**, before any code.
  - **§1, the lane rule** (amends the 2026-10-28 amendment for `disk`). A kind that takes no lock and writes nothing another command reads runs beside the queue. Each such kind has one running process at a time, and kinds never wait for each other. Today these kinds are `disk` and `disk agents`. A whole-disk walk of several minutes must not hold up a session list.
  - **§6, the new command.**
    - Its schema, and that its exit codes follow `ddc disk`: 0 on success, a closed stdout included, and the usage and environment errors of every command.
    - The carve-out: fields passed through from a harness (`status`, `entrypoint`) are free strings, never strict enums, because their values are another program's internals.
  - **Consequences.** The count of CLI contracts becomes eight: the rules listing of item 6.8 joins the list too.
- [ ] **ddc-core: `agents::HARNESSES`.** One table in code holds every harness's folders, taken from the research note: `~/.claude`, `~/Library/Application Support/Claude/claude-code`, `~/.codex`, `~/.cache/codex-runtimes`, `~/.cursor`, `~/.local/share/opencode`, `~/.cache/opencode` and `~/.config/opencode`. Nested entries: `~/.claude/projects`, `~/.codex/sessions` and `~/.codex/worktrees`. They are the source of the app's nested folder rows.
  - It excludes the rest of `~/Library/Application Support/Claude` (the desktop chat app) and `~/Library/Application Support/Cursor` (editor state).
  - DDC-003's catalog rules should read this table rather than keep their own list. That is recorded under Dependencies, for that epic's plan gate.
- [ ] **CLI: `ddc disk agents [--json] [--no-sizes]`.** `--no-sizes` skips the walk; the report then marks its sizes absent, while the transcript counts, which come from names, stay. It is read-only, takes no lock, writes no file and sends no notification. The report has its own `AGENTS_SCHEMA_VERSION` and a schema file in `docs/`. Per harness it gives:
  - `id`: `claude-code`, `codex`, `cursor` or `opencode`;
  - `running`: the processes of the harness that run, each with its kind. The kind is ddc's own classification, a strict enum with an `other` case: `cli`, `app-server`, `app`, `other`. They come from the process list of `sys::procs`, matched by the exact executable basename (`claude`, `codex`, `opencode`) or the app bundle (`Cursor`), never by a substring. The Claude desktop chat app runs as `Claude` and is never Claude Code;
  - `folders`: each path with its allocated bytes. A nested folder (`~/.claude/projects` inside `~/.claude`) is reported inclusive and marked nested. The harness total sums only the top-level folders;
  - `sessions`: for Claude Code and Codex only.
    - Claude Code sessions carry `id`, `started_at`, `status`, `entrypoint`, `name` and `cwd`.
    - Codex threads carry `id` and the rollout file's start time.
    - Each session has a state: `live`; `stale` for a record left behind with no live process; or `unknown` for a record without the fields the test needs.
  - `records`: `readable` or `unknown`, with a reason. It is `unknown` when the records folder is missing or no record has the expected shape. Then no session is called stale, because a changed format must not read as "everything ended";
  - `transcripts`: the count and allocated bytes of the transcripts that no live session writes. For Claude Code, a transcript is `projects/<p>/<id>.jsonl` together with its `projects/<p>/<id>/` folder. For Codex, it is the `sessions/` rollouts.

  A harness whose folders are all absent is left out. Liveness is never stored: each run reads it again.
- [ ] **Liveness, Claude Code.** For each `sessions/<n>.json` whose name is a number:
  - The record is read for `pid`, `sessionId`, `startedAt`, `procStart`, `status`, `entrypoint`, `name` and `cwd` only. `procStart` is read for the comparison and never emitted. `messagingSocketPath` and the `.key` files are never read into the report.
  - The session is live when `sys::procs::bsd_info(pid)` finds a process whose uid equals `uid()` and for which either proof holds: its start time is no later than `startedAt`, or its start time formatted as UTC ctime (`%a %b %e %H:%M:%S %Y`) equals `procStart`. The second proof covers a resumed session that may keep an older `startedAt`.
  - It is stale when there is no such process.
  - It is unknown when the record has no `pid` or `startedAt`, or when its `pidDomain` is present and not `darwin` (the pid is not a local one).
  - An empty `sessions/` folder is readable with no sessions: Claude Code removes records on exit. `records: unknown` is for a missing folder, or for candidate records none of which parses.
  - Records are parsed key by key from a JSON map, with a size cap, and only files named `<digits>.json` are read. The per-session key files are named `<pid>.<hex>.key`.
  - An npm-installed Claude Code runs as `node` and does not show under `running`; its sessions are still found from the records.
- [ ] **Liveness, Codex.** A thread is open when any process of the current user holds `thread-writer-locks/<id>.lock` open.
  - `sys::procs` gains `open_vnodes(pid)`, which returns `(dev, ino, path)` for each open file from the fd's vnode info (`PROC_PIDFDVNODEPATHINFO`, `vst_dev` and `vst_ino`). It is called per pid, because the process snapshot leaves the open files of the caller's children empty.
  - Only the lock files in `thread-writer-locks/` are ever `stat`ed, to get their `(dev, ino)`; no open path of any process is `stat`ed, since a file in a protected folder could make macOS ask.
  - Only names of the form `<thread-id>.lock` count; `.coordination.lock` and other hidden files are ignored.
  - No `lsof` and no `flock` probe.
  - The rollout is found by the `<id>` suffix of its file name.
- [ ] **Sizes.** The size-only visitor of `ddc disk` walks these folders only, never the whole home. A folder that cannot be read is counted as unreadable, as in the disk report.
- [ ] **Skips.** `skips` in `settings.json` bind the walk and the record read alike (menu bar ADR §6.10). A harness folder under a skip is never opened, and the harness is reported with the skip as its reason. A skip inside a harness folder (`~/.claude/projects`) is never opened either: that folder is reported skipped, and the transcripts read through it are reported absent with the same reason.
- [ ] **App: a Coding agents section** under the area rows of the Whole disk view.
  - **Rows.**
    - One row per harness: its name, what of it runs, and its total size.
    - Nested rows for its folders.
    - Nested rows for live sessions: the project folder name, the status and the start time. Codex threads are labelled "open in Codex".
    - One line for the remaining transcripts: their count and size.
    - When the records are `unknown`, one line instead of the sessions: "ddc cannot read Claude Code's session records".
  - **When it runs.** The view runs `ddc disk agents --json --no-sizes` and then `ddc disk agents --json`, one after the other in its own read-only lane, when it appears and when the section's Refresh button is pressed. The first report shows the session rows at once; the second adds the sizes. The section shows even when there is no whole-disk report yet. It never runs from the scheduled path.
  - **A failed run** shows its message in the section and keeps the area rows. When the sizes run fails after a good `--no-sizes` run, the session rows of the first report stay, with the sizes shown as absent.
- [ ] **Canvas.** Bring the purple "AI agent harnesses" block on artboard 13 in line with this section, and replace its "Idle N d" chips: a session either runs or is not listed. Re-read the canvas before any republish, because another session changed it.
- [ ] **README.** One line on `ddc disk agents` in the CLI guide and in the Whole disk bullet. It must say that the output shows project folders and session names.

## Implementation Plan

## Acceptance Criteria

**Test**
- [ ] `ddc disk agents --json` validates against its schema file, and `DDC_UPDATE_SCHEMA=1` regenerates it.
- [ ] Claude Code liveness, over a temp HOME with fixture records:
  - a record whose pid is this test process, with `startedAt` after its start, is live;
  - a record with a dead pid is stale;
  - a record whose pid is this process but whose `startedAt` is before the process start (the reused-pid case) is stale;
  - a record without `pid` is unknown and does not make the others stale;
  - a record whose `startedAt` is before the process start but whose `procStart` equals the process's UTC start is live (the resumed-session case);
  - a missing `sessions/` folder gives `records: unknown`.
- [ ] A record with an unknown `status`, an unknown `entrypoint` and unknown extra keys still decodes, in the CLI and in the app, and its session is live.
- [ ] The report never contains `messagingSocketPath`, `procStart`, a `.key` file's name or its contents.
- [ ] `running`: a process named `Claude` in a `Claude.app` bundle is not Claude Code; a process whose name only contains `claude` is not either. A fixture with all three proves it.
- [ ] Codex liveness, over a temp HOME:
  - a lock file that a test child process holds open is open;
  - the other lock files are not;
  - `.coordination.lock` is ignored;
  - no `flock` is called anywhere in the module, and no open path is `stat`ed (a source test, as `ContractTests` does for the app).
- [ ] Transcripts: a live Claude Code session with no transcript yet is tolerated. The remaining-transcripts count excludes the live sessions' transcripts.
- [ ] Sizes:
  - a fixture tree with a hard link counts it once;
  - an unreadable folder is reported unreadable, not as zero;
  - a nested folder is inclusive and marked nested;
  - the harness total sums the top-level folders only.
- [ ] With `~/.claude` in `skips`, no record under it is read and no size is walked; the harness is reported as skipped.
- [ ] A harness with no folders is absent. Cursor and opencode carry `running` and `folders` but no `sessions`.
- [ ] A fixture transcript and a fixture `opencode.db` without read permission do not make the command fail, and their sizes are still reported.
- [ ] `CLIRunnerTests`: a `disk agents` run is not delayed by a `disk` run held at a gate, nor by a queued scan. A second `disk agents` request with the same arguments joins the running one; one with other arguments waits for it and is then run.
- [ ] App:
  - a fixture report decodes;
  - the section shows one row per harness, with live sessions nested, the transcripts line, and the unknown-records line;
  - a failed run keeps the area rows and shows its message;
  - a failed sizes run after a good `--no-sizes` run keeps the session rows;
  - a Refresh pressed while the two runs are going starts no third run.
- [ ] `ContractTests`: the app runs `disk agents` only from the Whole disk view; `ScheduledWork` and `PopoverModel` never run it; nothing in the section sends a destructive command.

**Manual (owner)**
- [ ] On the reference machine, with a few Claude Code and Codex sessions open, the section lists them as live with their project folders. After a session is closed, Refresh no longer lists it, and the count of remaining transcripts rises by one.
- [ ] The harness sizes in the section are within 5 % of `du -sk` on the same folders.
- [ ] `ddc disk agents --json` finishes in under 10 s on the reference machine, and `ddc disk agents --json --no-sizes` in under 1 s. Record the time and the entry count in the Final Summary; the research note measured 5.1 s for about 124,000 entries without the Claude desktop folder.

## Final Summary

## Technical Notes

- **Privacy.** The report and the app show project folder paths and session names, which are personal. They stay on the machine, like `last-scan.json`. Fixtures in the code repository use `/Users/example`, and no real session id or path enters the vault.
- **Why `disk agents` is a command of its own and not a section of `last-disk.json`.** A whole-disk report can be a day old, and a session list that old is wrong. The sizes alone could go into the disk report, but then the app would draw two dates for one section.
- **Why the name `disk agents`.** It sits beside `ddc disk free` and says where the answer shows in the app. A bare `ddc agents` would read too close to DDC-003's `ddc clean --agent` and the repository's `agents/` folder.
- **Why not the DDC-003 rules.** Those rules decide what may be cleaned, and they produce findings in the home scan. This section names and sizes. When DDC-003 needs to hold back a live session's data, it can reuse this story's liveness code; that is its ADR's decision.
- **The visitor.** The size-only visitor of `ddc disk` takes any set of roots. Its hard-link dedup covers these folders as it covers the areas.
- **Open questions for the plan gate:**
  - whether a resumed session keeps its old `startedAt`: resume a throwaway session and compare its record's `startedAt` and `procStart` with `ps -o lstart=` in UTC;
  - whether the ChatGPT app's embedded Codex writes to `~/.codex`, so that "Codex runs" names the right binary;
  - the time zone of the time in a Codex rollout's file name; until settled, it is passed through as written.

## Dependencies

- "Whole disk in the app" (DDC-002): the view this section joins, and the `CLIRunner` lane it shares a rule with.
- "Whole disk: what lies outside the home folder" (DDC-002): the size-only visitor.
- The research note [[research/coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session|Coding-agent harnesses on macOS: where their data lives and how to tell a live session]].
- "Harness data rules" (DDC-003): the cleaning side.
  - It is not a dependency in either direction.
  - Its path list differs from the research note in `.codex/log`, `.cache/opencode`, `.config/opencode` and the Claude desktop folder.
  - When that branch is next touched, its plan gate should take the paths from `agents::HARNESSES`.

## Attachments

- Design canvas, artboard 13, the owner's comment of 2026-10-28.

---

*Last updated: 2026-10-28*
