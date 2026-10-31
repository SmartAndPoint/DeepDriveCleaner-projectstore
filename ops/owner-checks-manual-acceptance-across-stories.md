---
type: runbook
slug: "owner-checks-manual-acceptance-across-stories"
title: "Owner checks: manual acceptance across stories"
status: draft
date: 2026-10-30
authors: ["Evgenii Konev"]
tags: [acceptance, manual-checks, owner]
---

# Owner checks: manual acceptance across stories

## Purpose

The owner decided on 2026-10-30: a story closes as `done` when every automated acceptance criterion is ticked with evidence. Its manual checks, the ones only the owner can run on a real machine, move here and wait until the owner runs them.

Use this list when a release is installed on the reference machine and there is time to check it by hand. Each check sits under the story it came from. A failed check opens a new bug story in that story's epic; it does not reopen the closed story.

## Prerequisites

- [ ] The current release, 0.12.1, installed through Homebrew on the reference machine.
- [ ] Full Disk Access granted to DeepDriveCleaner.app, and the app started from Finder, not from Terminal or Xcode.
- [ ] For a check that needs a tool or an app (Docker Desktop, Google Chrome, conda, Xcode simulator runtimes, Homebrew formulae): that tool installed.
- [ ] For the checks that name another machine: the owner's work laptop, where Terminal has no Full Disk Access, and a second Mac or a fresh user account.

## Steps

**Most important:** the clean-machine access check on the work laptop, under "Distribution: the 0.2.0 cask with the app" below. It is the only check of Rationale 1 of the ADR "Menu bar app: Swift shell over the ddc CLI" (one Full Disk Access grant, to the app, that survives upgrades) on a Mac where Terminal has no grant. Run it first.

### Background scan and notifications in the app (DDC-002)

Story: [Background scan and notifications in the app](../epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md)

- [ ] Manual check on the reference machine with the app quit: at the next tick (`app.log` shows `--tick`, then a `--scheduled` launch) the app is opened with `--scheduled`, scans, posts and quits, with no Dock icon and no change of focus; Review opens the Review window; Remind me tomorrow posts nothing until the first scheduled scan after the snooze, which posts if still over the threshold or still almost full
- [ ] Manual check: after the first launch and the migration, System Settings → Login Items & Extensions lists the scheduled scan under "DeepDriveCleaner" with the app's icon, not under a developer name
- [ ] Manual check after the migration: the 0.1.0 LaunchAgent is gone; `launchctl print gui/$UID/com.smartandpoint.deepdrivecleaner` finds no job and `~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` does not exist
- [ ] Manual check: after the app is quit and replaced (as `brew upgrade` does), the next tick opens the new app and the scheduled scan runs
- [ ] Manual check: double-clicking the app in Finder during a scheduled scan shows the menu bar icon, and the app keeps running after the scan
- [ ] Manual check: with the Review window open, a Dock-icon click brings it forward, and the hourly tick does not
- [ ] Manual check: with the app quit, Remind me tomorrow on a notification launches it in normal mode with its status item, and the badge is cleared
- [ ] Manual check: `ddc schedule install` refuses while the app's agent is registered, and installs after the app is removed (the refusal shipped with "CLI: shared settings, types and the notify decision"; this checks it against the real agent)

### Faster clean: re-check only what the selection needs (DDC-002)

Story: [Faster clean: re-check only what the selection needs](../epics/DDC-002/stories/story-faster-clean-re-check-only-what-the-selection-needs.md)

- [ ] **AC 9.** On the reference machine, with the app log's times (manual check by the owner):
  - a clean of caches only re-checks its items in under 1 s;
  - the whole time before the first action stays under 3 s.

### Popover: the Next scan line opens the Schedule settings (DDC-002)

Story: [Popover: the Next scan line opens the Schedule settings](../epics/DDC-002/stories/story-popover-the-next-scan-line-opens-the-schedule-settings.md)

- [ ] Manual (owner): clicking "Next scan …" closes the popover and opens Settings → Schedule, both when Settings was closed and when it was open on another pane.

### Review window: after a clean, only skips that can clear stay selected (DDC-002)

Story: [Review window: after a clean, only skips that can clear stay selected](../epics/DDC-002/stories/story-review-window-after-a-clean-only-skips-that-can-clear-stay-selected.md)

- [ ] Manual (owner): reproduce the recording's mix (a running app's cache, a folder changed after the scan, the Docker item with Docker Desktop quit), clean, and check what stays selected and what the result says; the sheet never shows an empty panel between its stages.

### Review window: Clean is not offered when every item would be skipped (DDC-002)

Story: [Review window: Clean is not offered when every item would be skipped](../epics/DDC-002/stories/story-review-window-clean-is-not-offered-when-every-item-would-be-skipped.md)

- [ ] Manual (owner): with Google Chrome running, select only the Chrome cache, open the sheet: the button is disabled and the line says to quit Google Chrome. Quit Chrome, Check Again: the button is enabled and the clean deletes it.

### Review window: say when the saved scan is not of the home folder (DDC-002)

Story: [Review window: say when the saved scan is not of the home folder](../epics/DDC-002/stories/story-review-window-say-when-the-saved-scan-is-not-of-the-home-folder.md)

- [ ] Manual (owner): after a terminal `ddc scan` of a folder outside the home, the Review window says so and Clean is off; Scan Now replaces the report and the notice goes.

### Settings: folders, types, notifications (DDC-002)

Story: [Settings: folders, types, notifications](../epics/DDC-002/stories/story-settings-folders-types-notifications.md)

- [ ] Manual check:
  - Settings opens from the gear, the right-click menu and ⌘,;
  - the tabs are toolbar tabs, and the window title is the pane's name;
  - the window resizes for each pane;
  - the app shows in the Dock while the window is open;
  - all of this in light and dark appearance.
- [ ] Manual check:
  - the folder picker adds an external volume;
  - Open Privacy Settings opens Full Disk Access;
  - a "Not readable" row opens Files and Folders.
- [ ] Manual check: Open at login reflects System Settings.

### Sparkle updates: check daily, Update and Relaunch, and the cask's auto_updates (DDC-002)

Story: [Sparkle updates: check daily, Update and Relaunch, and the cask's auto_updates](../epics/DDC-002/stories/story-sparkle-updates-check-daily-update-and-relaunch-and-the-casks-autoupdates.md)

See also the runbook [First Sparkle releases: shipping the updater, then proving it](./first-sparkle-releases-shipping-the-updater-then-proving-it.md).

- [ ] Manual (owner): `generate_keys` run once, the public key in Info.plist, the private key backed up offline and in no file of the repository or the vault.
- [ ] Manual (owner): on a Debug build against a local feed, **Update and Relaunch** updates a copy installed with the Homebrew cask layout and a hand-dragged copy; after each, the app relaunches at the same path, Full Disk Access still holds, the scheduler agent is registered again, Login Items still lists the app (the login item is not re-registered; it survives because `SMAppService.mainApp` is keyed to the bundle at the same path), and `ddc --version` prints the new version.
- [ ] Manual (owner): during a running scan, **Update and Relaunch** is disabled; a postponed install proceeds after the scan ends. A `ddc scan` started from Terminal during the install finishes normally.
- [ ] Manual (owner): on the first launch of the first Sparkle release, exactly one request is made; with the Settings switch off, no request is made for a day (Little Snitch, `nettop`, or the app log); **Check Now** makes one.
- [ ] Manual (owner): a Homebrew copy one release behind is updated by a plain `brew upgrade`; after a Sparkle update `brew upgrade` leaves the app alone, `brew upgrade --greedy deepdrivecleaner` reinstalls cleanly, and `brew uninstall --cask deepdrivecleaner` removes the app and both links.

### The Trash: its own row, not Your data (DDC-002)

Story: [The Trash: its own row, not Your data](../epics/DDC-002/stories/story-the-trash-its-own-row-not-your-data.md)

- [ ] Manual (owner): move something to the Trash with a clean, Scan Again: the Trash row has grown and is called Trash; Open the Trash opens Finder's Trash.

### User data: move a folder's picked files to the Trash from that folder (DDC-002)

Story: [User data: move a folder's picked files to the Trash from that folder](../epics/DDC-002/stories/story-user-data-move-a-folders-picked-files-to-the-trash-from-that-folder.md)

- [ ] Manual (owner): in Movies, pick one entry, press the button: the sheet holds that entry only and moves it to the Trash.

### VM images: Docker prune runs only while Docker Desktop answers (DDC-002)

Story: [VM images: Docker prune runs only while Docker Desktop answers](../epics/DDC-002/stories/story-vm-images-docker-prune-runs-only-while-docker-desktop-answers.md)

- [ ] Manual (owner): with Docker Desktop quit, select the Docker item: the sheet says it will be skipped because Docker Desktop is not running. Start it, Check Again, clean: both prune commands run and the result says what they reported.

### Whole disk: Conda environments and its package cache, with conda (DDC-002)

Story: [Whole disk: Conda environments and its package cache, with conda](../epics/DDC-002/stories/story-whole-disk-conda-environments-and-its-package-cache-with-conda.md)

- [ ] Manual (owner): on a Mac with conda, an environment shows in Python environments with its size; removing it in Review removes it from `conda env list`.

### Whole disk: free space with the owning tool (DDC-002)

Story: [Whole disk: free space with the owning tool](../epics/DDC-002/stories/story-whole-disk-free-space-with-the-owning-tool.md)

- [ ] Manual (owner): deleting an unused simulator runtime from the app frees its size, and `xcrun simctl runtime list` no longer lists it. Whether macOS asks for a password is recorded here.
- [ ] Manual (owner): Homebrew cleanup from the app shows the same list as `HOMEBREW_NO_AUTOREMOVE=1 brew cleanup -n` in Terminal (with no other `HOMEBREW_*` variable set in the shell), then frees about that much.

### Whole disk: free the checked areas in one sheet (DDC-002)

Story: [Whole disk: free the checked areas in one sheet](../epics/DDC-002/stories/story-whole-disk-free-the-checked-areas-in-one-sheet.md)

- [ ] Manual (owner): with two deletable runtimes, check both, free them in one sheet, and compare with `xcrun simctl runtime list`.

### Whole disk: Homebrew packages, their sizes, and brew uninstall (DDC-002)

Story: [Whole disk: Homebrew packages, their sizes, and brew uninstall](../epics/DDC-002/stories/story-whole-disk-homebrew-packages-their-sizes-and-brew-uninstall.md)

- [ ] Manual (owner): the Homebrew row lists the installed formulae with sizes; removing a leaf formula frees its Cellar size; a formula with two versions is removed in the sheet with both; its orphans are offered after, and removing them frees theirs.

### Whole disk in the app (DDC-002)

Story: [Whole disk in the app](../epics/DDC-002/stories/story-whole-disk-in-the-app.md)

- [ ] Manual (owner): the view opens from the popover and from the Review window, reads correctly in light and dark, and its bar matches the rows.
- [ ] Manual (owner): start a whole-disk scan, quit the app from its menu (not from Xcode), wait about a minute, and check that `last-disk.json`'s date moved on and it reads `"cancelled": false`. This shows that macOS does not stop the orphaned CLI on quit.

### Whole disk: what lies outside the home folder (DDC-002)

Story: [Whole disk: what lies outside the home folder](../epics/DDC-002/stories/story-whole-disk-what-lies-outside-the-home-folder.md)

- [ ] Manual (owner): on the reference machine, `ddc disk` finishes in under 3 minutes. Each named area that is not partial is within 5% of a `du -skx` of the same paths taken the same day, less the `du` of the areas nested in it; a partial area is at most its `du`; and "not counted" is within 10 GB of the Data volume's consumed bytes minus everything counted.

### Coding agents in the Whole disk view (DDC-002)

Story: [Coding agents in the Whole disk view](../epics/DDC-002/stories/story-coding-agents-in-the-whole-disk-view.md)

- [ ] Manual (owner): on the reference machine, with a few Claude Code and Codex sessions open, the section lists them as live with their project folders. After a session is closed, Refresh no longer lists it, and the count of remaining transcripts rises by one.
- [ ] Manual (owner): the harness sizes in the section are within 5 % of `du -sk` on the same folders.
- [ ] Manual (owner): `ddc disk agents --json` finishes in under 10 s on the reference machine, and `ddc disk agents --json --no-sizes` in under 1 s. Write the two times and the entry count on this line when ticking it; the research note measured 5.1 s for about 124,000 entries without the Claude desktop folder.

### Whole disk: Show in Finder and the honest action for each row (DDC-002)

Story: [Whole disk: Show in Finder and the honest action for each row](../epics/DDC-002/stories/story-whole-disk-show-in-finder-and-the-honest-action-for-each-row.md)

- [ ] Manual (owner): on the Whole disk view, each row's button or its absence matches the story's table; Show in Finder opens the right folder at its usual path, not under `/System/Volumes/Data`.
- [ ] Manual (owner): in the Coding Agents section, Show in Finder on a harness and on a project folder visibly selects it, including a hidden folder such as `~/.claude` while Finder hides hidden files. If nothing is visibly selected, open a bug story.

### Review window: Deselect All, then pick across groups and clean only that (DDC-002)

Story: [Review window: Deselect All, then pick across groups and clean only that](../epics/DDC-002/stories/story-review-window-deselect-all-then-pick-across-groups-and-clean-only-that.md)

- [ ] Manual (owner): in 0.9.0, Deselect All clears every group's checkboxes and reads "0 items selected"; after ticking one row in each of two groups, "Clean 2 items…" lists only those two. The button is greyed out while the confirmation sheet is open.

### Review window: sort a group's rows by name, size or last change (DDC-002)

Story: [Review window: sort a group's rows by name, size or last change](../epics/DDC-002/stories/story-review-window-sort-a-groups-rows-by-name-size-or-last-change.md)

- [ ] Manual (owner): in 0.10.0, in Python environments, click Changed: the oldest environments come first; click again: the newest; in Items the same header works; quit and reopen: the sort is kept.
- [ ] Manual (owner): with keyboard navigation on (System Settings → Keyboard), Tab reaches Name, Size and Changed with a focus ring and Space sorts; VoiceOver reads the sorted column as "Changed, sorted oldest first" and a row without a date as "No date".
- [ ] Manual (owner): with Region set to United Kingdom (System Settings → General → Language & Region), a date such as "25 Sept 2026" fits the Changed column without being cut; if the app ever shows Russian month names with a Russian system language, "25 сент. 2026 г." too.

### Protected by you: say what it is, and Remove Protection (DDC-002)

Story: [Protected by you: say what it is, and Remove Protection](../epics/DDC-002/stories/story-protected-by-you-say-what-it-is-and-remove-protection.md)

- [ ] Manual (owner): in 0.11.0, protect a folder from the Review window, scan, then Remove Protection… on its row in "Protected by you": the confirmation names the folder and the rule that would claim it, the row says "Protection removed. Scanning…", and after the scan the folder is in its own group and can be cleaned there.
- [ ] Manual (owner): on a row that holds a folder skipped in Settings, Remove Protection… removes that entry from Settings → Folders, and the next scan lists the row's folder in its own group, the skipped subfolder inside it. Both Protect… (on another group's row) and Remove Protection… open their confirmations.
- [ ] Manual (owner): protect a folder inside a git checkout and commit its `.ddc-keep`: Remove Protection… warns that git tracks it. With the marker uncommitted, no warning. With no git at the four fixed paths (a Mac without the developer tools), no warning and no install dialog.

### Harness data rules (DDC-003)

Story: [Harness data rules](../epics/DDC-003/stories/story-harness-data-rules.md)

- [ ] Manual (owner): on the reference machine, `ddc scan ~` lists each confirmed coding-agent path under "Coding agents", and `python3 -I bench/check_baseline.py` passes.
- [ ] Manual (owner): in 0.11.0, run `ddc scan`, then `ddc clean --items` with one stale Claude Code session's `<s>.jsonl` from the report: the transcript and its companions are in the Trash; after Put Back, `claude --resume` of it works. (Since 0.12.1 the Review window picks sessions too: story "Review window: pick a coding agent's sessions one by one".)
- [ ] Manual (owner): with Codex (or its app-server) running, a clean of the Codex folders is refused "Codex is running"; after quitting it, they move to the Trash.

### Review window: coding agents' data as a group (DDC-002)

Story: [Review window: coding agents' data as a group](../epics/DDC-002/stories/story-review-window-coding-agents-data-as-a-group.md)

- [ ] Manual (owner): in 0.11.0, the Review window shows "Coding agents" under Safe and under Review with sizes; cleaning a scratch row frees it. Since 0.12.1 Claude Code's transcripts and file-history folders are locked "Not cleaned whole · pick its sessions one by one" and open to their sessions (story "Review window: pick a coding agent's sessions one by one").
- [ ] Manual (owner): with Codex (or opencode, cursor-agent) running, select its folder under Review → Coding agents and clean: the check or the result says "<Harness> is running" and nothing moves. Quit it and clean again: the folder goes to the Trash.
- [ ] Manual (owner): in the Whole disk view, the Coding Agents section shows "In Review: …" and its Review… link opens the Review window on the Coding agents group: with the window already open it keeps what was selected; closed, it opens with the default selection.

### Review window: pick a coding agent's sessions one by one (DDC-002)

Story: [Review window: pick a coding agent's sessions one by one](../epics/DDC-002/stories/story-review-window-pick-a-coding-agents-sessions-one-by-one.md)

- [ ] Manual (owner): in 0.12.1, open Review → Coding agents, unfold a project's transcripts folder, pick one stale Claude Code session and clean it: its transcript and companions are in the Trash; after Put Back, `claude --resume` of it works.
- [ ] Manual (owner): with a Claude Code session running, its row says why it is locked and cannot be ticked; Select all never ticks a session; the folder row itself cannot be ticked.
- [ ] Manual (owner): the confirmation for picked sessions offers no Delete permanently and says that `--resume` of a transcript fails until it is put back from the Trash, and that /rewind cannot use moved file history.

### App skeleton and build pipeline (DDC-002)

Story: [App skeleton and build pipeline](../epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md)

- [ ] Manual (owner): Finder (`/Applications`, Get Info) and a notification the app posts (a scheduled scan's) show the app's icon, the drive in its ring of class colours, not a generic one. Login Items is checked under "Background scan and notifications in the app"; the status glyph in light and dark menu bars under "Menu bar popover".
- [ ] Manual (owner): with no window of the app open, it shows its status item and no Dock icon; Quit in the right-click menu exits it. (The Dock icon while a window is open is checked under "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open".)
- [ ] Manual (owner): turning Settings → Open at login on lists DeepDriveCleaner in System Settings → General → Login Items & Extensions; turning it off removes it. That the switch follows a change made in System Settings is checked under "Settings: folders, types, notifications".
- [ ] Manual (owner), Debug build only: Debug → Probe Full Disk Access writes its result and errno to the app log, and Debug → Post Test Notification posts one. If neither is wanted any more, remove both from `app/DeepDriveCleaner/DebugMenu.swift` instead.

### Distribution: the 0.2.0 cask with the app (DDC-002)

Story: [Distribution: the 0.2.0 cask with the app](../epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md)

- [ ] **Most important open check.** Clean-machine access check on the owner's work laptop, where Terminal has no Full Disk Access:
  - `brew install --cask smartandpoint/tap/deepdrivecleaner`, then open the app from Finder;
  - before any grant, the popover's footer shows Full Disk Access missing;
  - grant it to DeepDriveCleaner.app only (System Settings → Privacy & Security → Full Disk Access), then Scan Now: `~/Library/Logs/DeepDriveCleaner/app.log` and the report show `full_disk_access: true`;
  - `brew reinstall --cask deepdrivecleaner`, which removes the app and copies it again, then open it: the grant still holds and a Scan Now still reports `full_disk_access: true`.

  If the grant does not survive the reinstall, open a bug story: Rationale 1 of the ADR "Menu bar app: Swift shell over the ddc CLI" and the cask are revised.
- [ ] Manual (owner): update the Command Line Tools (Homebrew asked for those of Xcode 27 on 2026-10-31), then `brew audit --cask --online --strict smartandpoint/tap/deepdrivecleaner` reports no offenses.
- [ ] Manual (owner): on the reference machine, `brew reinstall --cask deepdrivecleaner` (it quits the app), open the app again: Full Disk Access still shows granted, and a Scan Now reports `full_disk_access: true`.
- [ ] Manual (owner): `brew uninstall --zap --cask deepdrivecleaner`, then `launchctl print` of `gui/$UID/com.smartandpoint.deepdrivecleaner.scheduler`, `…scheduler-open` and `gui/$UID/com.smartandpoint.deepdrivecleaner` (the 0.1.0 agent) finds no job. Reinstall afterwards.
- [ ] Manual (owner): on a second Mac or a fresh user account, download `DeepDriveCleaner-<version>.zip` from the release page, unzip it and open the app: no Gatekeeper warning; `spctl --assess --type execute -vv DeepDriveCleaner.app` says accepted, `source=Notarized Developer ID`.

### HIG pass: keyboard, colour, empty states and menus (DDC-002)

Story: [HIG pass: keyboard, colour, empty states and menus](../epics/DDC-002/stories/story-hig-pass-keyboard-colour-empty-states-and-menus.md)

- [ ] Manual (owner): on the second Mac, where the bug was seen, with a terminal frontmost, the first click on the icon opens the popover active, with blue switches and a blue Clean Selected. The `app.log` line after the click says the app is active and the window key.
- [ ] Manual (owner), keyboard in the Review window:
  - Esc in a non-empty filter clears it and the window stays open; Esc with an empty filter closes the window and opens the popover;
  - Return while typing a filter opens nothing; with an empty filter it opens the Clean sheet;
  - during a clean, Esc shows "Stopping…";
  - ⌘F focuses the filter;
  - with keyboard navigation on, Tab moves through the popover with a visible focus ring.
- [ ] Manual (owner), colour: with a red system accent (System Settings → Appearance), the Review footer's "Clean N items…" is red; with Multicolor it is the brand blue. Warning text is readable in Dark Mode.
- [ ] Manual (owner): Help → DeepDriveCleaner Help opens the README; with Reduce Motion on (System Settings → Accessibility → Display), the menu bar arc does not turn during a scan.

### Menu bar popover (DDC-002)

Story: [Menu bar popover](../epics/DDC-002/stories/story-menu-bar-popover.md)

- [ ] Manual (owner): with no report (move `~/Library/Application Support/DeepDriveCleaner/last-scan.json` aside, and put it back after), the popover offers the first scan.
- [ ] Manual (owner): with Full Disk Access turned off for the app, the footer says it is missing and its button opens the Privacy pane. Turn it on again.
- [ ] Manual (owner): the icon reads in a light and a dark menu bar in every state: idle, busy, warning, paused, paused while busy, and with the badge; the popover reads in Dark Mode.
- [ ] Manual (owner): Pause Scans in the right-click menu greys the icon and turns Scan Now off there and in the footer, which shows "Scans paused" with Resume; Resume restores both.

### Release 0.2.1: the Review window in the Dock, and skip what a running process holds open (DDC-002)

Story: [Release 0.2.1: the Review window in the Dock, and skip what a running process holds open](../epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md)

- [ ] Manual (owner): with the Review window open, the app is in the Dock and in Cmd-Tab; the same with only Settings or only the About panel open (Settings alone is also under "Settings: folders, types, notifications"). Close the last one with its close box or ⌘W: the app leaves the Dock and Cmd-Tab, and the app that was in front before is active again with its own menu bar.
- [ ] Manual (owner): the popover opens from the menu bar icon both with a window of the app open and with none. (A Dock click bringing the Review window forward is under "Background scan and notifications in the app"; the Close button returning to the popover under "Review and clean window".)
- [ ] Manual (owner): with a window of the app open, ⌘Q quits, ⌘W closes the window, and ⌘C, ⌘V and ⌘A work in the Review window's filter field.
- [ ] Manual (owner): with `uv run` servers running, select `~/.cache/uv` with a few other items and clean: within seconds the uv cache is skipped as in use by `uv`, and the clean goes on with the rest. (The Docker check of this story is the one under "VM images: Docker prune runs only while Docker Desktop answers".)

### Review and clean window (DDC-002)

Story: [Review and clean window](../epics/DDC-002/stories/story-review-and-clean-window.md)

- [ ] Manual (owner): on the reference machine, clean a few selected `safe` items from the window: the result shows free space before and after, and `~/Library/Logs/DeepDriveCleaner/actions.jsonl` lists exactly those items.
- [ ] Manual (owner): search, sort, the Groups and Items toggle, Check Again in the sheet and Show in Finder each work.
- [ ] Manual (owner): Stop during a clean of several items stops after the current one, and the result lists what was done.
- [ ] Manual (owner): the window's Close button returns to the popover, which stays open after the window is gone.

### Review window: clean only the selection in this group (DDC-002)

Story: [Review window: clean only the selection in this group](../epics/DDC-002/stories/story-review-window-clean-only-the-selection-in-this-group.md)

- [ ] Manual (owner): with rows selected in "Browser and app caches" (called "App caches" in the story) and in Python environments, the "Clean N from Browser and app caches…" button on the group's Select all row opens a sheet listing only that group's items; after the clean the Python environments are still on disk and still selected.

### Review window: sorted sidebar, Select all and folder groups (DDC-002)

Story: [Review window: sorted sidebar, Select all and folder groups](../epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md)

- [ ] Manual (owner): the sidebar's class tints read in light and dark mode.
- [ ] Manual (owner): a subgroup heading collapses and expands at once; Select all and a heading's checkbox tick and untick their rows.
- [ ] Manual (owner): a group of about 500 items opens and scrolls without visible lag. Write the group and its item count on this line when ticking it.

### User data: pick your own files for the Trash (DDC-002)

Story: [User data: pick your own files for the Trash](../epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash.md)

- [ ] Manual (owner): pick a file in Downloads that you can spare, clean it: it is in the Trash, and Put Back returns it to Downloads.
- [ ] Manual (owner): in Pictures, Photos Library is listed locked, with the app that manages it as the reason.
- [ ] Manual (owner): User data → Cloud folders says why they are not scanned and points to Remove Download in Finder.

### Linux scanner and rule catalog (DDC-007)

Story: [Linux scanner and rule catalog](../epics/DDC-007/stories/story-linux-scanner-and-rule-catalog.md)

- [ ] Manual (owner): on a clean Ubuntu VM with the released archive, `ddc scan ~ --json` on a seeded home finds the catalog's caches and project artifacts, and the report validates against `docs/scan.schema.json`.
- [ ] Manual (owner): `ddc clean --items` of an item on a second, writable volume moves it to that volume's `.Trash-$uid` (its `files` and `info`); on a volume whose root you cannot write, the item lands in `~/.local/share/Trash` by a copy, as the safety ADR's §5 Linux text says.

## Verification

- [ ] A passed check is ticked and stamped with the date and the version it was checked on, for example `- [x] Manual (owner): … — checked 2026-11-02 on 0.7.0`.
- [ ] A failed check stays unticked, with a link to the bug story it opened.
- [ ] A story's section is removed once all its checks are ticked; the commit that removes it records them.

## Rollback

Not applicable: the checks only observe the app and change nothing in the vault or the code.

## Common Issues

Some checks change the disk (a clean, a Trash move, a runtime delete, a formula uninstall): run them on items you can spare.

## References

- [[epics/DDC-002/epic|DDC-002: Menu bar app]]

---

*Last updated: 2026-10-31*
