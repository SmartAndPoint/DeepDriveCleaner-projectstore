---
type: research
slug: "coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session"
title: "Coding-agent harnesses on macOS: where their data lives and how to tell a live session"
status: final
date: 2026-10-28
authors: ["Evgenii Konev"]
tags: [agents, whole-disk, liveness]
review_status: reviewed
reviewed_at: 2026-11-02
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

## On Linux (addendum, 2026-11-02)

This addendum answers, for Linux, the questions the note answered for macOS. It is the research item of the story "Coding agents on Linux: ddc disk agents" (DDC-010). It covers Claude Code, Codex CLI, Cursor's agent CLI and opencode.

Each claim carries a mark:
- **[src]**: read in a primary source (official docs, the harness's own source or install script). The URL is under References.
- **[meas]**: measured, on one Linux machine (ARM64) or on the reference Mac.
- **[inf]**: inferred. Not checked.

### Method

- **Sources.** Read on 2026-11-02:
  - Claude Code's docs, its changelog, and its installer script `https://claude.ai/install.sh`. The script was read, not run.
  - Codex's repository at commit `a09812981de7`.
  - opencode's repository at commit `ff90ed91607c`, and its installer `https://opencode.ai/install`.
  - Cursor's installer `https://cursor.com/install` (it installs build `2026.10.22-27dcf5f`), and Cursor's CLI configuration docs.
- **Linux measurement.** On one Linux machine (ARM64), only a fixed list of 13 paths was checked: whether each exists, and its `du -sk`. The list:
  - `~/.claude`, `~/.claude/projects`, `~/.local/share/claude`, `~/.local/share/claude/versions`, `~/.local/bin/claude`;
  - `~/.codex`, `~/.codex/sessions`, `~/.cache/codex-runtimes`;
  - `~/.cursor`, `~/.local/share/cursor-agent`;
  - `~/.local/share/opencode`, `~/.config/opencode`, `~/.cache/opencode`.

  On top of that: the names inside `versions`, `readlink` of the launcher, and the first line of `claude --version`. Nothing else was listed or read. A second Linux machine had none of the 13 paths.
- **macOS cross-check.** On the reference Mac, the same `versions` folder was listed. Also read there:
  - which build each running `claude` process runs (`lsof -d txt`);
  - the key names of the files in `~/.local/state/claude/locks` (not their values).
- No harness was started, stopped or updated.

### Paths on Linux

`~` is the home folder. `$XDG_DATA_HOME` defaults to `~/.local/share`, `$XDG_CACHE_HOME` to `~/.cache`, `$XDG_CONFIG_HOME` to `~/.config`, and `$XDG_STATE_HOME` to `~/.local/state`.

| Harness | Kind | Linux path | Moved by | Mark |
|---|---|---|---|---|
| Claude Code | config, transcripts, session records | `~/.claude` (`projects/`, `sessions/`, and the rest as on macOS), and `~/.claude.json` | `CLAUDE_CONFIG_DIR` | [src] |
| Claude Code | builds | `~/.local/share/claude/versions/<version>`, one file per build | not known | [src] |
| Claude Code | launcher | `~/.local/bin/claude`, a symlink to a build | not known | [src] |
| Claude Code | per-build locks | `~/.local/state/claude/locks/<version>.lock` | not known | seen on the Mac [meas] |
| Claude Code | download staging | `~/.cache/claude/staging` | not known | [src] (changelog 2.1.277) |
| Claude Code | installer download | `~/.claude/downloads`; install.sh removes the downloaded file, not the folder | no variable | [src] (install.sh) |
| Claude Code | temporary files | `claude-<uid>` under `$TMPDIR`, else under `/tmp` (Node's `os.tmpdir()`) | `CLAUDE_CODE_TMPDIR`, which gets `claude-<uid>` appended | [src] |
| Claude Code | worktrees | `<repo>/.claude/worktrees/<name>`, inside each repository | a `WorktreeCreate` hook | [src] |
| Codex | everything else | `~/.codex`: `sessions/`, `log/`, `history.jsonl`, `thread-writer-locks/`, the SQLite stores | `CODEX_HOME`; SQLite also `CODEX_SQLITE_HOME` | [src] |
| Codex | builds (standalone installer) | `~/.codex/packages/standalone/releases/<version>-<target>`, with a `current` link; the daemon's in `packages/app-server-daemon/` | `CODEX_HOME` | [src] |
| Codex | launcher | `~/.local/bin/codex` | `CODEX_INSTALL_DIR` (installer only) | [src] |
| Codex | runtimes | `$XDG_CACHE_HOME/codex-runtimes` | `XDG_CACHE_HOME`, on Linux only | [src] |
| Codex | temporary files | `~/.codex/tmp/arg0-linux` (helper links), `~/.codex/.tmp` (plugin staging) | `CODEX_HOME` | [src] |
| Codex | worktrees | `~/.codex/worktrees` | `CODEX_HOME`, or the `desktop.git-worktree-root` setting | [src] |
| Cursor agent | builds | `~/.local/share/cursor-agent/versions/<build>` | nothing: the installer writes `$HOME/.local/share` itself | [src] |
| Cursor agent | launchers | `~/.local/bin/agent` and `~/.local/bin/cursor-agent`, both symlinks to `versions/<build>/cursor-agent` | nothing | [src] |
| Cursor agent | config | `~/.cursor/cli-config.json` | `CURSOR_CONFIG_DIR`; on Linux also `XDG_CONFIG_HOME` (`$XDG_CONFIG_HOME/cursor/cli-config.json`) | [src] |
| Cursor agent | transcripts, chats | `~/.cursor/projects`, `~/.cursor/chats` (as on macOS) | not known | [inf] |
| opencode | data, sessions | `$XDG_DATA_HOME/opencode`: `opencode.db`, `storage`, `snapshot`, `log`, `repos` | `XDG_DATA_HOME` | [src] |
| opencode | worktrees | `$XDG_DATA_HOME/opencode/worktree/<project>` | `XDG_DATA_HOME` | [src] |
| opencode | cache, downloaded tools | `$XDG_CACHE_HOME/opencode` (tools in `bin/`) | `XDG_CACHE_HOME` | [src] |
| opencode | config | `$XDG_CONFIG_HOME/opencode` | `XDG_CONFIG_HOME`, `OPENCODE_CONFIG_DIR` | [src] |
| opencode | state, locks | `$XDG_STATE_HOME/opencode` (`locks/`) | `XDG_STATE_HOME` | [src] |
| opencode | temporary files | `opencode` under `$TMPDIR`, else under `/tmp` | `TMPDIR` | [src] |
| opencode | binary | `~/.opencode/bin/opencode`, one file, replaced in place | nothing | [src] |

### Measured on one Linux machine (ARM64)

| Path | Size |
|---|---|
| `~/.claude` | 8 MB (`projects/` 2 MB) |
| `~/.local/share/claude` | 473 MB, nearly all in `versions/` |
| `~/.local/share/claude/versions` | two builds, `2.1.123` and `2.1.126` |
| `~/.local/bin/claude` | an absolute symlink to `~/.local/share/claude/versions/2.1.126` |
| `claude --version` | `2.1.126 (Claude Code)` |
| `.codex`, `.cache/codex-runtimes`, `.cursor`, `.local/share/cursor-agent`, the three opencode folders | absent |

So only Claude Code could be measured on Linux. The other three harnesses rest on their sources alone.

### Findings

**Claude Code's builds.**
- **Same path on both OSes.** The native installer keeps its builds in `~/.local/share/claude/versions/<version>` on macOS and Linux alike, and makes `~/.local/bin/claude` a symlink into it [src: setup docs, "Auto-updates"]. On the reference Mac, `versions` held four builds (1.1 GB), and the launcher pointed to the newest one [meas].
- **The XDG variables: not known.** The default paths fit the XDG layout (`~/.local/share`, `~/.local/state`, `~/.cache`), on macOS too, but no public source says whether `XDG_DATA_HOME`, `XDG_CACHE_HOME` or `XDG_STATE_HOME` move them [inf]. `CLAUDE_CONFIG_DIR` moves `~/.claude`, not the builds [src].
- **The installer prunes old builds itself.** The docs say: "Besides the version the launcher points to and any version a session on the same machine is running, it keeps the two newest versions and deletes the rest" [src: setup docs, "Install on network storage"]. How it orders "newest" (version or mtime), and when the cleanup runs, are not documented [inf].
- **Both measurements match that rule** [meas]:
  - On Linux: `2.1.126` is the launcher's target, and `2.1.123` is one of the "two newest". Nothing was due for deletion.
  - On the Mac: `2.1.316` is the launcher's target, and `2.1.314` is run by 6 live processes and holds a lock. `2.1.311` and `2.1.312` are the two newest of the rest. So the Mac keeps up to four builds, about 1.1 GB. So the installer's floor reads as "protected builds + 2", not 2 [inf]: the measurement fits it only if a cleanup ran after the newest update.
- **With a custom launcher, nothing is pruned.** If `~/.local/bin/claude` is the user's own script or symlink, "Claude Code also keeps every installed version on disk because it can't tell which version the launcher needs" [src].
- **A missing build comes back only on demand** [inf]: `claude install <version>` installs a given version [src: CLI reference], and nothing runs a build that is neither the launcher's target nor live. So an old build that is gone is fetched again by that command, or never.

  Removing the launcher's target would break `claude` until the installer runs again [inf]. A ddc rule must never do that.
- **The per-build lock.** On the Mac, `~/.local/state/claude/locks/` held one `<version>.lock` per build in use. Each held the keys `pid`, `version`, `execPath` and `acquiredAt`. Each `pid` was alive and ran that `execPath` [meas]. How the installer reads these locks is not documented [inf]; a build may run with no lock file, so a missing lock proves nothing.

**Claude Code: Linux-only and macOS-only.**
- **macOS only:**
  - `~/Library/Application Support/Claude/claude-code/<version>`: the builds the Claude desktop app runs. The reference Mac had two, 0.5 GB [meas].
  - `~/.local/share/claude/ClaudeCode.app`: an app bundle with one `claude` binary, 0.2 GB, a month older than the newest build [meas]. What it is for is not known. It is not in `versions`, so the installer's pruning does not cover it.
- **Linux:** the Claude desktop app exists only as a beta `.deb` for Debian and Ubuntu (`claude-desktop`). Its updates come through apt [src: "Claude Desktop on Linux"]. Where it keeps the Claude Code builds it runs is not documented. `~/.config/Claude/claude-code` is a guess from Electron's defaults [inf]. Not measured.
- **Temporary folder.** It differs by OS: `/tmp` on macOS, `os.tmpdir()` on Linux [src]. The docs also confirm what `Env::claude_temp_base` assumed: Claude Code appends `claude-<uid>` to `CLAUDE_CODE_TMPDIR` on Unix [src].

**Codex.**
- **Paths.** All under `CODEX_HOME`, default `~/.codex` [src: `utils/home-dir`]. Codex does not read the XDG variables, except one: the runtimes folder is `dirs::cache_dir()` on Linux, which honours `XDG_CACHE_HOME`. On macOS it is always `~/.cache` [src: `core-plugins/src/marketplace_policy.rs`]. Whether anything writes `codex-runtimes` on Linux, where there is no Codex desktop app, is not known.
- **Old builds.** The standalone installer prunes old releases itself. It keeps the selected release and the previous one, and any release a process uses. On Linux it checks `/proc/*/exe`, and it refuses to prune if a `codex*` process cannot be inspected [src: `scripts/install/install.sh`, `prune_standalone_releases`]. No ddc rule is needed for them.
- **Thread locks.** They work on Linux as on macOS. The lock is `File::try_lock` on `thread-writer-locks/<thread-id>.lock`, and `.coordination.lock` sits beside them [src: `rollout/src/writer_lock.rs`]. ddc already reads open files by `(dev, ino)` from `/proc/<pid>/fd` (`platform/linux/procs.rs`).

**Cursor's agent CLI.**
- **Builds.** The installer writes builds to `$HOME/.local/share/cursor-agent/versions/<build>`. It ignores `XDG_DATA_HOME`, and it relinks `~/.local/bin/agent` and `~/.local/bin/cursor-agent` [src]. Whether the CLI's own updater prunes old builds is not known.
- **Config.** On Linux, `XDG_CONFIG_HOME` moves the config file to `$XDG_CONFIG_HOME/cursor/cli-config.json`. That is a Linux-only difference [src]. Whether it also moves `~/.cursor/projects` or `~/.cursor/chats` is not documented.

**opencode.**
- **Paths.** opencode roots every folder in the XDG variables, on every OS. An empty variable falls back to the default [src: `packages/util/src/global-roots.ts`].
- **Not in `HARNESSES` today:**
  - `$XDG_STATE_HOME/opencode`, with its `locks/`. Each lock is a folder with a `meta.json` (with a `pid`) and a `heartbeat` [src: `flock.ts`]. What it locks, and whether it marks a live session, was not read.
  - `$XDG_DATA_HOME/opencode/worktree`: the worktrees, inside the data folder that `HARNESSES` already lists [src].

**How a live session is told on Linux.**
- **Process names and the name ddc gives them.** ddc names a Linux process after the file name of its resolved `/proc/<pid>/exe` (`describe` in `platform/linux/procs.rs`).
  - For Claude Code's native build, `/proc/<pid>/exe` resolves through the launcher to `versions/2.1.126`. So ddc names the process `2.1.126`, not `claude` [inf, from the measured `readlink` and the code].
  - `running::classify` matches `claude` by name. As written, it would find no Claude Code process on Linux.
  - Fix: match an exe under `$XDG_DATA_HOME/claude/versions/`, as `Installs::cursor_agent` already does for Cursor. `cmdline`'s first argument (`claude`) is a second signal. `/proc/<pid>/comm` is the name used in `execve`, so it is probably `claude` too [inf].
- **Codex's standalone CLI.**
  - It runs from `~/.codex/packages/standalone/releases/…/bin/codex` [src]. `classify` calls every exe under `~/.codex/packages` an app-server.
  - So a standalone CLI would be labelled `AppServer`, on both OSes [inf, from the code].
  - The daemon's own folder is `packages/app-server-daemon/` [src]. Excluding `packages/standalone` from the app-server match fixes it, and keeps the legacy `packages/<version>/bin/node` layout covered.
- **Claude Code's session records** `~/.claude/sessions/<pid>.json` were not read on Linux. That was outside the fixed path list.
  - `claude::state` treats a record as local only when `pidDomain` is absent or `darwin`. Records written on Linux most likely say `linux` [inf]. As written, every Linux session would read as `unknown`.
  - Fix: compare `pidDomain` with the OS ddc runs on.
  - The form of `procStart` on Linux is not known. The `startedAt` test does not need it.
- **Start time and cwd.** `/proc/<pid>/stat` gives the start time, and ddc already reads it (`process_start`). `/proc/<pid>/cwd` could check a record's `cwd` as a second proof, but it is not needed.
- **Under WSL or in a container,** the harness and ddc may see different pid namespaces. This is the story's Technical Note, and it stands.

### Proposed per-OS `HARNESSES`

`HARNESSES` becomes a function of the platform, in the same shape as the catalog's `PerOs`. Folders are named from a root, so that a later "Cache roots moved by the environment" story can resolve the XDG roots in one place:

```rust
pub enum Root { Home, XdgData, XdgCache, XdgConfig, XdgState }
pub struct FolderDef { pub root: Root, pub path: &'static str, pub nested: bool }

pub const fn harnesses(p: Platform) -> &'static [HarnessDef] { /* per OS */ }
```

| Harness | macOS | Linux |
|---|---|---|
| Claude Code | `.claude`, nested `.claude/projects`; XdgData `claude` (new; builds); `Library/Application Support/Claude/claude-code` | `.claude`, nested `.claude/projects`; XdgData `claude` (new); the desktop app's folder only once measured |
| Codex | `.codex`, nested `sessions`, nested `worktrees`; `.cache/codex-runtimes` (Home, not XdgCache) | the same, with `codex-runtimes` under XdgCache |
| Cursor | `.cursor`; `.local/share/cursor-agent` (new; Home, not XdgData) | the same |
| opencode | XdgData `opencode`, XdgCache `opencode`, XdgConfig `opencode`; XdgState `opencode` (new) | the same |

Notes:
- **Claude Code.** `~/.local/share/claude` goes in as a top folder on both OSes. On the Mac it also holds `ClaudeCode.app`, which the size then counts. The temporary folder stays where it is now, as `temp_root`.
- **XDG roots.** Until ddc resolves them, each XDG root is its default under the home folder. A harness that honours a moved root will then not be found. That is a missing row, not a wrong one.
- **Codex's runtimes.** Under XdgCache on Linux only, because Codex itself honours `XDG_CACHE_HOME` only there.
- **Cursor's builds.** Home-rooted on both OSes, because the installer ignores `XDG_DATA_HOME`.
- **`running::Installs`.** It gains `claude_versions` (`~/.local/share/claude/versions`). The Codex app-server match excludes `.codex/packages/standalone`, rather than narrowing to `app-server-daemon`: the legacy layout `~/.codex/packages/<version>/bin/node` stays covered (test `codex_is_a_cli_or_its_app_server`).
- **Windows.** Claude Code's docs give `%USERPROFILE%\.local\share\claude\versions` [src]; the catalog's per-OS paths have a Windows place for it.
- **Package installs.** On Linux, Claude Code can also come from apt, dnf or apk [src]. Such an install has no `versions` folder and no pruning; the rule's precondition leaves it out.

### Recommendation: the old-builds rule

- **Worth it, harness-neutral.** Claude Code already prunes to the launcher's target, the builds in use and two more [src: "keeps the two newest versions"; whether "two newest" counts the protected ones is not stated, so "protected + 2" is a reading, inf]. A Claude Code rule alone finds little: at most a build or two, about 0.5 GB, and only with an active native updater. Cursor's agent keeps its builds in `~/.local/share/cursor-agent/versions` with no documented pruning [src: installer], so there the growth has no bound. One rule shape for both is worth building; Claude Code's half may often be empty.
- **Matcher.** `KnownChildren` of `~/.local/share/claude/versions` (macOS, Linux; the Windows path above) and of Cursor's `versions`. One finding per build. Any name that is not a bare version is skipped: an install may be writing a temporary file there. "Newest" compares versions numerically (`2.1.10` after `2.1.9`), never as strings.
- **Per harness.** Each harness has its own build-name form and its own "newest":
  - Claude Code: a build is a file named by a bare version (`2.1.126`); newest by numeric version.
  - Cursor's agent: a build is a folder named `<date>-<hash>` (`2026.10.22-27dcf5f`); newest by the date prefix. Its installer leaves `.tmp-<build>-<time>` entries beside them [src: installer], which the bare-name check skips. Its launchers `~/.local/bin/agent` and `~/.local/bin/cursor-agent` point to `versions/<build>/cursor-agent` [src]; a process counts as running a build when its `exe` lies under `versions/<build>/`, as `Installs::cursor_agent` matches it.
- **Precondition.** For Claude Code, `~/.local/bin/claude` is a symlink whose target resolves inside `versions`; for Cursor, one of its launchers resolves inside its `versions`. With no launcher or a custom one, Claude Code keeps every build itself because it cannot tell which is needed [src], and the rule finds nothing.
- **Keep set.** Never findings:
  1. the launcher's target;
  2. every build a process of the user runs (`/proc/<pid>/exe` on Linux; on macOS this needs a new platform call, `proc_pidpath`, since `ProcInfo.exe` there is the unresolved launcher);
  3. while any readable process of another user is named `claude` (`/proc/<pid>/comm`), every build: another user's `exe` cannot be read without root, as Codex's installer limits its own check by name;
  4. every build whose `locks/<version>.lock` names a live pid;
  5. the newest build.

  If the process table is hidden (`hidepid`) or the home is on a network filesystem that other machines may run builds from, keep every build, as Codex's installer does. This reads only the user's own processes in full, as the safety ADR's amendment of 2026-10-31 says for Linux. The plan's in-use check runs again at clean time.
- **Class: `review`, removal Trash.** A build does not come back by itself: only `claude install <version>`, run by hand, fetches it again [src: CLI reference]. That is not the `safe` class ("cache that refills itself"), and it matches the catalog's precedent, `agent.codex-runtimes` (`review`, Trash).
- **Desktop app builds.** `Library/Application Support/Claude/claude-code/<version>` gets no rule here. Who prunes it, and which build the app needs, is not known.
- **Liveness in containers.** A `~/.claude` bind-mounted into a devcontainer holds session records whose pids are not this machine's. A fresh record with no matching process should read as `unknown`, not `gone`.
- **Temporary folder.** Node's `os.tmpdir()` on Linux reads `TMPDIR`, then `TMP` and `TEMP`; `Env::claude_temp_base` reads only `TMPDIR`. A follow-up for the per-session footprint story.

### Open questions

- **Claude Code on Linux:**
  - Does a `sessions/<pid>.json` record carry `pidDomain: "linux"`?
  - What is the form of its `procStart`?
  - Is `/proc/<pid>/comm` `claude`?

  To answer them, read one record's key names and the `comm` of one live session, with the owner's go-ahead for those two paths.
- Do the XDG variables move Claude Code's builds, locks and staging? The docs say nothing; a test with `XDG_DATA_HOME` set would tell.
- Is a per-version lock written for every live session? A live session without one would need `/proc/*/exe` alone.
- Where does the Linux desktop app (`claude-desktop`) keep its Claude Code builds and its worktrees?
- What is `~/.local/share/claude/ClaudeCode.app` on macOS for? Who updates or removes it?
- Does Cursor's agent CLI prune `cursor-agent/versions` when it updates itself? Do `CURSOR_CONFIG_DIR` or `XDG_CONFIG_HOME` move `~/.cursor/projects` and `~/.cursor/chats`?
- What do opencode's `$XDG_STATE_HOME/opencode/locks` lock? Is one of them per session?
- Does anything write `codex-runtimes` on Linux?

### References

Claude Code:
- Setup, "Auto-updates", "Install on network storage", "Uninstall": https://code.claude.com/docs/en/setup
- Environment variables (`CLAUDE_CONFIG_DIR`, `CLAUDE_CODE_TMPDIR`, `DISABLE_UPDATES`): https://code.claude.com/docs/en/env-vars
- Worktrees: https://code.claude.com/docs/en/worktrees
- Claude Desktop on Linux (beta): https://code.claude.com/docs/en/desktop-linux
- Installer script, read 2026-11-02: https://claude.ai/install.sh
- Changelog: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
  - 2.1.207: the updater no longer overwrites a custom launcher;
  - 2.1.243: "an auto-update cleaned up the version it was set up with";
  - 2.1.257: background sessions "piling up", and `cc-daemon-*` files in the temporary folder;
  - 2.1.277: staged downloads left in `~/.cache/claude/staging`.

Codex (https://github.com/openai/codex, commit `a09812981de7`):
- `codex-rs/utils/home-dir/src/lib.rs`
- `codex-rs/core-plugins/src/marketplace_policy.rs`
- `codex-rs/worktree/src/settings.rs`
- `codex-rs/rollout/src/writer_lock.rs`
- `codex-rs/config/src/config_toml.rs`
- `codex-rs/core/src/config/permissions.rs`
- `codex-rs/install-context/src/lib.rs`
- `scripts/install/install.sh`

opencode (https://github.com/sst/opencode, commit `ff90ed91607c`):
- `packages/util/src/global.ts`
- `packages/util/src/global-roots.ts`
- `packages/util/src/flock.ts`
- `packages/core/src/worktree/strategies.ts`
- Installer: https://opencode.ai/install

Cursor:
- Installer, read 2026-11-02: https://cursor.com/install
- CLI configuration: https://cursor.com/docs/cli/reference/configuration

ddc code read: `crates/ddc-core/src/agents/{mod,running,claude}.rs`, `crates/ddc-core/src/platform/linux/procs.rs`, `crates/ddc-core/src/env.rs`, `crates/ddc-core/src/rules/catalog/shared/agents.rs`.

---

*Last updated: 2026-11-02*
