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

- [ ] The current release, 0.7.0, installed through Homebrew on the reference machine.
- [ ] Full Disk Access granted to DeepDriveCleaner.app, and the app started from Finder, not from Terminal or Xcode.
- [ ] For a check that needs a tool or an app (Docker Desktop, Google Chrome, conda, Xcode simulator runtimes, Homebrew formulae): that tool installed.

## Steps

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

*Last updated: 2026-10-30*
