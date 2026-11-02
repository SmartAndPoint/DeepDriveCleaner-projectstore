---
type: research
slug: "what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app"
title: "What grew since the last scan: a UX concept for the macOS app"
status: draft
date: 2026-11-02
authors: ["Evgenii Konev"]
tags: [app, macos, ux, growth, disk]
review_status: pending
reviewed_at: null
---

# What grew since the last scan: a UX concept for the macOS app

## Question

On 2026-11-02 the owner's home folder grew by about 200 GB in one day, mostly cargo `target/` folders built by coding agents in worktrees. The app showed the folders, but as plain rows, not as growth, and no number it puts forward moved. The owner asked (2026-11-02): show explicitly what grew since the previous scan and by how much, attributed to a folder, a program or tool, or a cache; and notice cheaply that used space changed at all (like `df`, or a fast scan of ours rather than a full `du`). Designed from the person's point of view, macOS first.

## TL;DR

Today the CLI keeps exactly one scan report. Every writer replaces `last-scan.json`: a scan, a scheduled scan, the rescan a terminal `ddc clean` runs before cleaning, and the edits `ddc clean --items` and `ddc disk free` make afterwards. So there is nothing to compare against. The only "growth" rule in the product is the notification's repeat rule, and it tracks *reclaimable* bytes, not used space.

That gap matters more than it looks, because of how the owner's 200 GB day played out. New cargo `target/` folders in active worktrees are rebuildable, and rebuildable is selected by default only after `min_age_days` (14) idle days, or 2 days for an agent's worktree. So the popover's big number ("Selected to clean") barely moves. The threshold notification follows that number, so it stays quiet too. The new folders show up only as unticked rows inside the "found" total. The app saw the space, but no number it puts forward changed. **(inferred from the selection rules in README "`ddc scan`" and `crates/ddc-cli/src/notify.rs`; the owner's report was not read)**

The concept has three parts:

1. **A cheap signal.** A small CLI command, `ddc space --json`, reads the startup volume's free space with `statfs` (one syscall), compares it with the free space the last saved report recorded, and decides whether in-use space grew past a threshold. The app calls it on the hourly tick it already has, on wake, when the popover opens, and every ten minutes while it runs. When it fires, the popover says "198 GiB more in use than at the last scan" and the menu bar icon shows its badge. The rule lives in Rust, as every rule does.
2. **Attribution.** Each write of `last-scan.json` first moves the old file to `previous-scan.json`. A new command, `ddc changes`, compares the two reports and says where the growth went: by tool or group ("Rust build folders +186 GiB, 14 new, 9 in Claude Code worktrees"), by folder for growth that is not a finding, and, for the rest, "in folders ddc did not read, or outside your home folder", naming those folders. It compares only what both scans read and what rules both had.
3. **Surfaces.** A band in the popover, a "Grew since …" entry in the Review sidebar (the same rows, ticks and reasons, plus a Change column), a `ddc scan` text block, and later a Change column in Whole disk and a growth notification. Each of them leads to the Review or Whole disk actions that exist today. None of them cleans, and none ticks a row for the person.

The first release ships the signal and attribution together. The signal goes out first as its own pull request, because it also checks on a real Mac what `statfs` reports.

## Method

- A designer pass (a subagent briefed as a senior macOS product designer) read the code on `main` (the CLI's store, report model, notify rule, disk report; the app's popover, Review, Whole disk, scheduling and notifications), the vault's DDC-002 stories and ADRs, and public sources on APFS free space, FSEvents and how other tools compare scans. Nothing was built or run. Its marks: **(read)** means checked in code or docs, **(inferred)** means reasoned and not checked.
- The session checked the main claims against the code (one saved scan, replaced each time; volume free space already in each report; the notify rule on reclaimable bytes) and wrote the joint verdict.
- A critic pass (`projectstore:critic`) found one blocker and ten other points: "home vs elsewhere" mixed unread folders into "outside"; the list of writers of the saved report was incomplete; the first version did not yet make anyone *see* the change; the rule sat in the app instead of Rust; rule-set changes and the Trash could read as growth; a simpler storage design was not weighed. The session decided each point. This version applies them, with every code claim checked again on `main` at `018060d` (still the head on 2026-11-02 after a fetch).
- The owner's answers to the Review window concept bind this one: Whole disk stays useful and must be able to remove what it finds; the group action lives in the group header; "All Items" replaces the Groups/Items switch.

## Findings

### Today: what the app and the CLI keep and show

All on `main` at `018060d`.

| What | Where | Bears on growth how |
|---|---|---|
| One saved report, replaced atomically by every writer | `store::write_last_scan` (`crates/ddc-core/src/store.rs`). Its five callers: `ddc scan` (`crates/ddc-cli/src/cmd/scan.rs:48`); the scheduled scan (`cmd/background.rs:117`); a terminal `ddc clean`, which rescans and writes *before* cleaning and never after (`cmd/clean.rs:90`, its only write); `ddc clean --items`, which edits the report *after* the clean (`cmd/clean_items.rs:175`); `ddc disk free` after `brew cleanup`, which measures the Homebrew cache finding again (`crates/ddc-core/src/free/mod.rs:358`) **(read)** | **No previous report is kept.** Hooking one place, `write_last_scan`, covers every writer. |
| The edit after `clean --items` | `crates/ddc-core/src/report/after_clean.rs`: removes cleaned findings, moves trashed bytes into the `trash` finding of their volume (`trash_on`, `remove`), and sets each volume's `free_bytes` to the measured `free_after` (`finish`). It keeps `totals.scanned_bytes` true (`drop_scanned` and `resize` subtract or re-add what changed; trashed bytes stay, since they are still on disk in the `trash` finding), but it does not change `generated_at` **(read)** | The edited report is the right baseline for findings and its home total, but its time is stale. Its free space is fresh, from a later moment than `generated_at`. The Trash grows by what was trashed. |
| A terminal `ddc clean` | Writes the pre-clean rescan, then cleans, and leaves the cleaned items in the saved report until the next scan (`cmd/clean.rs`) **(read)** | Without a change, the next comparison would read those items as "gone" with no clean to explain them. |
| Per-report volume free/total | `ScanReport.volumes: [Volume {mount_point, roots, free_bytes, total_bytes}]`, `free_bytes` = "bytes available to an unprivileged user" via `statfs` (`model.rs:110`, `report/volumes.rs`); the app decodes it (`ScanReport.swift:126`) **(read)** | The signal's baseline exists already. |
| Findings | `Finding {path, rule, category, class, bytes, shared_bytes, private_bytes, dev, ino, project, origin {harness, kind}, modified, worktree, ignored, selected, not_selected}` (`model.rs:155`) **(read)** | Enough to match findings across reports and to name the tool (rule → group) and the harness (`origin.harness`: Claude Code, Codex, opencode, Cursor, git). |
| Totals | `Totals {scanned_bytes, unowned_bytes, classes, selected_bytes, selected_reclaimable_bytes, …}` (`model.rs:82`); `scanned_bytes` is "allocated bytes under the scan roots", so it counts only what the walk read **(read)** | Home growth, but only where both scans read the same folders (below). |
| What the walk did not read | `skipped: [{path, reason}]` and `skipped_total`; `not_scanned` (the user's skips only, "never truncated"); `inaccessible` (at most 10 000) and `errors_total`; `full_disk_access`; `cancelled` (`model.rs:30-78`). How each kind lands: the user's skips go to `not_scanned` *and* to `skipped` as `excluded` (`rules/visitor.rs:239-243`); the scheduled scan's prompting areas without Full Disk Access (`PROMPTING_AREAS`, `cmd/background.rs:26`, passed as `ScanOptions.skip`) go to `skipped` as `excluded` only (`visitor.rs:247`, `scan/walker.rs:271` → `skip`); cloud folders (`Library/CloudStorage`, `Library/Mobile Documents`, never traversed, `rules/catalog/macos.rs:234`) go to `skipped` as `excluded`; folders macOS refused go to `inaccessible`. `skipped` is cut to the shortest paths once it doubles `max_recorded` (`walker.rs:600-602`) **(read)** | The app already tells these apart: `ReviewItems.cloudSkipped` and `backgroundSkipped` (`ReviewItems.swift:262-273`) filter `skipped` by `excluded`, by the cloud paths and by `not_scanned`. A comparison can build "not read" from the same fields. |
| Per-folder sizes outside findings | Only `contents` of the user-data folders (`model.rs:388`) **(read)** | **Missing:** "~/Movies grew by 30 GiB" cannot be said today. |
| The rules in force | Not in the report; only `ddc_version` and each finding's `rule` **(read)** | **Missing:** a rule new in this version cannot be told from "nothing matched before". |
| The notify decision | `crates/ddc-cli/src/notify.rs`: threshold on `selected_reclaimable_bytes`, a repeat after ≥ 20 % more of that or a week, low disk under 10 % free with something selected on the volume; `notify-state.json` holds `last_at`, `last_bytes`, `snooze_until`; `notify ack --bytes` writes the first two (`cli.rs:193`) **(read)** | The one growth rule, on reclaimable bytes. Active builds do not move it. |
| Notification text and the badge | `NotificationText.make`: "<amount> can be freed" or "<disk> is N% full" (`NotificationText.swift`). A posted notification, or `afterScheduledScan == .menuBarOnly`, sets the badge (`ScheduledWork.swift:144-147`, `PopoverModel.notificationPosted`). It shows when `ShowMenuBarBadge` is on (`AppPreferences.swift:12`, default on), and clears when Review opens or on Remind Me Tomorrow (`PopoverModel.swift:593, 644`) **(read)** | The badge already exists, needs no notification permission, and the person can switch it off. |
| Only scheduled scans notify; rules stay in Rust | ADR "Menu bar app: Swift shell over the ddc CLI" §5 "Notifications" ("The notify decision rule stays in Rust") **(read)** | A growth rule belongs in the CLI. A growth notification needs a §5 amendment. |
| The hourly tick; the minute timer; wake | LaunchAgent `open -gjb … --scheduled` on `StartCalendarInterval {Minute: 0}` (ADR §5); a 60 s timer that calls `PopoverModel.refresh()` (`StatusItemController.swift:90`); refresh on activate (`AppDelegate.swift:219`); no wake handler and no free-space call in the app (grep over `app/`) **(read)** | Free places to call `ddc space`; wake is new. |
| The popover while paused | "Scans paused" with a Resume link (`PopoverView.swift:123-124`) **(read)** | The growth band need not repeat it. |
| Whole disk | `last-disk.json`, replaced by every `ddc disk`; `DiskReport {container.in_use_bytes, home {source last-scan or walk, as_of}, areas[] {id, bytes, partial, stale}, snapshots.count, not_counted}` (`disk/model.rs`). The container comes from `diskutil apfs list -plist`, with `statfs` per mounted volume as the fallback (`disk/apfs.rs`). Snapshots: a count only; "their size cannot be measured without root" (`disk/model.rs:313`) **(read)** | Areas have stable ids, so two reports diff easily, but only one is kept. |
| Prior vault work | No story or research on growth over time (grep over DDC-001, DDC-002 and research) **(read)** | New ground. |

**What is missing to compute growth:** a kept baseline; a free-space reading between scans, with its rule in Rust; the rules in force and a writer and time on each report; per-folder sizes outside findings; a comparison that knows what both sides read; a growth reason in the notify rule; a kept baseline for `ddc disk`.

### The person's jobs and moments

| Moment | What the person asks | What they decide next |
|---|---|---|
| The disk filled up during a work day (the owner's case: agents built in many worktrees) | "Where did 200 GB go, and is it mine to keep?" | Remove the agents' worktrees' `target/` folders now, or leave them and know they will be selected once idle |
| A glance at the menu bar | "Did anything change since I last looked?" | Nothing (most days), or open the popover because of the badge |
| The popover | "How much, and where?" | See What Grew, or Scan Now when no scan has looked yet |
| A notification while busy (later) | "Is this urgent, or can it wait?" | Review now, or Remind Me Tomorrow |
| The Review window | "Which of the new things can go, and which are my live work?" | Tick and clean through the sheet; protect a folder; leave active builds alone |
| Whole disk | "Is it in my home folder, or somewhere I can't see?" | The owning tool's action (Delete Runtime…, Clean Up Homebrew…), Show in Finder, or Time Machine for snapshots |
| Right after a clean | "Did it free what it said?" | Already answered by the clean result; the growth view must not count the clean as growth or shrinking |
| A coding agent hits `ENOSPC` | "What did my builds just produce?" | Show the person `ddc changes --json`; never clean `rebuildable` itself |

The first moment is the one that failed. The person needs to *notice* (the badge), then **one sentence with a number and a name** ("198 GiB more in use since yesterday, mostly Rust build folders"), then **one click to the rows**.

### The cheap signal: noticing that used space changed

#### What to read

- **`statfs` on the home folder's volume** (the Data volume on macOS), the same reading the report stores (`f_bavail`, `report/volumes.rs`), so the signal and the scan mean the same "free". One syscall, microseconds, no permission prompt, no walk.
- **What it counts.** `df` and `statfs` report only unconditionally free space: purgeable space and blocks held by local snapshots count as in use ([Eclectic Light, 2023-04-27: Where does macOS get its volume free space figures from?](https://eclecticlight.co/2023/04/27/where-does-macos-get-its-volume-free-space-figures-from/), the author's replies). That is what we want for an alarm: it never says "free" for space the next write cannot have. It also means a deleted file held by a snapshot frees nothing until Time Machine drops the snapshot, about a day later by the same source. Older notes disagree on whether `f_bavail` includes purgeable blocks, so **S5 checks it on a real Mac before the threshold is tuned.**
- "In use" = total − available. On APFS the volumes of a container share free space, so this counts the whole container: Data, VM (swap, sleep image), Preboot and local snapshots **(inferred from APFS's shared container)**. The sentence says "on the startup disk", never "in your home folder".
- `volumeAvailableCapacityForImportantUsage` counts purgeable space as available ([Apple: Checking volume storage capacity](https://developer.apple.com/documentation/foundation/checking-volume-storage-capacity)) and swings with macOS's own figures ([Eclectic Light, 2023-04-17](https://eclecticlight.co/2023/04/17/the-finder-confuses-with-wildly-inaccurate-figures-for-available-space/)). It is not used.

#### Where the rule lives: `ddc space`

Amended 2026-11-03: this section follows the story and the ADR's §6 item 13.

ADR "Menu bar app" puts rules in Rust and has the app read JSON. So the signal is a CLI command from the first version, `ddc space [--json] [--data-dir DIR]`. Its contract is settled in the ADR's §6 item 13 and the story "Popover: say when used space grew since the last scan":

- It reads `statfs` for the home folder's volume, reads the head of the saved report (`generated_at` and `volumes`, any version), applies the rule below, appends a sample to `space.json`, and prints the decision. It walks nothing, takes no lock and notifies nothing.
- `--json`: `{schema_version, checked_at, volume {mount_point, total_bytes, free_bytes}, baseline {source: scan | clean, at, free_bytes} | null, in_use_delta | null, grew, state: quiet | grew | low-disk}`.
- The baseline time is the volume's `read_at` when a `clean --items` edit read its free space again (the story adds that field), else the report's `generated_at`.
- "Once per saved report" for the badge is the app's (`GrowthBadgedFor`), not the CLI's, so a terminal call never consumes it. The settle rule waits for S6.
- `space.json`: `{schema_version, samples: [{at, mount_point, total, free}]}`, at most one sample an hour for 14 days plus the latest.
- A terminal user gets one sentence: "198 GiB more in use on the startup disk than at the last scan, 26 h ago. `ddc scan` shows where."

#### The rule

- **Baseline:** the saved report's `free_bytes` for the home volume, read at `read_at` after a clean edit, else at `generated_at`. After a `clean --items` edit, that reading is from the clean, later than the scan. The sentence names that time: "than after your clean at 15:20", or "than at the last scan (yesterday 10:04)" when the scan wrote it.
- **Grew:** in use grew by at least **max(20 GiB, 5 % of the volume)** since the baseline (defaults to confirm; open question 4). **Low disk** (under 10 % free) is its own state and wins.
- **Settled:** the latest two samples at least an hour apart differ by under 5 GiB. Only a later signal-triggered scan (S6) waits for it. The popover and the badge do not wait, and under 10 % free nothing waits.
- **Once per baseline:** the app badges the first time `grew` or `low-disk` holds for a saved report and records that report (`GrowthBadgedFor`); a new saved report re-arms it. A disk that keeps filling sets the badge once, not every hour.
- **Shrinking never fires.** More free space (a clean, a purge, a snapshot dropped) is not a reason to interrupt.

#### What the app does with it

| Option | Pros | Cons |
|---|---|---|
| A. The popover band, plus the menu bar badge | Visible without opening anything; no notification permission; no §5 amendment (the badge is not a notification); the person can switch it off (`ShowMenuBarBadge`) | Says how much, not where, until a scan |
| B. A background scan (`scan --background --notify json`, 3 threads), then a notification that says where | One notification with the answer | A new reason to scan outside the schedule (§5 amendment); the I/O and battery of a home walk |
| C. A notification at once, "scan to see where" | Fast | A half answer, then a second message |

**Recommendation: A in the first release, B later (S6).** The badge from the signal is its own flag beside today's `AttentionBadgePending` (the owner agreed, 2026-11-02). It **clears when the popover opens**, or when the baseline changes, because the popover band is where its message is. A notification's badge keeps its own rule (it clears when Review opens). VoiceOver reads "used space grew" for the signal's badge, beside today's "new results to review" (`IconState.swift:77`). B, when it comes: at most one signal-triggered scan per 6 hours, only once `settled`, never while paused, never during a clean (the lock refuses), as scheduled work under "Only notify during working hours". It never cleans: the background path never calls the executor (safety ADR §8), and the owner parked unattended cleaning on 2026-10-28.

**FSEvents** could say *which* folders changed since a stored event ID, without a walk ([Apple: File System Events Programming Guide](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/Introduction/Introduction.html)). But it gives paths, not sizes, a cargo build floods it, and protected folders need Full Disk Access. Later at most, to choose what to rescan first.

#### False alarms and blind spots

| Cause | Effect | What the person is told |
|---|---|---|
| Local Time Machine snapshots hold deleted or rewritten files | In use stays high after a clean; part of the growth is in no folder | "Local snapshots may hold part of it. Whole Disk shows how many there are. Time Machine removes them by itself, usually within a day; ddc does not delete them." |
| A macOS update downloading, swap, the sleep image | Growth outside the home folder | The remainder line, with Whole Disk… |
| iCloud or another cloud folder downloading files | Real growth in the home folder that no scan reads (cloud folders are never traversed) | The remainder line names "cloud folders" |
| Docker Desktop's disk image, under `~/Library/Containers` (inferred path) | A scheduled scan without Full Disk Access does not read it | The remainder line names it with Open Privacy Settings |
| Purgeable space reclaimed | Free space jumps up | Nothing; shrinking never fires |
| A scan while a build writes | Numbers drift during the walk | Rows say "≈"; S6 waits for `settled` |
| What `f_bavail` counts on APFS | Threshold off by the purgeable amount | Checked on a real Mac in S5 |

### Attribution: saying where the growth went

#### Three levels, in this order

1. **The disk:** "198 GiB more in use on the startup disk since yesterday 10:04." From the two free-space readings.
2. **Your home folder, only when it compares.** "191 GiB of it in your home folder" is said only when both reports read the same places: the same `scan_roots`, the same `full_disk_access`, the same not-read set (below), neither `cancelled`, `skipped` not cut on either side (`skipped_total` equals `skipped.len()`). Otherwise the home total reads **"Home total not compared: the two scans read different folders"**, and only group deltas are shown.
3. **The rest, named for what it is:** disk delta − home delta (when the home compares) or − Σ finding deltas (when it does not) is **"In folders ddc did not read, or outside your home folder"**. Under it, the not-read places of the *current* report, each with the action that fits:
   - cloud folders (`skipped`, `excluded`, under `Library/CloudStorage` or `Library/Mobile Documents`): no action, and the cloud warning the Review window already has;
   - the scheduled scan's prompting areas (`skipped`, `excluded`, neither cloud nor under `not_scanned`, exactly `ReviewItems.backgroundSkipped`'s test): "Scan Now to read them", and Open Privacy Settings when `full_disk_access` is false;
   - your skips (`not_scanned`): Settings…;
   - places macOS refused (`inaccessible`, collapsed to their prompting area as `NotReadable.areas` does): Open Privacy Settings;
   - outside the home folder: Whole Disk…, and the snapshots sentence when `ddc disk` counted any.

Inside the home folder, by what it is:

- **By tool or kind** (the default): the Review group name from `rules --json` ("Rust build folders", "Node packages", "Xcode DerivedData", "Docker"). The app's existing words.
- **By harness**, as a second fact on the group line: "9 in Claude Code worktrees" from `origin.harness`.
- **By folder**, for growth that is no finding: "~/Movies/Exports +2.9 GiB", once the report carries folder sizes (S3). Until then one line: "+3.0 GiB in folders that are no finding".
- **Caches** are groups like any other ("Package caches +4.1 GiB").

#### Matching two reports

- **Identity is `(path, rule)`**, what the person sees. `(dev, ino)` changes when cargo deletes and re-creates `target/`, so it is not the key.
- **Moved:** a `gone` and a `new` with the same `(dev, ino)` and rule are one `moved` row, with only the size change counted (a renamed worktree).
- **Not read:** the not-read set of a report is `not_scanned` ∪ every path in `skipped`, whatever its reason (`excluded`, `dataless`, `other-filesystem`, `changed`, `automount`) ∪ the `inaccessible` paths. The reasons only choose the label shown (cloud, prompting area, your skip, "evicted cloud content", "another disk", "changed while read"). A finding of one side whose path is at or under a not-read path of the *other* side is `not-compared`, whichever way it went. So a Downloads folder read for the first time is not "+30 GiB new", and a skip removed since the last scan does not make its findings "new".
- **Rules in force:** each report records the rule ids in force (`rules_in_force`, new; disabled types stay in force, since their findings are reported as `ignored`). A finding whose rule is absent from the baseline's list is `not-compared`: a rule added by an upgrade does not read as new space. `ignored` findings are compared like the rest and shown by the current settings, as `ReportSummary` does.
- **Cleaned and trashed:** from `actions.jsonl` between the two reports' `written_at`, `cleaned_since` has two numbers: deleted, and moved to the Trash. Findings gone that the journal names are "cleaned by you", never "gone". The `trash` finding's growth, up to the trashed amount, is "moved to the Trash by you", never growth; only the part above it (Finder deletes, other apps) is growth.
- **Rows:** a change of at least 100 MiB, or among the 20 biggest; the rest fold into "N smaller changes". Sizes are `bytes` (allocated), and rows with `shared_bytes > 0` keep the "≈" the Review concept gives them.

`ddc changes [--json]` prints it, under a new `docs/changes.schema.json`:

```
Changes {
  schema_version,
  baseline {written_at, written_by, generated_at},   // a volume's read_at: ADR §6 item 13
  current  {written_at, written_by, generated_at},
  disk  {mount_point, in_use_before, in_use_now, delta},
  home  {compared: bool, reasons: [different-roots | different-access | different-not-read | cancelled | skipped-cut],
         before?, now?, delta?},
  rest  {delta, not_read: [{path, kind: cloud | prompting | skip | refused | evicted | other-disk | changed}], snapshots?},
  cleaned_since {deleted, trashed},
  groups: [{group, delta, new, grew, shrank, gone, cleaned, by_harness: [{harness, delta, count}]}],
  items:  [{path, rule, kind: new | grew | shrank | gone | cleaned | moved | not-compared, before, now, delta, moved_from?}],
  folders?: [{path, delta}]                                 // after S3
}
```

It reads two files and walks nothing: a hash join over a few thousand findings, milliseconds **(inferred: finding counts not measured)**. The app calls it after each report load, queued like `rules`.

#### Which report is the baseline

- **The report as last written before this one.** After a `clean --items` edit, that is the edited report: a `target/` cleaned and rebuilt reads "new 80 GiB", not "shrank 20 GiB".
- Two Scan Nows in a row leave a baseline minutes old, and the band then says so ("No change since 10 min ago"). The "since yesterday / last week" picker comes with kept history (S8). Open question 1.
- A `cancelled` report never becomes the baseline: the rotation skips it and keeps the older one.

#### Edge cases

| Case | Behaviour |
|---|---|
| First scan, or a previous report of another schema | No band, no entry. `ddc changes`: "Growth shows after the next scan." |
| Different roots | Compare the findings under roots both share; "Compared in ~/Projects only: the earlier scan covered other folders". No shared root: no comparison. |
| A scheduled scan without Full Disk Access vs a Scan Now with it | The home total does not compare; the prompting areas are `not-compared`; group deltas only |
| A folder moved | One `moved` row, net change only |
| A clean between the scans | The baseline is the edited report; `cleaned_since` is its own line ("You cleaned 40 GiB since then; 12 GiB of it is in the Trash"), never growth |
| A terminal `ddc clean` | After S1 it edits the saved report after the clean, as `--items` does, so the next comparison sees what it removed |
| Scans paused | `ddc space` still runs (it is not a scan); the band shows the number; no signal-triggered scan |
| An upgrade added a rule | Its findings are `not-compared` until the next comparison of two reports that both have it |
| A type switched on, or `min_size` lowered | The findings were already in both reports as `ignored`; they compare normally |
| A skip removed | What lay under it is `not-compared` |
| The previous report replaced while `ddc changes` reads | The current report records `previous_written_at`. If the previous file's `written_at` does not match, read both again once, then say "A scan is saving; try again." |
| `--data-dir` | Its own two reports |

## Verdict

The designer's recommendation, revised by the critic pass and the session's decisions:

**First release (the signal and attribution, for findings):**

- **S5, its own pull request, first:** `ddc space` (the rule in Rust), `space.json`, the app's calls on the tick, wake, popover and timer, the popover band and the badge. It also checks what `f_bavail` reports on a real Mac.
- **S1:** `previous-scan.json` and the new report fields (writer, times, rules in force) and a terminal `ddc clean` that edits the saved report after cleaning.
- **S2:** `ddc changes` and the `ddc scan` text block.
- **S4:** the popover band names the top group, and the Review entry "Grew since …". It can land on today's Groups/Items switch (a sidebar entry works in Groups mode) and does not wait for the Review concept's "All Items" story.

**The simpler storage wins for now.** Two whole reports, `last-scan.json` and `previous-scan.json`, rotated in `write_last_scan`, instead of a digest format with retention. It holds up: every comparison the first release needs is between the last two writes, and nothing new has to stay in step with the report's schema. It needs two small additive report fields that a digest would have needed anyway (`rules_in_force`, the writer and times). It fails only at "since yesterday" after a second Scan Now, which the band then says plainly. Digests with retention come with the picker (S8).

**Later:** folder sizes for growth that is no finding (S3); the signal-triggered scan and the growth notification (S6, §5 amendment); Whole disk's Change column (S7); the picker and history (S8); FSEvents; a 14-day in-use sparkline in the popover, as iStat Menus keeps history graphs ([iStat Menus on the App Store](https://apple.co/2VlAxEw)).

**Cut:**

- A red/green growth tree in the style of TreeSize's compare view ([TreeSize: Disk Usage Comparison](https://manuals.jam-software.com/treesize/EN/disk_usage_comparison.html)). This app's unit is the finding and its group. A second tree would be a second job, the weight the Review audit just removed.
- "Select all that grew" across groups. It would act like a second default selection and tick active builds in use right now.
- Auto-clean on growth (parked by the owner, 2026-10-28: scans inform, people remove).
- Notifications about shrinking.
- A separate History window.
- Snapshot sizes: they need root, and `ddc` never asks for a password. Deleting snapshots stays Time Machine's.

For comparison: TreeSize Professional compares against saved scans or Windows shadow copies, red for grown and green for shrunk, and lists free space per snapshot to find when space went ([TreeSize docs](https://manuals.jam-software.com/treesize/EN/disk_usage_comparison.html)). DaisyDisk and WizTree do not compare scans as far as could be found ([FolderSizes vs WizTree](https://www.foldersizes.com/features/wiztree)). None of them ties growth to *what made it*, a tool or an agent. This app's rules already name the tool.

### Joint verdict (designer and session)

Agreed, with the critic's points applied. The owner asked to see *where* the growth went, and to know *that* it happened. The signal alone covers "that", through the badge and the band. The first release adds "where" for findings: S5 as its own pull request first, then S1, S2 and S4. S3, S6, S7 and S8 follow, each alone. Every rule (the signal's threshold, the comparison, a later growth notification) lives in the CLI, so the Windows app gets it with the CLI.

## Concept

The voice follows the app: short, plain, one number per fact. Sizes in GiB as `ByteSize.human` prints them; times as the app says them ("2 h ago", "yesterday 10:04"). The example below is one day on a 931 GiB disk: 239 GiB free at yesterday's scan, 41 GiB free now (95 % full, `NotificationText`'s rounding down). In use grew by 198 GiB: 191 GiB in the home folder (Rust build folders +186, Node packages +2.0, folders that are no finding +3.0), and 7 GiB in folders ddc did not read or outside the home folder.

### Menu bar icon

The badge appears when `ddc space` first says `grew` for this baseline (and `ShowMenuBarBadge` is on). It clears when the popover opens. No sound, no notification.

### Popover

The band sits between the header and the main block, shown only when there is something to say. It is information, not an error: secondary text with an up-arrow symbol. The warning ink stays for low disk.

After a scan that found growth (S4):

```
┌────────────────────────────────────────────────┐
│ DeepDriveCleaner                            ⚙  │
│ Last scan 2 h ago · 5.5 M files in 39.6 s      │
├────────────────────────────────────────────────┤
│ ↑ 198 GiB more in use since yesterday 10:04    │
│   Rust build folders +186 GiB · 14 new         │
│   See What Grew                                │  ← link: Review on "Grew since …"
├────────────────────────────────────────────────┤
│ SELECTED TO CLEAN                              │
│ 12.0 GiB                                       │
│ of 236 GiB found. Nothing is removed until     │
│ you confirm.                                   │
│ …                                              │
```

The signal fired and no scan has looked yet (S5):

```
├────────────────────────────────────────────────┤
│ ↑ 198 GiB more in use than at the last scan    │
│   (yesterday 10:04). A scan can tell where.    │
│   [Scan Now]                                   │
├────────────────────────────────────────────────┤
```

After a `clean --items` edit, the time is the clean's: "than after your clean at 15:20".

While scans are paused, the band shows only the number. The popover already shows "Scans paused · Resume".

```
│ ↑ 198 GiB more in use than at the last scan    │
│   (yesterday 10:04).                           │
```

Low disk (warning ink; low disk wins):

```
│ ⚠ The startup disk is 95% full: 198 GiB more   │
│   in use than at the last scan.  [Scan Now]    │
```

When most of the growth is in folders ddc did not read or outside the home folder: "↑ 40 GiB more in use since yesterday · most of it in folders ddc did not read or outside your home folder · Where Does the Rest Go?". The band names one group at most ("mostly …" when one group is at least 60 % of what the findings explain), or "in 3 groups". It never shows shrinking.

### Notification (later, S6)

```
┌──────────────────────────────────────────────────────┐
│ DeepDriveCleaner                                now  │
│ 198 GiB more in use since yesterday                  │
│ Mostly Rust build folders (186 GiB, 14 new).         │
│ Nothing is removed until you review them.            │
│                       [Review]  [Remind Me Tomorrow] │
└──────────────────────────────────────────────────────┘
```

With low disk, low disk keeps the title as today, and the growth goes into the body: "The startup disk is 95% full" / "41 GiB free. 198 GiB more in use since yesterday, mostly Rust build folders." When the growth is mostly in folders ddc did not read or outside the home folder: "40 GiB more in use since yesterday" / "Most of it is in folders ddc did not read or outside your home folder." with [Review] [Remind Me Tomorrow]. Review opens the Review entry, and its foot names those folders and Whole Disk….

S6 rules:

- `afterScheduledScan == .menuBarOnly` is honoured: no notification, only the badge, as `ScheduledWork.flushPending` does today.
- The decision is a new `growth` reason in `notify.rs`, in Rust.
- A growth notification is acknowledged with `notify ack --growth <written_at>`. That records only which report it was about (a new optional `growth_for` in `notify-state.json`) and leaves `last_at` and `last_bytes`, the reclaimable repeat baseline, untouched. One growth notification per baseline. Remind Me Tomorrow snoozes it as today.

### Review window

It fits the Review concept of 2026-11-02 ("Review window on macOS: a UX audit and a lighter concept"), and works in today's Groups mode too: the entry is one more sidebar row above the class sections.

```
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│ Review and Clean                            [⌕ Filter by path or rule   ]   [↻ Scan Again] │
│ Scanned 2 h ago · your home folder                                                        │
├────────────────────────────────┬──────────────────────────────────────────────────────────┤
│  Grew since yesterday +191 GiB │ Grew since yesterday 10:04                                │
│                                │ 198 GiB more in use on the startup disk: 191 GiB in your │
│  ● Safe                        │ home folder, 7 GiB in folders ddc did not read or outside │
│    Package caches  9.2 of 13.1 │ your home folder.                                         │
│  ● Rebuildable                 │ ──────────────────────────────────────────────────────────│
│    Rust build folders  …       │  Name                                Size    Change ▾    │
│  …                             │ ◩ Rust build folders · 14 new · 9 in Claude Code worktrees│
│                                │                                  [Clean 1 from Rust build folders…]
│  Not scanned (3)               │   ☐ ~/Projects/app/.claude/worktrees/a1/target 21.3 GiB new│
│                                │      Project changed 2 h ago · not selected               │
│                                │   ☐ ~/Projects/app/.claude/worktrees/b7/target 19.8 GiB new│
│                                │      Project changed 40 min ago · not selected            │
│                                │   ☑ ~/Projects/old-tool/target             3.1 GiB  +3.1 GiB│
│                                │   ▸ 11 more                                               │
│                                │ ☐ Node packages · 2 grew                   6.2 GiB  +2.0 GiB│
│                                │ ── Folders that are no finding ──────────────── +3.0 GiB │
│                                │     ddc does not clean these.                             │
│                                │     ~/Movies/Exports                       +2.9 GiB  [Show in Finder]
│                                │ ── In folders ddc did not read, or outside your home ── +7.0 GiB │
│                                │     Cloud folders: not read by any scan.                  │
│                                │     Downloads, Library/Containers: the scheduled scan     │
│                                │     could not read them.  [Scan Now] [Open Privacy Settings]
│                                │     Outside your home folder: Whole Disk…                 │
│                                │ ── Cleaned by you or gone (23) ▸                          │
├────────────────────────────────┴──────────────────────────────────────────────────────────┤
│ 39 items in 5 groups · 12.0 GiB · frees about 11.4 GiB     Deselect All  Close  [Clean 39 items…] │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

(Whole Disk… stays at the sidebar's foot in the Review concept's layout, or as today in Groups mode.)

How it behaves:

- **The sidebar number is the home growth the entry's rows add up to** (+191 GiB: 186 + 2.0 + 3.0). The disk number (198 GiB) stays in the popover and in the entry's header line. When the home total does not compare, the sidebar shows the findings' sum ("+188 GiB") and the header says "Home total not compared: the two scans read different folders."
- The entry is **a view over the same rows**, like "All Items": the same checkboxes, ticks, lock and not-selected lines, and the group action in the header. Ticking here ticks there. The footer and the sheet are unchanged: the sheet is still the only thing that starts a clean.
- **Nothing is ticked for the person.** Rows keep the default selection, and the footer counts every tick in the window, as today (39 items, the popover's 12.0 GiB). Today's fresh `target/` folders are in active projects, so they are unticked, with the reason inline ("Project changed 2 h ago · not selected").
- **The group checkbox is today's Select all, no more.** It appears only where Select all appears today (never on user data, never over sessions). It ticks the group's shown rows that can be ticked, skipping locked ones, as `toggleAll` does. In this entry, "shown" means the rows that grew, and that can include active builds the default leaves unticked. That is a person's click on rows they can see, not a selection made for them, the same as Select all in the group's own entry. It is said here because the entry makes such rows easy to reach.
- **Change column:** "new", "+3.1 GiB", "−1.2 GiB", "moved". Sorted by Change by default in this entry.
- **Rows that are no finding** have no checkbox, only Show in Finder, under "ddc does not clean these". They can never reach the sheet.
- **Cleaned or gone** rows fold away, with no checkbox.
- No growth: no entry. First scan: no entry.

### Whole disk (later, S7)

Whole disk keeps its job and every action button. It gains a Change column against the previous whole-disk report (`previous-disk.json`, rotated like the scan report). Its baseline is that report's date, which can differ from the popover's, so the header names it.

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ Whole disk                                                   [Scan Whole Disk]       │
│ Measured 3 h ago · compared with the whole-disk scan of Oct 28                       │
│ [████ home ████████████|██ outside ██|░ not counted ░|▒ system ▒|  free  ]           │
│                                                       Size      Change               │
│  Home folder                                        612 GiB   +193 GiB  See What Grew │
│  Applications                                        48 GiB    +1.0 GiB [Show in Finder]
│  Simulator runtimes                                  31 GiB         —   [Delete Runtime…]
│  Homebrew                                            12 GiB    +2.2 GiB [Clean Up Homebrew…]
│  Other areas outside the home folder                101 GiB    +0.8 GiB              │
│  Not counted                                         44 GiB    +4.0 GiB              │
│    12 local snapshots exist. Time Machine removes them by itself; ddc does not.      │
│  System and other volumes                            42 GiB    −1.0 GiB              │
│  Free                                                41 GiB    −200 GiB              │
│                                                     931 GiB                          │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

The rows add up to the disk (612 + 48 + 31 + 12 + 101 + 44 + 42 + 41 = 931 GiB), and the changes to the free space's (193 + 1.0 + 2.2 + 0.8 + 4.0 − 1.0 = 200). An area `partial` or `stale` on either side shows "—". Whole disk still never runs by itself; the band only links to it.

### Terminal

`ddc scan` prints one block after its summary when a comparison exists:

```
Since the last scan (yesterday 10:04): 198 GiB more in use on the startup disk.
  Rust build folders     +186 GiB   14 new, 9 in Claude Code worktrees
  Node packages            +2.0 GiB
  No finding               +3.0 GiB
  Not read, or outside ~   +7.0 GiB   cloud folders; `ddc disk` for outside
`ddc changes` lists every row.
```

**Coding agents.** `ddc changes --json` is read-only, and the agent guidance may use it to show the person what grew. What an agent may do with it is bounded by ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §7 (2026-11-02): an agent never removes `rebuildable` items, its worktrees' `target/` folders included. It hands the person the list, with the `ddc clean --items` command or the Review window. Only `safe` items in its own tree, behind the Settings opt-in, are its to clean. `ddc changes` lists the whole home folder, so the guidance names only what the person asked about. For one project, `ddc agent footprint` stays the scoped command.

### How it links to action, and what it never does

| From | Leads to | Never |
|---|---|---|
| Badge | The popover | Notifies, sounds |
| Popover band | See What Grew → Review entry; Scan Now; Where Does the Rest Go? | Cleans, ticks |
| Notification (S6) | Review → the entry; Remind Me Tomorrow | Cleans, offers "Clean" |
| Review entry | Tick → the existing sheet (Clean N items…, the group's Clean N from X…); Scan Now; Open Privacy Settings; Settings… | Ticks for the person; puts a non-finding in the sheet |
| Whole disk Change (S7) | The row's existing action sheet (`ddc disk free`), Show in Finder | Runs `ddc disk` by itself; touches snapshots |
| `ddc space`, `ddc changes` | The person's own `ddc scan` or `ddc clean` | Write anything but `space.json` and their output |

### Safety guarantees kept

| Change | Guarantee | How it holds |
|---|---|---|
| The growth entry in Review | The sheet is the only thing that starts a clean | The entry calls the same `openSheet` paths; no new caller of the clean |
| Rows keep their ticks | Nothing is ticked for the person beyond the default selection | `defaultSelected` is unchanged; the group checkbox is `toggleAll` where `layout.selectAll` exists |
| Rows that are no finding | Only findings can be cleaned | No checkbox; not in `ReviewItems` |
| The signal and its scan | The background path never cleans (safety ADR §8); no unattended clean (owner, 2026-10-28) | `ddc space` reads; a later signal-triggered scan is `scan --background` |
| The badge | Nothing interrupts without the person's settings | `ShowMenuBarBadge` governs it |

## Rollout

Each story ships alone. The app stories belong in DDC-002 "Menu bar app". The CLI stories could sit there too, as "CLI: shared settings, types and the notify decision" did, or in DDC-001 "Disk space cleaner CLI for developers". The planner decides.

| # | Story (working title) | Core/CLI | App | Depends on | Release |
|---|---|---|---|---|---|
| S5 | Notice when used space grew: `ddc space`, the badge and the popover band | `ddc space [--json]`, `docs/space.schema.json`, `space.json` (the CLI its one writer); ADR §6 item 13; README "Where things live" | Calls on the tick (after its scan), wake (new), popover open, and from refresh at most once per 10 minutes; the band (four states, shown only while its baseline is the loaded report's); the signal's badge flag, cleared when the popover opens or the baseline changes; the `f_bavail` check on a real Mac | — | first, its own PR |
| S1 | Keep the previous report, and say who wrote each | `write_last_scan` moves the old file to `previous-scan.json` (not when the outgoing report is `cancelled`); additive report fields `written_by` (`scan`, `background`, `clean-recheck`, `clean-items`, `disk-free`), `written_at`, `previous_written_at`, `rules_in_force` (the volume's `read_at`, set by the `clean --items` edit, came with S5); a terminal `ddc clean` edits the saved report after the clean, as `--items` does, and its pre-clean rescan writes without rotating, so yesterday's report stays the baseline | Decoders gain the fields | — | first |
| S2 | `ddc changes`: what grew between the last two reports | The comparison with not-read sets, rules in force, moved, cleaned and trashed from the journal; `docs/changes.schema.json`; the `ddc scan` block | — | S1 | first |
| S4 | Popover and Review: "Grew since …" | — | `changes --json` after a report load; the band names the group; the Review entry with the Change column, the no-finding and not-read sections and their actions. Lands on today's Groups/Items switch | S2, S5 | first |
| S3 | Folder sizes in the scan report | `folders` (the roots' children and their large children, with a depth and a size floor), compared in `changes`; a load test | The no-finding rows with Show in Finder | S2 | later |
| S6 | A scan when used space grew, and its notification | A `growth` reason in `notify.rs`; `notify ack --growth`; `growth_for` in `notify-state.json` | The due function's new trigger (settled, rate-limited, not paused); `NotificationText` wording; `menuBarOnly` honoured; ADR §5 amendment | S2, S5 | later |
| S7 | Whole disk: what changed since the last whole-disk scan | `previous-disk.json` rotated in `write_last_disk`; area changes by `id` in `ddc disk --json` or `ddc changes --disk` | The Change column, the snapshots sentence, the home row's link | S1's pattern | later |
| S8 | Since yesterday, since last week | Digests of past reports with retention (newest two never deleted, retention run only after a successful write), `ddc changes --since`; a digest gone mid-read is reported, not guessed | A picker in the entry; a Change column in All Items | S2, S4 | later |

**Windows and Linux.** S1, S2, S3 and the core of S5, S6 and S7 are platform-neutral and reach them with the CLI. `statfs` has its Windows counterpart behind the platform seam. Their apps (the Tauri app of ADR "Windows app: Tauri 2 shell over the ddc CLI"; Linux has none) follow later.

**Before implementation**, per the project's rules: this note through the critic again; an ADR "Menu bar app" amendment for §6 item 13 (`ddc space`), and later for §5 (S6); the planner on each story.

## Risks

- **Privacy.** `previous-scan.json` holds the same paths as `last-scan.json` (project, worktree and folder names), in the same data folder; `space.json` holds only sizes and times. Nothing leaves the Mac (README "Network": the only requests are Sparkle's). Both files follow `--data-dir`. The cask's `zap` paths should cover them **(inferred: not checked)**.
- **Disk cost.** One more full report. Its size was not measured (the real data folder was off limits); several MB is plausible for a large home **(inferred)**. Measure on a fixture home in S1.
- **Performance.** `ddc space` is a process start and a syscall, a few times an hour. `ddc changes` is a join over two reports in memory, parsed once each. The one real cost is S6's background walk: settle first, rate-limit, let pause win.
- **Trust.** A false "+30 GiB new" for a folder that was only unread last time would teach people to ignore the band. The not-read and rules-in-force rules ship with S2, not after.
- **Two numbers that disagree.** "On the startup disk" includes snapshots and system data, "in your home folder" does not. Both are always named, and the remainder line says what it holds.
- **The badge's two meanings.** A signal badge clears when the popover opens, a notification badge when Review opens. If one icon carries both, VoiceOver names both. The owner agreed to the signal's badge (open question 3, 2026-11-02).
- **Schema churn.** Two new contracts with their own `schema_version` (`space`, `changes`) and additive report fields. The app decodes strictly (ADR §1), so app and CLI change together.

## Conclusion

Notice first: `ddc space` compares the disk's free space with the last report and lights the badge, so a 200 GB day does not go unseen. Then say where: keep the previous report, compare the two in the CLI over what both read and what rules both had, and say it where the person already looks. That is a band in the popover with a number and a name, and a "Grew since …" entry in Review over the same rows and ticks. A Change column in Whole disk and a growth notification come later. The remainder is named for what it is, folders ddc did not read or the world outside the home folder, never passed off as "outside". Nothing cleans, nothing is ticked for the person: scans inform, people remove.

## Open questions (for the owner)

| # | Question | Earlier decision | Recommendation |
|---|---|---|---|
| 1 | Baseline: the previous report (whatever its age), with "since yesterday / last week" only once history is kept (S8)? | None | Yes; the band says the baseline's age |
| 2 | May the app start a background scan, outside the schedule, when in-use grows past the threshold? | ADR "Menu bar app" §5: the due function decides from the schedule, a reminder and pause | Yes in S6: settled, at most once per 6 h, never while paused, as scheduled work |
| 3 | May the signal set the menu bar badge (no notification), cleared when the popover opens? **Decided 2026-11-02: yes.** | §5: the badge follows a posted notification and clears when Review opens; `ShowMenuBarBadge` | Yes: it is how a person notices without a notification |
| 4 | Threshold: max(20 GiB, 5 % of the disk) since the last report? One field in Settings → Notifications later? | `notify_threshold` (10G) is for reclaimable bytes | Those defaults; the field later |
| 5 | May a growth notification be posted after a scan? A signal-only one? | §5: "Only scheduled scans notify" | Yes after a scan (S6); no signal-only notification |
| 6 | Show shrinking anywhere? | None | Only in the Review entry, folded |
| 7 | A Change column in All Items, or only in the growth entry? | Review concept: All Items replaces the switch | Only the entry first; All Items with S8 |
| 8 | A public `ddc changes`, readable by coding agents under the "Agent footprint" ADR §7 bound? | README "For coding agents" | Yes |
| 9 | Folder sizes in the scan report: depth and size floor (say depth 3, ≥ 1 GiB)? | None | Yes, as S3, with a load test |
| 10 | Should a terminal `ddc clean` edit the saved report after the clean, as `--items` does? | Menu bar ADR §6 item 1 (the edit after `--items`) | Yes, in S1 |

## References

Code on `main` at `018060d`:
- `crates/ddc-core/src/store.rs`; `model.rs` (ScanReport, Totals, Volume, Finding, Origin, SkippedPath); `report/mod.rs` (`ScanOptions.skip`, `user_skips`, `count_findings`); `report/after_clean.rs`; `report/volumes.rs`; `rules/visitor.rs` (`before_read`, `not_scanned`); `scan/walker.rs` (`skip`, `max_recorded`); `rules/catalog/macos.rs` (cloud folders); `free/mod.rs` (`edit_last_scan`); `disk/model.rs`, `disk/apfs.rs`, `disk/areas.rs`; `notify.rs`
- `crates/ddc-cli/src/cmd/scan.rs`, `cmd/background.rs` (`PROMPTING_AREAS`), `cmd/clean.rs`, `cmd/clean_items.rs`, `notify.rs`, `cli.rs` (`notify ack`)
- `app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift`, `ScanReport.swift`, `ReviewItems.swift` (`cloudSkipped`, `backgroundSkipped`), `NotReadable.swift`, `NotificationText.swift`, `ScheduledWork.swift`, `AppPreferences.swift`, `IconState.swift`, `ReportSummary.swift`, `GroupLayout.swift`, `Formatting.swift`
- `app/DeepDriveCleaner/PopoverView.swift`, `DiskView.swift`, `StatusItemController.swift`, `AppDelegate.swift`
- README: "The menu bar app", "`ddc scan`", "`ddc disk`", "`ddc schedule`", "Where things live", "Settings", "For coding agents", "Network"

Vault:
- [[epics/DDC-002/epic|Menu bar app]]
- Research "Review window on macOS: a UX audit and a lighter concept" (2026-11-02)
- ADR "Menu bar app: Swift shell over the ddc CLI" (§1, §5, §6 items 1, 9, 10); ADR "Cleanup safety model: risk classes, plan and confirmation" (§8); ADR "Agent footprint: worktrees, harness data and bounded consent for agents" (§7, 2026-11-02)
- DDC-002 stories "Background scan and notifications in the app", "Menu bar popover", "Whole disk in the app", "CLI: shared settings, types and the notify decision", "Faster clean: re-check only what the selection needs"; DDC-001 "Scheduled background scan"

Web:
- [Apple: Checking volume storage capacity](https://developer.apple.com/documentation/foundation/checking-volume-storage-capacity)
- [Apple: File System Events Programming Guide](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/Introduction/Introduction.html)
- [Eclectic Light: Where does macOS get its volume free space figures from? (2023-04-27)](https://eclecticlight.co/2023/04/27/where-does-macos-get-its-volume-free-space-figures-from/)
- [Eclectic Light: The Finder confuses with wildly inaccurate figures for available space (2023-04-17)](https://eclecticlight.co/2023/04/17/the-finder-confuses-with-wildly-inaccurate-figures-for-available-space/)
- [Eclectic Light: Aren't snapshots purgeable? (2026)](https://eclecticlight.co/2026/08/24/arent-snapshots-purgeable/)
- [TreeSize: Disk Usage Comparison](https://manuals.jam-software.com/treesize/EN/disk_usage_comparison.html)
- [FolderSizes vs WizTree](https://www.foldersizes.com/features/wiztree)
- [iStat Menus on the App Store](https://apple.co/2VlAxEw)

---

*Last updated: 2026-11-03*
