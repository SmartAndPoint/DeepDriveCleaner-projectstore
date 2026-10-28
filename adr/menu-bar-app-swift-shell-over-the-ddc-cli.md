---
type: adr
id: "menu-bar-app-swift-shell-over-the-ddc-cli"
title: "Menu bar app: Swift shell over the ddc CLI"
status: proposed
date: 2026-10-18
authors: ["Evgenii Konev"]
tags: [app, macos, swift, architecture]
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
supersedes: null
superseded_by: null
review_status: reviewed
reviewed_at: 2026-10-28
---

# Menu bar app: Swift shell over the ddc CLI

| Field | Value |
|---|---|
| **Status** | proposed |
| **Date** | 2026-10-18 |
| **Authors** | Evgenii Konev |

---

## Context

DeepDriveCleaner 0.1.0 is a terminal tool distributed as a signed, notarized Homebrew cask (epic DDC-001). The owner wants the next version to be a menu bar app, in the spirit of GrammarHelper:

- one click on the status item shows what the latest scan found;
- the user reviews by group or by item, then cleans;
- settings choose which folders are scanned and which kinds of objects may be cleaned;
- small notifications say when a lot can be freed.

The ADR "Rust core: library + CLI with JSON output" recorded a direction, not a decision: SwiftUI first runs `ddc --json`, and links `ddc-core` through UniFFI later if needed. It asked for a separate ADR when the app starts. This is that ADR.

Facts that constrain the choice:

1. **Full Disk Access is the product.** Without it a scan cannot read much of `~/Library`.
   - TCC keys a bare executable by its absolute path as well as its signature. Homebrew puts every cask version in a new `Caskroom/<token>/<version>/` directory, so the 0.1.0 CLI loses its grant on every `brew upgrade`.
   - An app bundle is keyed by its bundle id instead.
   - A process an app spawns is attributed to the app, its responsible process, so the app's grant governs it. For the same reason, `ddc` run from Terminal is governed by Terminal's grant, not by its own.
   - A process launchd starts is its own responsible process. An app that LaunchServices launches (for example through `open`) is its own responsible process too.
2. **The CLI cannot do what the app needs yet.**
   - `ddc clean` always rescans and is interactive: it needs a TTY, or `--yes` limited to items selected by default in the given classes.
   - It has no `--json` output, no explicit item list, no machine-readable progress and no cancellation.
   - Today's exit codes: refusals exit 2 (run as root, `--yes` with `review`, no terminal; `crates/ddc-cli/src/cmd/clean.rs`), and a held lock exits 1. `scan --background` exits 0 when the lock is held and logs "skipped" (`cmd/background.rs`).
   - Settings exist only as command-line flags.
   - Free space and the low-disk check read only the home volume (`statfs(home)` in `cmd/background.rs` and `exec/mod.rs`).
3. **Notifications.** The CLI sends them by running `/usr/bin/osascript display notification`. They have no actions and are attributed to Script Editor.
4. **GrammarHelper's conventions** (its `CLAUDE.md`):
   - an AppKit `NSStatusItem` with an `NSMenu`;
   - `LSUIElement`;
   - a login item through `SMAppService.mainApp`;
   - settings in `UserDefaults`;
   - a log file under `~/Library/Logs`;
   - Developer ID signing, hardened runtime, notarization and stapling;
   - no sandbox, no Sparkle, updates through `brew upgrade`;
   - a bundle id that never changes, because TCC grants depend on it;
   - macOS 26 as the minimum.

   Its login-item toggle is a `UserDefaults` bool, not `SMAppService.status`, so it can drift. Its bundle id `smartandpoint.com.GrammarHelper` is not reverse-DNS.
5. **A Homebrew cask's `uninstall quit:` also runs on `brew upgrade` and `brew reinstall`, and nothing relaunches the app.** A scheduler that lives only inside the running app therefore stops after every upgrade.
6. **Identifiers.** The 0.1.0 CLI is signed with the identifier `com.smartandpoint.deepdrivecleaner` (`scripts/build-dist.sh`). That is the natural bundle id for the app.
7. **The owner may move to a SmartAndPoint organization Developer ID team.** A team change changes the designated requirement, and every TCC grant is lost once.

## Decision

### 1. A Swift app that runs the bundled CLI

`DeepDriveCleaner.app` is a Swift app. It holds no cleaning logic. It runs the `deepdrivecleaner` binary that ships inside its bundle at `Contents/Helpers/deepdrivecleaner`, not `Contents/MacOS`: on a case-insensitive volume `deepdrivecleaner` and the app's main executable `DeepDriveCleaner` would be the same file. The app reads JSON from it:

- the saved report `<data-dir>/last-scan.json`, read directly, for an instant popover;
- the output of `scan` and `clean` on stdout;
- progress events on stderr.

The app and the CLI ship in one bundle and one version, so the app always reads the `schema_version` it was built for. A mismatch is a bug, shown as "the report could not be read", never decoded loosely.

All CLI runs go through one in-app runner that serialises them. A scheduled scan, Scan now and a clean never overlap. A request that arrives while one runs is coalesced, never run twice in parallel.

Amended 2026-10-28, story "Whole disk in the app" (epic DDC-002): every run but `disk` goes through one queue and never overlaps another. Two runs that take the ddc lock would make one fail as busy, and `rules`, `notify` and `schedule`, which take no lock, stay queued too. `disk` is the one that takes no lock and writes nothing another command reads: today only `ddc disk` writes `last-disk.json`. The runner therefore runs it in a lane of its own, beside the queue: it never waits for a scan or a clean, and a scan or a clean never waits for it. A whole-disk measurement beside a scheduled scan reads the disk at the same time; that is accepted. The lane holds one `disk` run at a time, and a second request joins the running one, as Scan now joins a queued scan. Stop on the Whole disk view cancels that run only, by its ticket. On quit the app leaves a running `disk` alone: SIGTERM would replace the last full report with a partial one, and the CLI saves its report before it prints, so an orphaned run still finishes and writes it. The app never starts `disk` on its own: the scheduled path stays the home folder's scan. `ddc disk free` (story "Whole disk: free space with the owning tool") takes the lock and stays in the queue. It will also edit `last-disk.json` (it marks the area it freed as stale), so the two never run at once in the app: each button is disabled while the other's run is queued or running.

Amended 2026-10-28, story "Coding agents in the Whole disk view" (epic DDC-002): the lane above becomes a rule. A kind gets a lane of its own, beside the queue, when three things hold:
- the user waits on it for seconds or minutes;
- it takes no lock;
- whatever it writes, the app keeps apart from any run that reads or edits the same file.

`disk` meets the third through the button rule above: it writes `last-disk.json`, and the app never runs it beside `disk free`. Today there are two such kinds: `disk` and `disk agents`. `rules` stays queued: it finishes in milliseconds, so a lane would gain nothing.

How the lanes behave:
- Each lane holds one running process. The model that owns the view joins a second request to its run in progress, so each run has one ticket, and a Stop, where the view has one, cancels it. Under it the runner joins only a request with the same arguments, and a request with other arguments waits for the running one. So a request for the sizes report never receives the `--no-sizes` one. For `disk` every request has the same arguments, and the runner's join stays as it was.
- A Refresh pressed while the two `disk agents` runs are going joins them. The sessions it shows are then a few seconds old; that is accepted.
- Kinds never wait for each other, nor for the queue. A whole-disk walk of several minutes therefore never holds up the session list.
- Two walks beside each other read the disk at the same time. That is accepted, as for `disk` beside a scan.
- On quit the app leaves a lane's run alone. `disk agents` writes no file, so an orphaned run only prints into a closed pipe.
- The app never starts `disk agents` on its own; only the Whole disk view runs it.

Amended 2026-10-28, story "Whole disk: free space with the owning tool" (epic DDC-002): `disk free` is a kind of the queue, beside `clean`, never a lane; its dry run too, as `clean`'s dry run is, so a preview never runs beside a clean or a `disk free`. It takes the ddc lock, so a Scan now pressed during a long `brew cleanup` waits in the queue instead of failing busy, and a scheduled scan behind it waits too. It is never coalesced: two confirmed runs are two runs. The app never runs it beside `disk`: the Whole disk view disables Measure while a `disk free` run is queued or running, and every area action's button while a `disk` run is going, because both write `last-disk.json`. From the terminal they can still overlap; then the last to finish writes `last-disk.json`, as "Whole disk: what lies outside the home folder" accepts. Stop cancels the run by its ticket; the CLI forwards the signal to the tool's process group (item 6.4), so a stopped `brew cleanup` ends as a stopped native command does. A runtime's deletion is done by CoreSimulator's daemon, so a stop during it only ends ddc's wait (the safety ADR's amendment of 2026-10-28). The exclusive lock is held for at most one `NATIVE_TIMEOUT` per action.

UniFFI is not used now (Alternative A).

### 2. UI: `NSStatusItem` and `NSPopover` hosting SwiftUI

- The app delegate owns an `NSStatusItem`. Its button opens an `NSPopover` whose content is a SwiftUI view in an `NSHostingController`.
- The Review window and the Settings window are `NSWindow`s hosting SwiftUI.
- A right click opens a small `NSMenu`: Scan now, Pause scans or Resume scans (added 2026-10-19), Settings, Quit.
- **Icons.** The app has a real icon: an `AppIcon` asset in every macOS size, compiled into the bundle's `.icns` by the build. The status item uses a template glyph; it turns grey while scans are paused, and a mark in its corner shows a warning when something needs the user, or the badge after a notification (2026-10-19). Both are the drive outline with a line and a dot drawn on the canvas, so Finder, Login Items and notifications show the app, not a generic icon. The app icon sets the drive inside a ring of the four class colours, tuned for its dark tile (option B, "Class ring", owner, 2026-10-20).
- `LSUIElement` is set and the activation policy is `.accessory`. The app activates while one of its windows is open. (Amended 2026-10-20 for 0.2.1, story "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open": the owner found the Review window unreachable once another app was in front. While any titled window of the app is open (Review, Settings, the About panel), the policy is `.regular`: the app is in the Dock and in Cmd-Tab, a click on the Dock icon brings its windows to the front, and a main menu (app, Edit, Window) gives ⌘Q, ⌘W and the Edit shortcuts. After the last of them closes, the policy returns to `.accessory` and the app yields the activation to the app that had it, except when the Review window's Close button closed it: that close opens the popover, which activates the app itself, and a yield followed at once by a re-activation can be refused. The popover and the status item never count as windows.)

`MenuBarExtra(.window)` is rejected (Alternative B).

### 3. Minimum macOS 26

The deployment target is macOS 26, as GrammarHelper: current OS first (the owner's decision, 2026-10-18). It is the only version the reference machine runs, so nothing older is claimed. The cask says `depends_on macos: :tahoe`, the form GrammarHelper's cask uses, which current Homebrew reads as Tahoe or later.

### 4. Identifiers

- **The app:** `com.smartandpoint.deepdrivecleaner`, fixed forever. It continues the reverse-DNS family of the 0.1.0 identifiers.
- **The embedded CLI:** signed with the identifier `com.smartandpoint.deepdrivecleaner.cli`. In 0.1.0 the CLI used `com.smartandpoint.deepdrivecleaner`. Keeping that would give two executables the same signing identifier and an overlapping designated requirement with the app, so 0.2.0 renames it. The CLI's Full Disk Access grant was lost on that upgrade anyway, because of the path.
- **The scheduler agent's label:** `com.smartandpoint.deepdrivecleaner.scheduler`. The 0.1.0 CLI agent keeps its label `com.smartandpoint.deepdrivecleaner` until it is migrated (item 5).

### 5. Scheduling: a launch agent that opens the app

**The agent.** The app registers a launch agent with `SMAppService.agent`:

- its plist lives in the bundle at `Contents/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.scheduler.plist`;
- its program is `/usr/bin/open -gjb com.smartandpoint.deepdrivecleaner --args --scheduled`;
- it runs `StartCalendarInterval {Minute: 0}`, an hourly tick.

The plist inside a signed bundle cannot carry the user's chosen time. So the tick is fixed, and one due function decides whether a scan is due. Its inputs:
- the schedule preference: daily or weekly, at a whole hour ("Daily at 10:00" on the canvas); whole hours match the tick;
- the time of the last **scheduled** scan, which the app keeps in `UserDefaults`. It is not `last-scan.json`: a Scan now in the popover or a `ddc scan` in the terminal neither shifts the slot nor suppresses that day's notification;
- a pending reminder scan (Remind me tomorrow, below);
- whether scans are paused (`ScansPaused`, from the menu; 2026-10-19). While paused nothing is due and the agent stays registered; after Resume a missed slot is due at the next tick. The rescan after a clean is part of the clean and runs while paused. (Amended 2026-10-24, story "Faster clean: re-check only what the selection needs": the app runs no scan after a clean that wrote `last-scan.json`, §6 item 1; after a crash that followed the `cleaning` event it still rescans, while paused too. The clean's own re-check runs while paused.)

A slot missed while the Mac slept is caught up at the next tick, because the last scheduled scan is then older than the slot. The popover's "next scan" is computed by the same function.

**When the tick fires:**

- **The app is not running.** LaunchServices launches it with `--scheduled`. In that mode it creates no status item. It scans if one is due, posts a pending notification if one is waiting (below), and quits.
- **The app is running.** `open` delivers a reopen event without arguments. The app treats it as "scan if due".

**A scheduled instance never swallows a user launch.** LaunchServices keeps one instance per bundle id. A launch from Finder, Spotlight or the login item while a `--scheduled` instance runs only reaches that instance, as a reopen or activate event. So in `--scheduled` mode any reopen or activate event, any further launch, and any notification response promotes the instance to the full app: it creates the status item and keeps running after its scan. A tick cannot cause a false promotion: a scheduled instance lives about a minute, and the next tick is an hour later.

Because LaunchServices launches the app, the app is its own responsible process. Its Full Disk Access grant, keyed by bundle id, covers every scheduled and manual scan.

Scans keep running while the app is quit and after `brew upgrade`: `uninstall quit:` stops the app, and the next tick opens the new one.

**How it is listed.** System Settings → Login Items & Extensions lists the 0.1.0 agent as "Evgenii Konev · 1 item", with a generic icon. The "App Background Activity" banner says "Software from “Evgenii Konev”". That is the signer's name, because the agent's program is a bare binary outside any bundle.
- An agent registered with `SMAppService.agent` from the app's bundle is attributed to the bundle, so it is listed under "DeepDriveCleaner" with the app's icon.
- For a plist outside the bundle, the mechanism is the `AssociatedBundleIdentifiers` key. `man launchd.plist` says it "indicates which bundles are associated with this job in the System Settings Login Items UI". When the CLI runs from inside the app bundle, `ddc schedule install` adds that key, naming the app's bundle id.
- The developer name shown in these places becomes the organization's only after the switch to the SmartAndPoint team (item 8).

**The login item.** `SMAppService.mainApp` is separate. It keeps the menu bar icon at login and is not needed for scheduled scans. Both toggles in Settings show the system's state (`SMAppService.status`), never a stored bool.

**Registration heals itself.** At every launch with scheduled scans on (the Scan setting is not Off), the app calls `register()`, which is idempotent. That covers an agent left at `.requiresApproval` and approved later, and an agent dropped when `brew` replaced the bundle.

**Who owns the schedule.** `settings.json` records it as `scheduler: {by: "app", app: "<path of the app bundle>"}`. The app writes the record whenever the agent's status is `.enabled`. It clears it only when scheduled scans are off (the app then also unregisters the agent) or when registration fails. A status of `.requiresApproval` leaves the record as it is.

`ddc schedule install` refuses only while that record exists **and** the recorded app path exists. A stale record, left because the app was removed, is ignored and cleared with a note. Nothing locks the CLI out after the app is gone.

**2026-10-28, "Background scan and notifications in the app".** Three changes to this section:
- **The agent's program.** The bundled agent runs the app binary with `--tick`, the fallback in Open Questions, and `--tick` opens the app by its own bundle path. `SMAppService` documents bundle programs, so Login Items is expected to attribute the agent to the app rather than to `/usr/bin/open`. The first manual check of the 0.3.0 release confirms it. If the bundle program is refused, the app falls back at runtime to a second bundled plist that runs `/usr/bin/open`.
- **When the migration offer shows.** It shows whenever the legacy plist exists and the offer has not been dismissed, not only at the first launch. This covers agents installed by 0.1.0 and by a 0.2.x `ddc schedule install`.
- **No second scheduler.** While that plist exists, the app neither registers its agent nor runs scheduled scans, so two schedulers never run together.
- **Remind me tomorrow** is anchored on the tap, not on the notifying scan. `<time>` below is the schedule hour on the calendar day after the tap. A notification held back by working hours, or tapped days later in Notification Center, still means "tomorrow".

**Migration from 0.1.0** is offered, never silent. While the legacy plist exists (see the note above), the popover offers to replace the CLI's background scan. The same offer stays in Settings → Permissions ("Old background scan found · Replace it") for as long as `~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` exists. Replacing it:

1. reads the agent's interval into the app's schedule and its threshold into `settings.json`. The 0.1.0 agent runs at 11:30, daily or on Mondays (`crates/ddc-cli/src/launchd.rs`); the app's schedule becomes 11:00 with the same interval, since the app schedules whole hours;
2. runs `deepdrivecleaner schedule uninstall`;
3. registers the app's agent.

**Notifications.** Only scheduled scans notify; a scan the user starts shows its result in the popover. The notify decision rule stays in Rust. The background scan gains `--notify json`, which prints the decision and writes no notification state; it still saves the scan to `last-scan.json` and logs to `background.log`, because the app reads the scan after a scheduled run.

The app posts the notification with `UNUserNotificationCenter`:

- a category with two actions, as on the canvas:
  - **Review** opens the Review window;
  - **Remind me tomorrow** snoozes notifications for 24 hours (the owner's decision, 2026-10-18). If the space is still over the threshold, or the disk still almost full, at the first scheduled scan after the snooze ends, the app notifies: for the threshold even when the weekly or the 20 % growth repeat rule would hold it back, and for low disk because that rule never holds it back (§6 item 9). With a weekly schedule there is no scheduled scan the next day, so the due function adds one reminder scan at the first tick after the snooze ends, and the label stays true. If the app is quit when the user picks the action, macOS launches it to deliver the response: it starts in normal mode, keeps its status item, and the badge clears;
- an attention badge on the status item, if "Show a badge on the menu bar icon" is on;
- "Only notify during working hours" (9:00 to 19:00 on weekdays on the canvas). A decision made outside them is kept as pending in `UserDefaults` and posted at the first hourly tick inside them; `--scheduled` can post a pending notification without scanning. A newer decision replaces a pending one.

**Notification permission.** It is asked for at the first normal-mode launch after the first scan, or in the migration flow, never by a `--scheduled` instance. If a `--scheduled` instance finds the permission not yet determined, it sets a flag that the next popover open surfaces with a button to allow notifications.

**The notify state.** `notify-state.json` has one writer, the CLI. The decision path (`--notify json`) only reads it. The app writes it through two hidden subcommands:

- `deepdrivecleaner notify ack --bytes <n>`, after a notification was actually shown. It records the time and the amount, as 0.1.0 did after `osascript` succeeded, and ends any snooze. A decision waiting for working hours is acknowledged when it is posted. One held back by denied permission is not acknowledged, so the next scan decides again. A low-disk notification is acknowledged like any other, so it becomes the threshold's repeat baseline: once the volume recovers, a threshold notification waits for a fifth more than the amount the last low-disk notification named, or a week from it (owner, 2026-10-19).
- `deepdrivecleaner notify snooze --until <time>`, when the user picks Remind me tomorrow. It records `snooze_until` and clears the repeat baseline (`last_at`, `last_bytes`). While a scan starts before `snooze_until`, the decision is "do not notify". The first scan after it decides as if no notification had ever been shown, which is the override the owner asked for. The app sets `<time>` to the notifying scan's slot plus 24 hours. For a daily schedule that is tomorrow's slot, so tomorrow's scan is the first after the snooze, not the one after it; for a weekly schedule it is the reminder scan.

### 6. What the CLI adds

These changes are additive. One change is visible in the terminal: a held lock exits 75 instead of 1. The existing refusals keep exit code 2.

1. **`ddc clean --items <file|-> --json`.**
   - **Input.** A JSON list (`schema_version`) of items from the report the user reviewed: `path`, `dev`, `ino`, `rule`, `class`, `action` (`delete`, `trash`, `native`, `native-only`) and, for `review` items, `permanent`.
   - **Matching.** It rescans, as every clean does (the safety ADR §5; amended 2026-10-24, see "Re-check walk" below). A listed item runs only if the fresh plan finds it again with the same path, `(dev, ino)`, rule and class, and derives the same action for its `permanent`. Any mismatch skips it as `changed-since-scan`. A `user-data` item is the exception: it is an entry of a user-data folder, matched against the fresh scan's full list of that folder's entries (the safety ADR's amendment of 2026-10-20). A `review` item confirmed for the Trash can never run as a delete because a lock file appeared in the meantime.
   - **Selection.** The list is the selection: no class filter, no default selection. `disabled_rules` never drops a listed item. The app never lists an `ignored` item (item 7).
   - **Refusals.** A global `--permanent` together with `--items`, and a malformed list, are refused with exit code 2.
   - **Output.** The execution report as JSON: the outcome per item, skip reasons, and free space before and after for each volume touched.
   - **Re-check walk** (amended 2026-10-24, story "Faster clean: re-check only what the selection needs"). The "fresh plan" above comes from the walk prescribed by the safety ADR's amendment of 2026-10-24:
     - pruned to the listed items, their ancestors and, for a `Siblings` rule, their siblings;
     - the whole tree when the list holds a `Tree` item, when an item's ancestor cannot be read, or when no report of the current schema is on disk; added 2026-10-25, also when the report on disk cannot be read or was scanned from other roots or another home (`ddc scan ~/Code`).
   - **The report after the clean** (amended 2026-10-24). The CLI writes `last-scan.json` once, after the clean:
     - in the pruned case, by editing the report on disk under its lock, whichever scan wrote it;
     - in the full-walk case, by editing the rescan's.

     The app runs no scan after a clean that wrote it; the clean report says so in `last_scan_saved` (added 2026-10-25: written only by `clean --items`, false when the write failed, absent from an older CLI, which the app reads as false). After a crash, or an exit before the write (a journal write failure mid-run), the file is the one from before the clean and lists what was removed. The app then rescans, as today, when it saw the `cleaning` event.

     What the edited report holds:
     - A deleted item leaves, with its bytes, files and directories in every total.
     - A trashed item leaves. When it lies on the home volume and the report holds the `trash` finding, its bytes, files and directories move to that finding; otherwise it counts as deleted. A picked user-data entry also leaves its folder's contents, with the findings inside it. Added 2026-10-25 at the story's plan gate: a picked folder moves its bytes only, since its files and directories are not known; the findings inside it leave without moving bytes, which the entry's bytes already hold.
     - An item a tool cleaned is measured again and the difference subtracted. Skipped, failed and not-run items stay.
     - In the pruned case `generated_at`, `duration_ms` and `ddc_version` stay the previous report's; in the full-walk case they are the rescan's.
     - After a cooperative cancel the file is written with what was done. A crash leaves the report from before the clean.
2. **`ddc clean --items … --dry-run --json`** gives the confirmation sheet its live warnings, such as "Spotify is running · will be skipped". It does **not** rescan. It runs the executor's per-item re-checks against the listed items only:
   - identity: the path still has the listed `(dev, ino)`;
   - class and action, derived again from the rule and the project's lock file;
   - kept (`.ddc-keep` in the item or above it, an exclude), in use, owner running, protected.

   It walks no unlisted tree. Results stream on stdout, one JSON line per item as its checks finish, then a summary line, so the sheet opens at once and the warnings fill in. Each line says would delete, would trash, would run a command, or would skip with its reason (`owner-running`, `in-use`, `changed-since-scan`, …). It renames, deletes and trashes nothing and runs no native command. The real clean still re-checks everything with item 1's walk (amended 2026-10-24: pruned unless the list needs the whole tree).
3. **Progress.** `--progress json` writes newline-delimited events to stderr:
   - scan: files, directories and bytes so far;
   - clean: the preparing phase, then item `i` of `n` with its path and outcome.

   The core counter belongs to the DDC-001 story "Progress output while ddc clean runs", which this depends on.

   (Superseded in part 2026-10-25, this and the next paragraph; see the note below.) Amended 2026-10-24, story "Faster clean: re-check only what the selection needs": the `cleaning` phase event is written to stderr, flushed, after the journal is open and the signal handler installed, immediately before `exec::execute`. The executor's own preparation (the local-snapshot count, the keep-marker walk of the chosen items, the process snapshot) runs after it. The event therefore means the CLI may touch files from now on, not that the first action has started; a crash in that preparation points to the journal although nothing was done. Story "CLI: progress events", whose preparing phase today ends after the keep-marker walk, decides whether the event moves into the executor when its counters land. The app reads two things from it, through the runner's live stderr stream and never from a bounded tail or the report file's date:
   - the switch from preparing to cleaning;
   - after a crash, whether anything may have been touched: event seen, the journal lists what was done; not seen, nothing was touched.

   `--progress json` is accepted by `clean` as a whole, with the phase events (`preparing`, `cleaning`) only until story "CLI: progress events" adds the counters and item events; a first `docs/progress.schema.json` holds them. The app ignores lines on that stream that are not JSON, such as warnings or "stopping after the current item".

   Amended 2026-10-25, story "CLI: progress events": the `cleaning` event moved into the executor. The paragraph above describes the place planned for it on 2026-10-24, before `exec::execute`, which never shipped; `docs/progress.schema.json` was created by this story with the whole stream, not by story "Faster clean: re-check only what the selection needs" with phase events only. A hook on the executing thread (`ExecEvents` in `ExecOptions`) writes the event after the executor's own preparation (the local-snapshot count when the caller did not pass one, the keep-marker walk, the process snapshot) and immediately before the first action. It is written even when no item runs, a cancel during the marker walk included, and never when `execute` refuses (root, `review` with `--yes`). The event now means the first action is about to start, so the crash rule above holds without the false alarm of the preparation. Until story "Faster clean: re-check only what the selection needs" moves the write of `last-scan.json` after the clean (item 1), the clean still saves its full rescan before the executor runs, and the app's crash rule is "the file was saved, or the `cleaning` event was seen"; a crash in the executor's preparation therefore still points to the journal, through the saved file. Once that story lands only the event remains, and such a crash reads "nothing was touched". The sheet's step 1, "Checking again", covers the re-check walk, then the keep-marker walk. The schema, version 1, holds `phase`; `walk` ticks (step `scan` under `ddc scan`; `recheck`, then `markers`, under `clean`); `items` ticks; and one `item` line per item the executor was given, a `not-run` item included. No `items` tick comes before `cleaning`, and no `walk` tick after it. The runner keeps lines that start with `{` out of its stderr tail.

   Amended 2026-10-28, story "Whole disk: what lies outside the home folder" (epic DDC-002): `ddc disk` writes `walk` ticks with step `disk`, the last carrying the report's walk totals. The schema stays at version 1: an app that does not know the step fails to decode that one line and ignores it, while a version bump would make it read every event as another version.
4. **Cancellation.**
   - A scan needs no handler: SIGTERM ends the process, and `last-scan.json` is only ever replaced atomically at the end.
   - A clean installs a handler for SIGINT and SIGTERM (in the core's `sys` module, or through `signal-hook`) that sets a cooperative flag checked by `exec::execute` between items.
   - A native command runs in its own process group (`CommandExt::process_group`, stable std, no `unsafe`), so the app can stop it: `go clean -modcache` can take up to 10 minutes. That also takes it out of the terminal's foreground group, so Ctrl-C would no longer reach it. The handler therefore forwards both SIGINT and SIGTERM to the command's group (`-pgid`).
   - A clean stops after the current item, from the app and from Ctrl-C alike, and the journal holds every completed action.
5. **Exit codes.**
   - 0: done;
   - 1: error;
   - 2: refused, both the existing refusals and the new `--items` ones;
   - 75 (`EX_TEMPFAIL`): another scan or clean holds the lock (today 1).

   `scan --background` keeps exiting 0 on a held lock and logs "skipped". With `--notify json` it also prints `{"status": "skipped", "reason": "busy"}`.
6. **A settings file**, `<data-dir>/settings.json`, with its own `schema_version`. It holds what the CLI reads:
   - scan roots and excludes;
   - `disabled_rules`, `min_age_days` and `min_size`;
   - the notification threshold and the low-disk notification switch;
   - the `scheduler` record (item 5).

   `scan`, `clean` and the background scan read it when a flag is absent; a flag wins. One exception, decided by the owner on 2026-10-19: an `--exclude` that drops folders the file protects first names exactly those folders. `clean` asks `[y/N]` in a terminal, before the lock, and a declined answer changes nothing; it refuses (exit 2) without a terminal or with `--items`. A scan and a plain `clean --dry-run`, which change nothing, only warn. A settings root or exclude that does not exist (an unplugged disk) is skipped with a note. A malformed file is an error that names it. The app writes the file atomically. The CLI writes it only to clear a stale `scheduler` record.

   **2026-10-28, "Settings: folders, types, notifications".** The file also holds `skips`: folders that are never read. See the safety ADR's amendment of that date. `--skip` only adds to `skips`, and `--exclude` never drops them, so the exclude confirmation above does not apply to them. The app writes `skips` only when the list is not empty, and the field keeps `schema_version` 1. A 0.2.x CLI refuses the unknown key and names the file.

   Preferences only the app uses stay in its `UserDefaults`, as in GrammarHelper: the schedule, whether scans are paused (`ScansPaused`, 2026-10-19), the size next to the icon, the badge, working hours, and whether `review` items are deleted instead of moved to the Trash by default in the confirmation sheet.
7. **`ignored` findings.** A finding whose rule is in `disabled_rules`, or whose size is under `min_size`, gets `ignored: true` and `selected: false`. It keeps the `not_selected` it would have had, so a type switched back on can be applied at once; `ignored` takes precedence. It is not counted in the class or selected totals or in the notify decision; the report counts it apart.

   `not_selected` is a tagged enum, so a new reason there would bump `schema_version`. `ignored` is a new optional field and does not.

   `ignored` is computed at scan time. When the user changes Types or the minimum size, the app applies the same rule to the loaded report at once, and the next scan writes it.

   `min_size` has two meanings, kept apart. The settings value is a scan-time floor that makes findings `ignored`. The `--min-size` flag of `clean` stays a plan filter. A plan leaves out items under the larger of the two.
8. **Rule groups.** `ddc rules --json` lists every rule id with its category, class, removal method, whether the Types tab can switch it off, and a `group` label ("Python environments", "node_modules", "Package downloads", …; the groups were re-cut on 2026-10-28 to match the Settings design, see "Settings: folders, types, notifications"). It is the source of the app's Types settings.
9. **Volumes.** The report gains an optional `volumes` list: the free and total bytes of the volume of each scan root. The low-disk check applies to every scanned volume, not only the home folder's, and the notify decision names the volume. As in 0.1.0, it fires for a volume only when something selected lies on it: otherwise `ddc clean` cannot help (owner, 2026-10-19). Unlike 0.1.0, the repeat rule (a fifth more, or a week) holds back only the threshold. While a volume stays almost full with something selected on it, every scheduled scan notifies, a scan caught up after sleep included, until nothing selected is left on it or the user picks Remind me tomorrow. There is no minimum amount and no minimum gap: a few KiB selected on a full disk is enough. A 0.1.0-style agent from `ddc schedule install` repeats the same way, and `notify_low_disk: false` in `settings.json` stops it (owner, 2026-10-19).

10. **The whole-disk reports** (amended 2026-10-28, epic DDC-002).
    - **`ddc disk [--json] [--progress json]`** (story "Whole disk: what lies outside the home folder"). It prints the disk report with its own `schema_version` and saves it to `last-disk.json`.
    - **`ddc disk agents [--json] [--no-sizes]`** (story "Coding agents in the Whole disk view"). It prints what coding-agent harnesses keep under the home folder, and their live sessions, with its own `schema_version`. It saves nothing, takes no lock and sends no notification. Liveness is read again on every run. With `--no-sizes` it walks nothing, so it answers in a fraction of a second, and the report says that its sizes are absent. The app runs it that way first, then without the flag, one after the other in the same lane, so the session rows show before the sizes. It reports no progress: the first run already fills the section, and the second only adds the sizes.
    - **Skips.** `skips` in `settings.json` bind both commands, the session-record read included. A harness folder under a skip is never opened: no sizes are walked and no record is read. The harness is reported with the skip as its reason. A skip inside a harness folder, such as `~/.claude/projects`, is never opened either: that folder is reported skipped, and what is read only through it, such as the transcripts, is reported absent with the same reason. This holds the safety ADR's promise that a skipped folder is never opened.
    - **Exit codes for both.**
      - 0 when done, a closed stdout included.
      - 2 on a usage error.
      - 1 when no report can be made at all, for example without a home folder. A folder, a record or a database that cannot be read is reported in the output, never turned into an error.
      - For `ddc disk`, 1 also when the report could not be saved. It is still printed; the app shows the last saved report with the error.
      - A stopped run ends by its signal, as a clean does (item 4).
      - Neither command takes the lock, so neither returns 75.
    - **What is a string and what is an enum.**
      - **Values copied from a harness record** are typed in the schema as they were read: as strings or numbers. They are never re-typed into an enum. Examples are a Claude Code session's `status` and `entrypoint`. Their values are another program's internals, and a new value must not make the report unreadable.
      - **Reading the records.** The CLI reads them leniently and ignores unknown keys. A record without the fields its test needs is reported as `unknown`, and a records folder of an unexpected shape as `records: unknown`. Neither is ever reported as an ended session.
      - **Values that ddc assigns itself** are strict enums, such as the kind of a running harness process: `cli`, `app-server`, `app` or `other`. Here `other` is a value the CLI emits for a process it cannot name, not a catch-all for the decoder. A new variant bumps `schema_version`, as for every report.
      - **The app** decodes both reports strictly, as §1 requires.

11. **`ddc disk free`** (amended 2026-10-28, story "Whole disk: free space with the owning tool"). It runs an area action of the disk report: `simctl-runtime-delete` for one simulator runtime, or `brew-cleanup`. The safety ADR's amendment of 2026-10-28 holds the guard, the lock, the measuring and the journal.
    - **`ddc disk free --actions <file|-> [--dry-run] [--json]`.** The list is `{"schema_version": 1, "actions": [{"action": "simctl-runtime-delete", "identifier": "<asset id>"}, {"action": "brew-cleanup"}]}`, the ids those of the disk report's `action`. An unknown field anywhere in it, an action named twice, an unknown id, an `identifier` on `brew-cleanup` or none on a runtime, or a malformed list refuses the whole list (exit 2) before anything runs, as `--items` does.
    - **`--dry-run`** runs each tool's own dry run after the same location probe: `simctl runtime delete <identifier> --dry-run`, and `brew cleanup -n` with the run's environment. It prints what each would remove and its estimate, takes the shared lock as `clean --items --dry-run` does, writes no journal line and changes no report. The confirmation sheet opens on it.
    - **Without `--actions`** at a terminal it lists the actions available now, with their dry-run estimates, and asks for each, default No, before it takes the exclusive lock. Without a terminal it refuses (exit 2). There is no `--yes`.
    - **Which rows offer an action.** A runtime whose `parentMountPath` lies under `/System/Library/AssetsV2/`, not `deletable: false`, with a `runtimeIdentifier` and no booted device; the area `homebrew`, whose folder is `/opt/homebrew`. The app disables the button otherwise and says why; the CLI's probe decides again at run time.
    - **Output.** A report per action, in the order listed: ran, refused (with the reason), failed, still deleting, or removed but not freed; the bytes before and after; and for a failure the exact command to run in Terminal. Schema `docs/disk-free.schema.json`, version 1.
    - **After the run**, under the exclusive lock and before the report is printed:
      - it measures the `cache.homebrew` finding in `last-scan.json` again, when that report has the current schema and the home's roots (item 1's "The report after the clean");
      - it edits `last-disk.json`, written atomically, and skips that with a note when the file is absent or of another schema version. After `brew-cleanup` the `homebrew` area gets `stale: true`. A runtime that left the list leaves `simulators.runtimes`, and `listed_bytes` becomes the sum of those that remain. After any runtime action that ran, one still deleting included, the `simulator-runtimes` area gets `stale: true`. A stopped run still makes these edits before it ends by its signal, as `clean --items` writes its report after a cancel. Every other number, `not_counted` included, and `generated_at` stay as measured. `stale` is a new optional field: the app ignores unknown keys and decodes only the version and the enums strictly, so it needs no version bump, as `ignored` did. The app shows a stale area's bytes as "before cleanup" until the next Measure.
    - **Exit codes**, as `clean --items`: 0 when the list was carried out, refused or failed actions included (the report says which); 2 when refused before anything ran; 75 when the lock is held; 1 on an error before any action, or, without `--json`, when an action failed.

### 7. Consent: the confirmation sheet is interactive consent

This **amends** §5 of the ADR "Cleanup safety model: risk classes, plan and confirmation". That ADR stays accepted, and the change is recorded there as a dated amendment.

**What it allows.** An explicit item list that a person confirmed counts as interactive confirmation for every listed item. That includes `review` items and `native-only` commands. The list comes from the app's confirmation sheet, or from a person at a terminal naming items with `ddc clean --items`.

**What this changes.** Until now no non-interactive call could remove a `review` item: `--yes` refuses the `review` class. With `--items`, a script without a TTY can move named `review` items to the Trash. With `permanent: true` per item, it can delete them permanently. That is the cost of a machine-readable consent channel. It is bounded by:

- consent is per item: no blanket `--permanent`, and the class and action must match the fresh scan (item 6.1);
- the executor still refuses actions the rule does not allow: `user-data` (except an entry the scan lists inside a user-data folder, moved to the Trash: the safety ADR's amendment of 2026-10-20), `none`, and anything but its commands for `native-only`. It re-checks every item (§1 and §5 of the safety ADR);
- in the app, the sheet's Clean action is the only caller of `clean`, and a test enforces this. Notifications, the scheduler and the popover can only open the Review window or the sheet.

§8 of the safety ADR ("Background mode") still holds: the background path never calls the executor.

Amended 2026-10-28, story "Whole disk: free space with the owning tool" (epic DDC-002): an area action is consented to the same way. The confirmed action list that the Whole disk view's confirmation sheet passes to `ddc disk free --actions` counts as interactive confirmation for each listed action, as an item list does for `clean`. The sheet shows the tool's dry run, the exact command, and what the action cannot undo. One caller per destructive command: in the app the sheet's confirm button is the only caller of `disk free` without `--dry-run`, as Clean is the only caller of `clean`, and a contract test enforces both. Notifications, the scheduler and the popover never call it. `--actions` lets a script without a TTY run `brew cleanup` or delete a runtime; that is the cost of the machine-readable consent channel, as it is for `--items`.

### 8. Build, signing and distribution

- **Build.** An Xcode project, `app/DeepDriveCleaner.xcodeproj`, in this repository. `make app` builds the CLI first and copies it into `Contents/Helpers`.
- **Signing.** The CLI is signed first, then the app, with the Developer ID, hardened runtime and timestamp. No sandbox; no entitlements beyond the defaults.
- **Notarization.** The zipped app is notarized, the app is stapled and zipped again, and `codesign --verify --deep --strict` and `spctl --assess --type execute` check it.
- **Distribution.** One cask for the CLI and the app (confirmed by the owner, 2026-10-18), `deepdrivecleaner`, version **0.2.0**:
  - `app "DeepDriveCleaner.app"`;
  - `binary "#{appdir}/DeepDriveCleaner.app/Contents/Helpers/deepdrivecleaner"`, and a second `binary` to the same file with `target: "ddc"`;
  - `depends_on macos: :tahoe` (item 3) and `depends_on arch: :arm64`;
  - `uninstall quit: "com.smartandpoint.deepdrivecleaner"`. It also runs on upgrade, and the scheduler agent reopens the new app at the next tick. (Note, 2026-10-20: 0.2.0 ships without the scheduler agent, which comes with 0.3.0, so after an upgrade nothing reopens the app before the next login or a manual open; the 0.2.0 release notes say so. Story "Distribution: the 0.2.0 cask with the app".)
  - `zap` the data, logs and preferences, with `launchctl:` for both the 0.1.0 agent and the scheduler agent.

  A zip, not a DMG: the cask is the install path, and a zip of a stapled app also works when downloaded by hand.
- **The CLI's path.** It becomes stable (`/Applications/DeepDriveCleaner.app/Contents/Helpers/deepdrivecleaner`). That matters only when launchd starts the CLI (`ddc schedule install` for people who use only the CLI): its own Full Disk Access grant then survives upgrades. Run from Terminal, the CLI is governed by Terminal's grant.
- **Version.** The workspace version and the app's `MARKETING_VERSION` are the same. `make dist` refuses a mismatch.
- **Team.** 0.2.0 is signed by the existing team, 8MSDJTBW2D (the owner's decision, 2026-10-18): a proper app first. The switch to a SmartAndPoint organization team is later work, tracked outside this vault, not a step of this release. It has two visible effects:
  - a one-time cost: the designated requirement includes the team, so every user grants Full Disk Access once more;
  - the developer name in Login Items & Extensions and in the background-activity banner becomes the organization's.

## Rationale

1. **One Full Disk Access grant that survives upgrades** is the main user-facing win. It needs the app, launched by LaunchServices, to be the responsible process for every scan.
2. **A calendar agent that opens the app** keeps scheduled scans running while the app is quit and after upgrades, without a second grant.
3. **The process boundary keeps the safety model in one place.** The re-checks, the tombstones, the journal and the refusals stay in the executor that DDC-001 tested and ran for real. The app can only pass a list whose class and action are checked again.
4. **Crash and cancel isolation.** A walker bug or a stuck native command kills a child process, not the menu bar.
5. **AppKit for the shell, SwiftUI for the views** matches GrammarHelper where it works, with the control `MenuBarExtra` lacks.

## Alternatives Considered

### Alternative A: UniFFI, the core linked into the app

**Pros**:
- No serialization; typed calls; progress callbacks without parsing.
- No second executable in the bundle.

**Cons**:
- A panic or a hang in the walker happens in the app's process.
- Cancellation must be cooperative everywhere, including native commands.
- Two build products of the core (a static library and the CLI), since the CLI still ships.
- A Rust static library per architecture in Xcode, plus generated bindings.

**Rejected because**: the boundary costs one `posix_spawn` and a JSON parse per operation, nothing next to a 40-second scan, and isolation matters more for a tool that deletes files.

### Alternative B: SwiftUI `MenuBarExtra(.window)`

**Pros**:
- Less code: a scene instead of an app delegate and a popover.

**Cons**:
- No API to open the panel programmatically, which a notification's Review action needs.
- No right-click menu; limited control over the status item image (badge, size).

**Rejected because**: the notification and badge flows need the control.

### Alternative C: `NSBackgroundActivityScheduler` inside the app

**Pros**:
- No launchd job.

**Cons**:
- It runs only while the process lives. `uninstall quit:` on every `brew upgrade` stops it, and so does quitting the app.
- Its `interval` is an average with a tolerance of about half the interval. There is no time of day, which the Settings design and 0.1.0's `StartCalendarInterval` offer.

**Rejected because**: scans would silently stop after an upgrade and could not run at a chosen time.

### Alternative D: An `SMAppService.agent` that runs the CLI directly

**Pros**:
- Scans run while the app is quit; close to the 0.1.0 LaunchAgent.

**Cons**:
- launchd starts the CLI as its own responsible process, so it needs a second Full Disk Access grant.
- The notification would have to come from the CLI again, without actions.

**Rejected because**: two grants is the problem the app is meant to remove. Opening the app (item 5) keeps one.

### Alternative E: Two casks, CLI and app

**Pros**:
- Terminal users install no app.

**Cons**:
- Two versions to keep in step, two release assets.
- The CLI cask keeps the path-keyed Full Disk Access problem for `ddc schedule`.

**Rejected because**: one bundle gives one version and a stable CLI path. Terminal users simply never launch the app.

## Consequences

**Positive**:
- One Full Disk Access grant, by bundle id, covers manual and scheduled scans and survives `brew upgrade`.
- Scheduled scans run at the user's time, also while the app is quit.
- The cleaning logic and its tests stay in Rust.
- Notifications have actions and show the app's name and icon.

**Negative / trade-offs**:
- Every clean still rescans first: about 40 s on the reference machine, shown as progress. (Amended 2026-10-24, story "Faster clean: re-check only what the selection needs": only the terminal's `ddc clean` and a list that needs the whole tree, which is one of these: a list holding a toolchain version (`toolchain.nvm`, `toolchain.pyenv`, `toolchain.rustup`), a list with an item whose ancestor cannot be read, or a run with no readable report; added 2026-10-25, a report scanned from other roots or another home counts as no report. Any other list re-checks only its items and what lies above and beside them.)
- The scheduler wakes the app once an hour to check whether a scan is due.
- Login Items lists two entries for the app: the login item and the scheduler agent.
- `brew uninstall` without `--zap` leaves both background items in Login Items until macOS drops them; the README says so.
- `--items` lets a script remove named `review` items without a TTY (item 7).
- Five new CLI contracts must stay stable, each with a `schema_version`: the `--items` input, the clean report, the progress events, the settings file and the notify decision. Amended 2026-10-28: the list is now the `--items` input, the clean report, the progress events, the settings file, the notify decision, the rules listing (item 6.8), the disk report and the agents report (item 6.10).
- A Swift toolchain (Xcode) joins Rust in the build.

**What changes in code / process**:
- New: `app/` (Xcode project); `make app`. `make dist` and `scripts/publish-release.sh` produce a zipped app.
- CLI:
  - `clean --items` with `--json` and `--dry-run`;
  - `--progress json`;
  - SIGTERM handling for clean, with forwarding to native commands;
  - exit code 75 for a held lock;
  - `settings.json`, `ignored`, `rules --json` and `volumes`;
  - `scan --background --notify json`, `notify ack` and `notify snooze`;
  - `schedule install` respecting the `scheduler` record, and adding `AssociatedBundleIdentifiers` when run from inside the app.
- Cask: `app` plus `binary` from the bundle, `depends_on macos`, `uninstall quit:`, `zap launchctl:` for both agents.
- Safety ADR §5: dated amendment pointing here, and a second one of 2026-10-24 (story "Faster clean: re-check only what the selection needs").

**Implementation notes**:
- **Pipes.** The app reads the CLI's stdout and stderr concurrently. A full 64 KiB pipe would deadlock a CLI that writes to the other one.
- **The attribution check** verifies item 5's claim on a clean machine: the owner's work laptop, after the 0.2.0 cask is installed, with Full Disk Access granted to the app only and the app started through LaunchServices, never from Xcode (the debugger changes attribution) or Terminal (owner's decision, 2026-10-18; it replaced a spike on a fresh user account). Development dogfoods on the reference machine first, with the grant on the app, as GrammarHelper does; Debug builds carry spike tools (a Full Disk Access probe, agent registration, a test notification).

## Open Questions

- Settled on 2026-10-28: the scheduler agent runs the app binary with `--tick`, as described in §5. The first manual check of the 0.3.0 release confirms that `SMAppService` accepts it.
- Does `--items` need a full rescan (about 40 s), or would a scoped re-walk of the listed findings plus the executor's re-checks give the same safety? Decided at that story's plan gate (2026-10-18): the full rescan stays, because only a whole-tree walk sees the toolchain pins (`toolchain::dropped` needs every pin file and sibling version the walk found) and the ancestor claims (a rule that would now claim a directory above an item). `--exclude` is a path-prefix test and needs no walk. Revisited on 2026-10-24, after the owner's decision of 2026-10-21 (story "Faster clean: re-check only what the selection needs"): the full rescan stays only for lists that need the whole tree. Both reasons above still hold. The pins make the toolchain rules `Tree`, and an ancestor claim is seen because the pruned walk passes through every ancestor with the same rule visitor. The objection to a second classification path does not apply: the pruned walk reuses the visitor, the classification and the plan unchanged. See the safety ADR's amendment of 2026-10-24.

The owner's questions (minimum macOS, the notification's second action, the signing team for 0.2.0, one cask) were decided on 2026-10-18 and are recorded in items 3, 5 and 8. The 2026-10-19 decisions (the exclude confirmation, the per-volume low-disk gate, low disk outside the repeat rule) are in §5 and in items 6 and 9 of §6; the owner's icon and pause decisions of the same day are in §2, §5 and item 6 of §6; the app icon's design (2026-10-20) is in §2.

## References

- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release|Homebrew tap and the v0.1.0 release]]: path-keyed Full Disk Access for the 0.1.0 cask
- [[epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs|Progress output while ddc clean runs]]
- GrammarHelper repository: `App/AppDelegate.swift`, `App/SettingsManager.swift`, `Makefile`, `scripts/publish-release.sh`, `CLAUDE.md`
- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-28*
