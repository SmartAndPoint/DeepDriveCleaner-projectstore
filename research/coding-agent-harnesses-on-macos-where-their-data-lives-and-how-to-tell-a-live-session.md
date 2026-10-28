---
type: research
slug: "coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session"
title: "Coding-agent harnesses on macOS: where their data lives and how to tell a live session"
status: final
date: 2026-10-28
authors: ["Evgenii Konev"]
tags: [agents, whole-disk, liveness]
review_status: reviewed
reviewed_at: 2026-10-28
---

# Coding-agent harnesses on macOS: where their data lives and how to tell a live session

## Question

The owner asked, in a comment on artboard 13 of the design canvas (2026-10-28), whether the Whole disk view could single out the files of particular coding-agent harnesses, list their sessions, and say which session is still active. This note answers three questions for Claude Code, Codex, Cursor and opencode:

1. Where does each harness keep its data, and how large is it on the reference machine?
2. What does each harness write per session?
3. Is there a signal, readable without side effects, that says a session is still live?

This note does not ask which harness data may be cleaned, or how. That belongs to the epic "Agent footprint" (DDC-003) and its ADR "Agent footprint: worktrees, harness data and bounded consent for agents", on the vault branch `ddc-003-agent-footprint`. The path table below is also meant to be the confirmation that DDC-003's story "Harness data rules" asks for at its plan gate.

## TL;DR

- **Claude Code** gives the best signal: one `~/.claude/sessions/<pid>.json` per running session.
  - The record holds the pid, `startedAt` (epoch milliseconds) and a `status`.
  - The session is live when a process of the current user has that pid and either started no later than `startedAt` or has a UTC start time equal to `procStart`. A reused pid belongs to a process that started after the record was written, so it fails both tests.
  - On the reference machine all 13 records belonged to live processes. Claude Code removes the record when a session ends normally.
- **Codex** keeps one empty `~/.codex/thread-writer-locks/<thread-id>.lock` per open thread.
  - While a process holds that file open, the thread is open in Codex.
  - On the reference machine 7 of 7 lock files were held, by 2 processes: the `codex` CLI and the Codex app-server daemon.
  - Each thread's transcript is `~/.codex/sessions/YYYY/MM/DD/rollout-<time>-<thread-id>.jsonl`, so a held lock names a transcript in use.
  - A daemon holding a thread means the thread is loaded, not that a person is typing in it. "Open in Codex" is the honest label.
- **Cursor** and **opencode** were not found to keep a per-session file that marks liveness. For them only "the app or process runs" can be shown.
- All of these formats are undocumented internals of each harness. Seen here: Claude Code 2.1.3xx, Codex 0.16x. A reader must handle a changed or missing format as "unknown", never as "ended".
- Sizes on the reference machine, allocated bytes:

  | Location | Size | Whose |
  |---|---|---|
  | `~/.claude` | 2.7 GB (`projects/` 2.3 GB) | Claude Code |
  | `~/Library/Application Support/Claude/claude-code` | 0.5 GB | Claude Code binaries the Claude desktop app runs sessions from |
  | the rest of `~/Library/Application Support/Claude` | 2.5 GB | the Claude desktop chat app; not a coding agent's data |
  | `~/.codex` | 3.0 GB (`sessions/` 106 MB, `worktrees/` 21 MB) | Codex |
  | `~/.cache/codex-runtimes` | 1.6 GB | Codex |
  | `~/.cursor` | 2.8 GB | Cursor's agent data and extensions |
  | `~/Library/Application Support/Cursor` | 2.0 GB | Cursor's editor state; excluded, see Cursor |
  | `~/.local/share/opencode` | under 1 MB | opencode |
  | `~/.cache/opencode` | 10.5 MB | opencode |
  | `~/.config/opencode` | 61 MB, mostly `node_modules` | opencode |

  The coding-agent total, without the two excluded rows, is about 11 GB.

## Method

- On 2026-10-28, listed the entries of each harness folder under `~` on the reference machine, hidden entries included. Sizes are allocated bytes from `du -sk`.
- Only names, counts, key names and value shapes were recorded here. No transcript, prompt, project path or session id is copied into this note.
- **Claude Code:**
  - read every `~/.claude/sessions/*.json`;
  - compared each `pid` with the file name, with `ps -o lstart=` in UTC and in local time, and with `startedAt`.
- **Codex:**
  - listed `thread-writer-locks/`;
  - found which processes hold each file open, with `lsof` on that folder only;
  - read the naming of the `sessions/` files.
- **Cursor and opencode:** listed their folders and looked for a per-session file with a pid or a lock.
- No harness was started, stopped or signalled, and no lock was taken.
- **Cost:** `du -sk` over `~/.claude`, `~/.codex`, `~/.cursor` and `~/.cache/codex-runtimes` took 5.1 s wall clock for about 124,000 entries. `Application Support/Claude` adds about 80,000 more.

## Findings

### Claude Code

- **Data.**
  - **Transcripts.** `~/.claude/projects/<project>/<session-id>.jsonl` holds them, one file per session. Subagent transcripts sit in `projects/<project>/<session-id>/`.
  - **Other folders under `~/.claude`.** `file-history`, `paste-cache`, `shell-snapshots`, `debug`, `todos`, `plugins`, `skills` and `sessions`.
  - **Where DDC-003 stands.** It reports the transcripts and never removes them, and it never claims their `memory/`.
  - **The Claude desktop app.** It keeps its own state in `~/Library/Application Support/Claude`. Only `claude-code/` in that folder is Claude Code: the binaries the desktop app starts coding sessions from. The rest is the chat app's state.
- **Session records.** Each `~/.claude/sessions/<pid>.json` holds these keys:
  - `cwd`, `entrypoint` (seen: `cli`, `claude-desktop`, `sdk-ts`), `kind`, `messagingSocketPath`, `name`;
  - `pid`, `procStart`, `sessionId`, `startedAt`, `status` (seen: `idle`, `waiting`, `busy`; absent on the `sdk-ts` session), `updatedAt`, `version`;
  - and, on some records, `pidDomain` (seen: `darwin`), `hostSessionId`, `bridgeSessionId`, `peerProtocol`, `peerFeatures`, `waitingFor`, `statusUpdatedAt`, `nameSince`, `nameSource`.

  The file's name is its `pid`. Beside each record is a per-session key file, named `<pid>.<hex>.key`. It is not session metadata, and a reader should neither read it nor report it, nor the `messagingSocketPath`.
- **`procStart`** is a 24-character ctime-style string (`%a %b %e %H:%M:%S %Y`). It is the process start time in UTC: 13 of 13 equal `ps -o lstart=` with `TZ=UTC`, and 0 of 13 in local time.
- **`startedAt`** is never earlier than the process start (13 of 13). So the zone-free test is this: the process with that pid has the current user's uid and a start time no later than `startedAt`. One `proc_pidinfo(PROC_PIDTBSDINFO)` call gives the uid and the start time, and `sys::procs::bsd_info` already makes that call. `startedAt` minus the process start was under 5 s for 11 records and under 60 s for 2. The records carry no resume marker, so whether any of the 13 was resumed is not known, nor whether `--resume` keeps the old `startedAt`; and `procStart` is the second proof for that case.
- **Liveness, observed.** 13 of 13 records were live: 9 `idle`, 2 `waiting`, 1 `busy`, and 1 without a status.
  - No stale record was seen. A record left behind by a crash is a case to handle, not one observed.
  - `sessionId` points at the transcript in 12 of 13 live sessions. One live session had no transcript yet.

### Codex

- **Data.**
  - **Under `~/.codex`:**
    - `sessions/`: the transcripts, by date;
    - `log/`, `worktrees/` and `history.jsonl`;
    - several SQLite stores: `state_5`, `logs_2`, `goals_1`, `thread_history_1`, `queue_1`;
    - `shell_snapshots`, `plugins`, `skills`, and `packages/`, where the app-server daemon lives.
  - **Elsewhere.** `~/.cache/codex-runtimes` holds 1.6 GB of runtimes.
- **Sessions.** Each thread has one transcript: `sessions/YYYY/MM/DD/rollout-<time>-<thread-id>.jsonl`. The reference machine had 34 rollouts and 7 open threads.
- **Liveness.** A thread is open while any process of the current user holds `thread-writer-locks/<thread-id>.lock` open.
  - Observed: 7 lock files, all held, by 2 processes. One was the `codex` CLI and one the app-server daemon under `~/.codex/packages/`. Restricting the check to processes named `codex` would miss the daemon.
  - **Lock files to ignore.** A hidden `.coordination.lock` sits in the same folder and was held by no process. Only `<thread-id>.lock` names a thread.
  - **No date signal.** The lock files are empty, and their dates do not show liveness.
  - **How to probe.** Read the open files of the current user's processes with `proc_pidinfo`, and match the lock by `(dev, ino)` rather than by path string: a home folder can be reached through a symlink, and a string prefix is not proof. The kernel gives the pair with each open file (`proc_pidfdinfo` with `PROC_PIDFDVNODEPATHINFO`), so no open path needs a `stat`. Taking the lock to test it is ruled out: it could make Codex wait or fail.
  - **Other Codex binaries.** Codex also ships inside the ChatGPT desktop app, in `Resources/codex` and in `Frameworks/Codex` helpers. On the reference machine those held no lock. Whether they write to `~/.codex` is not known.

### Cursor

- **Data.**
  - `~/.cursor` holds `projects/` (56 folders), `worktrees/`, `extensions`, `plugins` and `ai-tracking`.
  - `~/Library/Application Support/Cursor` (2.0 GB) is the editor's state, as VS Code keeps it, agent chats included in its state databases. It is editor data, not a harness folder, and it is excluded from the totals above.
- **Sessions.** No per-session file with a pid or a lock was found under `~/.cursor`.
- **Liveness.** Only whether the Cursor app runs.

### opencode

- **Data.**
  - `~/.local/share/opencode` holds `opencode.db` (SQLite, with WAL), `storage`, `snapshot`, `repos` and `log`.
  - `~/.cache/opencode` holds 10.5 MB.
  - `~/.config/opencode` holds 61 MB, mostly `node_modules`.
- **Sessions.** They are in `opencode.db`, whose schema was not read.
- **Liveness.** No lock or pid file was found, so the only signal is whether an `opencode` process runs.

## Comparison

| Harness | Per-session record | Live signal | Cost to read |
|--------|------|------|------|
| Claude Code | `sessions/<pid>.json` | the process with that pid has the user's uid and started no later than `startedAt`, or its UTC start equals `procStart` | one `proc_pidinfo` per record |
| Codex | `thread-writer-locks/<id>.lock` and a `rollout-…-<id>.jsonl` | the lock file is open in a process of the user | `proc_pidinfo` over the user's processes' open files |
| Cursor | none found | the app runs | process list |
| opencode | rows in `opencode.db` (unread) | the process runs | process list |

## Conclusion

- **A Coding agents block is possible.** The Whole disk view can show it read-only. Per harness, it lists:
  - its folders and their sizes;
  - for Claude Code, its live sessions;
  - for Codex, its open threads;
  - for Cursor and opencode, whether they run.
- **Liveness is cheap; sizes are not.** Liveness costs a few hundred milliseconds: a sweep of the open files of every process of the user, about 700 here. Sizes cost a walk of about 200,000 entries, about 5 s here.
- **Nothing intrusive is needed.** Every signal is read without taking a lock, without opening a database, and without reading a transcript's content.
- **The CLI computes it, the app only draws.** The CLI is the right place for it, as for the rest of the whole-disk report. The size-only visitor of `ddc disk` can be reused: its tag type and its hard-link dedup work for any set of roots.
- **Cleaning stays with DDC-003.** This block only names and sizes. A live session should later make the DDC-003 rules hold back that session's data; that is a question for its ADR, not for this view.

## Open questions

- Does a resumed Claude Code session (`--resume`, `--continue`) keep the original `startedAt`? If it does, only `procStart` proves it live.
- Does Claude Code remove `sessions/<pid>.json` on every exit, including a crash? To test it, kill a throwaway session's process with SIGKILL and check for the file.
- Does the ChatGPT app's embedded Codex write to `~/.codex`? If it does, "Codex runs" is true whenever ChatGPT is open, and the label should name the binary.
- Where does Cursor keep agent sessions inside its state databases, and does anything there mark a live one?

## References

- The owner's comment on artboard 13 of the design canvas, thread of 2026-10-28.
- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]: the first measurement of `~/.claude` and the worktrees.
- The ADR "Agent footprint: worktrees, harness data and bounded consent for agents" (DDC-003, branch `ddc-003-agent-footprint`): which harness data may be cleaned.

---

*Last updated: 2026-10-28*
