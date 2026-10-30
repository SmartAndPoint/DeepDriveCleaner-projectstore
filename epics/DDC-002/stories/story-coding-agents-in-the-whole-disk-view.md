---
type: story
id: "story-coding-agents-in-the-whole-disk-view"
epic: "DDC-002"
title: "Coding agents in the Whole disk view"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [cli, app, agents, whole-disk]
code_refs: ["crates/ddc-core/src/agents/", "crates/ddc-core/src/platform/macos/procs.rs", "crates/ddc-core/src/disk/visitor.rs", "crates/ddc-core/src/disk/mod.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-core/tests/agents.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/src/cmd/disk_agents.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-cli/tests/disk_agents_cli.rs", "crates/ddc-cli/tests/readme_cli.rs", "crates/ddc-cli/tests/structure.rs", "docs/agents.schema.json", "README.md", "app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift", "app/Packages/DDCKit/Sources/DDCKit/AgentsReport.swift", "app/Packages/DDCKit/Sources/DDCKit/AgentsSummary.swift", "app/Packages/DDCKit/Sources/DDCKit/AgentsModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/AgentsTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CLIRunnerTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/FakeCLI.swift", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/agents-report.json", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/agents-fast.json", "app/DeepDriveCleaner/AgentsSection.swift", "app/DeepDriveCleaner/DiskView.swift", "app/DeepDriveCleaner/DiskWindowController.swift", "app/DeepDriveCleaner/AppDelegate.swift"]
specs: []
started_at: "2026-10-28T14:35:35.291Z"
closed_at: "2026-10-30T14:28:50.924Z"
plan_updated_at: "2026-10-28T14:35:35.291Z"
---

# Coding agents in the Whole disk view

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
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


- [x] **Amendment to the menu bar ADR, "Menu bar app: Swift shell over the ddc CLI"**, before any code. — done: vault commit dc92fc1 (ADR §1 lane rule, §6.10)
  - **§1, the lane rule** (amends the 2026-10-28 amendment for `disk`). A kind that takes no lock and writes nothing another command reads runs beside the queue. Each such kind has one running process at a time, and kinds never wait for each other. Today these kinds are `disk` and `disk agents`. A whole-disk walk of several minutes must not hold up a session list.
  - **§6, the new command.**
    - Its schema, and that its exit codes follow `ddc disk`: 0 on success, a closed stdout included, and the usage and environment errors of every command.
    - The carve-out: fields passed through from a harness (`status`, `entrypoint`) are free strings, never strict enums, because their values are another program's internals.
  - **Consequences.** The count of CLI contracts becomes eight: the rules listing of item 6.8 joins the list too.
- [x] **ddc-core: `agents::HARNESSES`.** One table in code holds every harness's folders, taken from the research note: `~/.claude`, `~/Library/Application Support/Claude/claude-code`, `~/.codex`, `~/.cache/codex-runtimes`, `~/.cursor`, `~/.local/share/opencode`, `~/.cache/opencode` and `~/.config/opencode`. Nested entries: `~/.claude/projects`, `~/.codex/sessions` and `~/.codex/worktrees`. They are the source of the app's nested folder rows. — done: `crates/ddc-core/src/agents/mod.rs`
  - It excludes the rest of `~/Library/Application Support/Claude` (the desktop chat app) and `~/Library/Application Support/Cursor` (editor state).
  - DDC-003's catalog rules should read this table rather than keep their own list. That is recorded under Dependencies, for that epic's plan gate.
- [x] **CLI: `ddc disk agents [--json] [--no-sizes]`.** `--no-sizes` skips the walk; the report then marks its sizes absent, while the transcript counts, which come from names, stay. It is read-only, takes no lock, writes no file and sends no notification. The report has its own `AGENTS_SCHEMA_VERSION` and a schema file in `docs/`. Per harness it gives: — done: `crates/ddc-cli/src/cmd/disk_agents.rs`, `docs/agents.schema.json`
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
- [x] **Liveness, Claude Code.** For each `sessions/<n>.json` whose name is a number: — done: `agents/claude.rs`
  - The record is read for `pid`, `sessionId`, `startedAt`, `procStart`, `status`, `entrypoint`, `name` and `cwd` only. `procStart` is read for the comparison and never emitted. `messagingSocketPath` and the `.key` files are never read into the report.
  - The session is live when `sys::procs::bsd_info(pid)` finds a process whose uid equals `uid()` and for which either proof holds: its start time is no later than `startedAt`, or its start time formatted as UTC ctime (`%a %b %e %H:%M:%S %Y`) equals `procStart`. The second proof covers a resumed session that may keep an older `startedAt`.
  - It is stale when there is no such process.
  - It is unknown when the record has no `pid` or `startedAt`, or when its `pidDomain` is present and not `darwin` (the pid is not a local one).
  - An empty `sessions/` folder is readable with no sessions: Claude Code removes records on exit. `records: unknown` is for a missing folder, or for candidate records none of which parses.
  - Records are parsed key by key from a JSON map, with a size cap, and only files named `<digits>.json` are read. The per-session key files are named `<pid>.<hex>.key`.
  - An npm-installed Claude Code runs as `node` and does not show under `running`; its sessions are still found from the records.
- [x] **Liveness, Codex.** A thread is open when any process of the current user holds `thread-writer-locks/<id>.lock` open. — done: `agents/codex.rs`, `sys/procs.rs::open_vnodes` (now `platform/macos/procs.rs`)
  - `sys::procs` gains `open_vnodes(pid)`, which returns `(dev, ino, path)` for each open file from the fd's vnode info (`PROC_PIDFDVNODEPATHINFO`, `vst_dev` and `vst_ino`). It is called per pid, because the process snapshot leaves the open files of the caller's children empty.
  - Only the lock files in `thread-writer-locks/` are ever `stat`ed, to get their `(dev, ino)`; no open path of any process is `stat`ed, since a file in a protected folder could make macOS ask.
  - Only names of the form `<thread-id>.lock` count; `.coordination.lock` and other hidden files are ignored.
  - No `lsof` and no `flock` probe.
  - The rollout is found by the `<id>` suffix of its file name.
- [x] **Sizes.** The size-only visitor of `ddc disk` walks these folders only, never the whole home. A folder that cannot be read is counted as unreadable, as in the disk report. — done: `agents/sizes.rs`
- [x] **Skips.** `skips` in `settings.json` bind the walk and the record read alike (menu bar ADR §6.10). A harness folder under a skip is never opened, and the harness is reported with the skip as its reason. A skip inside a harness folder (`~/.claude/projects`) is never opened either: that folder is reported skipped, and the transcripts read through it are reported absent with the same reason. — done: `agents/mod.rs` (the name listings stop at a skip inside, after review)
- [x] **App: a Coding agents section** under the area rows of the Whole disk view. — done: `AgentsSection.swift`, `AgentsModel.swift`, `AgentsSummary.swift`
  - **Rows.**
    - One row per harness: its name, what of it runs, and its total size.
    - Nested rows for its folders.
    - Nested rows for live sessions: the project folder name, the status and the start time. Codex threads are labelled "open in Codex".
    - One line for the remaining transcripts: their count and size.
    - When the records are `unknown`, one line instead of the sessions: "ddc cannot read Claude Code's session records".
  - **When it runs.** The view runs `ddc disk agents --json --no-sizes` and then `ddc disk agents --json`, one after the other in its own read-only lane, when it appears and when the section's Refresh button is pressed. The first report shows the session rows at once; the second adds the sizes. The section shows even when there is no whole-disk report yet. It never runs from the scheduled path.
  - **A failed run** shows its message in the section and keeps the area rows. When the sizes run fails after a good `--no-sizes` run, the session rows of the first report stay, with the sizes shown as absent.
- [x] **Canvas.** Bring the purple "AI agent harnesses" block on artboard 13 in line with this section, and replace its "Idle N d" chips: a session either runs or is not listed. Re-read the canvas before any republish, because another session changed it. — done: artboard 13 republished 2026-10-28 (version 12): no bar segment, Live chips only
- [x] **README.** One line on `ddc disk agents` in the CLI guide and in the Whole disk bullet. It must say that the output shows project folders and session names. — done: README `ddc disk` section and the Whole disk bullet

## Implementation Plan

From the planner's pass (2026-10-28), in this order:

1. **ddc-core, `agents` module** (new): `mod.rs` holds `HARNESSES` and `report()`; `model.rs` the report types and `AGENTS_SCHEMA_VERSION`; `running.rs` the process classification; `claude.rs` and `codex.rs` the two liveness tests; `sizes.rs` the walk. `sys::procs` gains `open_vnodes`, `process_start` and a process list without open files. The walk reuses `disk::visitor::DiskVisitor`, made `pub(crate)`.
2. **CLI**: `DiskAction::Agents` under `ddc disk` (`cli.rs`), `cmd/disk_agents.rs`, the table in `render.rs`, `docs/agents.schema.json`, and the README lines.
3. **DDCKit**: `CLIRunner` lanes per kind (`disk`, `diskAgents`), where the same arguments join and other arguments wait; `AgentsReport` (decoding), `AgentsSummary` (rows), `AgentsModel` (the two runs, single flight).
4. **App**: `AgentsSection` in `DiskView`; one `AgentsModel` made in `AppDelegate` and passed to both places that build the view.
5. **Tests**: `ddc-core/tests/agents.rs`, `ddc-cli/tests/disk_agents_cli.rs`, `CLIRunnerTests`, `AgentsReportTests`, `AgentsSummaryTests`, `AgentsModelTests`, `ContractTests`, with fixtures under `/Users/example`.
6. **Canvas** and the vault close after the review.

## Acceptance Criteria

**Test**
- [x] `ddc disk agents --json` validates against its schema file, and `DDC_UPDATE_SCHEMA=1` regenerates it. — evidence: `ddc-core/tests/agents.rs::committed_agents_schema_is_current` (schema generated from the serialized type) + `disk_agents_cli.rs::the_report_lists_the_harnesses_and_their_live_sessions` (the output decodes as that type)
- [x] Claude Code liveness, over a temp HOME with fixture records: — evidence: `agents.rs::claude_sessions_are_live_stale_or_unknown`, `agents.rs::records_folder_missing_empty_or_unparsable`, `agents/claude.rs` unit tests
  - a record whose pid is this test process, with `startedAt` after its start, is live;
  - a record with a dead pid is stale;
  - a record whose pid is this process but whose `startedAt` is before the process start (the reused-pid case) is stale;
  - a record without `pid` is unknown and does not make the others stale;
  - a record whose `startedAt` is before the process start but whose `procStart` equals the process's UTC start is live (the resumed-session case);
  - a missing `sessions/` folder gives `records: unknown`.
- [x] A record with an unknown `status`, an unknown `entrypoint` and unknown extra keys still decodes, in the CLI and in the app, and its session is live. — evidence: `agents.rs::claude_sessions_are_live_stale_or_unknown` (CLI) + `AgentsTests.swift AgentsReportTests.decodes` (app)
- [x] The report never contains `messagingSocketPath`, `procStart`, a `.key` file's name or its contents. — evidence: `agents.rs::claude_sessions_are_live_stale_or_unknown` (asserts on the JSON)
- [x] `running`: a process named `Claude` in a `Claude.app` bundle is not Claude Code; a process whose name only contains `claude` is not either. A fixture with all three proves it. — evidence: `agents/running.rs::the_claude_desktop_app_and_a_name_containing_claude_are_not_claude_code`
- [x] Codex liveness, over a temp HOME: — evidence: `agents.rs::a_held_lock_is_an_open_thread`, `agents.rs::the_codex_sweep_stats_no_open_path_and_takes_no_lock`, `procs.rs::open_vnodes_of_a_child_carry_the_stat_identity`
  - a lock file that a test child process holds open is open;
  - the other lock files are not;
  - `.coordination.lock` is ignored;
  - no `flock` is called anywhere in the module, and no open path is `stat`ed (a source test, as `ContractTests` does for the app).
- [x] Transcripts: a live Claude Code session with no transcript yet is tolerated. The remaining-transcripts count excludes the live sessions' transcripts. — evidence: `agents.rs::transcripts_leave_out_the_live_sessions`
- [x] Sizes: — evidence: `agents.rs::sizes_count_hard_links_once_and_nested_folders_inclusively`, `agents.rs::an_unreadable_folder_is_unreadable_and_an_unreadable_file_is_still_sized`
  - a fixture tree with a hard link counts it once;
  - an unreadable folder is reported unreadable, not as zero;
  - a nested folder is inclusive and marked nested;
  - the harness total sums the top-level folders only.
- [x] With `~/.claude` in `skips`, no record under it is read and no size is walked; the harness is reported as skipped. — evidence: `agents.rs::a_skipped_harness_folder_is_never_opened` (also skips one level inside `projects/` and `sessions/`), `disk_agents_cli.rs::skips_in_the_settings_bind_the_record_read`
- [x] A harness with no folders is absent. Cursor and opencode carry `running` and `folders` but no `sessions`. — evidence: `agents.rs::a_harness_without_folders_is_absent_and_no_sizes_leaves_bytes_out`, `agents.rs::an_unreadable_folder_is_unreadable_and_an_unreadable_file_is_still_sized`
- [x] A fixture transcript and a fixture `opencode.db` without read permission do not make the command fail, and their sizes are still reported. — evidence: `agents.rs::an_unreadable_folder_is_unreadable_and_an_unreadable_file_is_still_sized`
- [x] `CLIRunnerTests`: a `disk agents` run is not delayed by a `disk` run held at a gate, nor by a queued scan. A second `disk agents` request with the same arguments joins the running one; one with other arguments waits for it and is then run. — evidence: `CLIRunnerTests.agentsBesideDiskAndQueue`, `CLIRunnerTests.agentsOtherArgumentsWait`
- [x] App: — evidence: `AgentsReportTests.decodes`; `AgentsSummaryTests.rows`, `unknownRecords`; `AgentsModelTests.failedFast`, `failedSizes`, `joins`; the section is drawn below the area rows in `DiskView.swift`
  - a fixture report decodes;
  - the section shows one row per harness, with live sessions nested, the transcripts line, and the unknown-records line;
  - a failed run keeps the area rows and shows its message;
  - a failed sizes run after a good `--no-sizes` run keeps the session rows;
  - a Refresh pressed while the two runs are going starts no third run.
- [x] `ContractTests`: the app runs `disk agents` only from the Whole disk view; `ScheduledWork` and `PopoverModel` never run it; nothing in the section sends a destructive command. — evidence: `ContractTests.agentsCallers`, `ContractTests.agentsContracts`

**Manual (owner)**
- [ ] On the reference machine, with a few Claude Code and Codex sessions open, the section lists them as live with their project folders. After a session is closed, Refresh no longer lists it, and the count of remaining transcripts rises by one. — moved to the runbook "Owner checks: manual acceptance across stories"
- [ ] The harness sizes in the section are within 5 % of `du -sk` on the same folders. — moved to the runbook "Owner checks: manual acceptance across stories"
- [ ] `ddc disk agents --json` finishes in under 10 s on the reference machine, and `ddc disk agents --json --no-sizes` in under 1 s. Record the time and the entry count in the Final Summary; the research note measured 5.1 s for about 124,000 entries without the Claude desktop folder. — moved to the runbook "Owner checks: manual acceptance across stories"

## Final Summary

**What changed.** `ddc disk agents [--json] [--no-sizes]` reports the coding-agent harnesses on the Mac (Claude Code, Codex, Cursor, opencode) from one table, `agents::HARNESSES`: what of each runs, its folders with allocated sizes, its live sessions and the transcripts no live session writes. Liveness is read again on each run, never stored. For Claude Code it comes from the session records against the process table, with the reused-pid and resumed-session cases. For Codex it comes from the thread lock files held open, read from the fd vnode info with no `flock` and no `stat` of an open path. The command is read-only, takes no lock and honours `skips`. The menu bar ADR was amended first (§1's lane rule, §6.10). The Whole disk view shows a Coding agents section under the area rows, fed by a fast `--no-sizes` run and then a sized one in their own lane. A failed run keeps the area rows. The README describes the command and says its output shows project folders and session names. Show in Finder on a folder or a harness came later, with story "Whole disk: Show in Finder and the honest action for each row" (PR #5).

**Why.** The owner asked, on artboard 13 of the design canvas, for the Whole disk view to single out what the harnesses keep (about 11 GB on the reference machine, shown until then only inside "Rest of the home folder") and to say which session is still live. The section is an overview; deleting belongs to story "Review window: coding agents' data as a group".

**Tests executed.** Every automated criterion is backed by the tests named beside it: `crates/ddc-core/tests/agents.rs`, `crates/ddc-cli/tests/disk_agents_cli.rs`, the unit tests in `agents/`, `CLIRunnerTests` and the DDCKit `Agents*` suites, plus `ContractTests.agentsCallers` and `agentsContracts`. The last Gate that ran the Rust jobs on main was 41065887647 (72c3765), and Gate 41068463547 (`app`, 385 tests) covers the app side. The reviewer re-ran the DDCKit suites on 2026-10-30: 59 tests in 6 suites passed.

**Risks and follow-ups.**
- The three manual criteria need the reference machine and the real HOME, which no agent may read. They are in the runbook "Owner checks: manual acceptance across stories", and the timing check says where to record its numbers.
- The harnesses' record formats are other programs' internals. A changed format reads as `records: unknown`, never as "every session is stale".
- No load profile covers `disk agents`, so its timing is only measured by hand.

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
  - It is not a dependency in either direction. Amended 2026-10-31 by story "Review window: coding agents' data as a group" (DDC-002): the cleaning side now exists, and this section links to it. It shows the Coding agents group's total from the last scan's findings and a Review… link that opens that group in the Review window. The section itself still removes nothing (`ContractTests.agentsCallers`).
  - Its path list differs from the research note in `.codex/log` (rules only), `.config/opencode` and the Claude desktop folder (note only); `.cache/opencode` joined the rules on 2026-10-30. Settled 2026-10-31: `agents::HARNESSES` was not adopted; the paths were confirmed on the reference machine (ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §3, "Settled at the plan gate").

## Attachments

- Design canvas, artboard 13, the owner's comment of 2026-10-28.

---

*Last updated: 2026-10-30*
