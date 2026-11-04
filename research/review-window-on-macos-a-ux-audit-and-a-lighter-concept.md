---
type: research
slug: "review-window-on-macos-a-ux-audit-and-a-lighter-concept"
title: "Review window on macOS: a UX audit and a lighter concept"
status: draft
date: 2026-11-02
authors: ["Evgenii Konev"]
tags: [app, macos, ux, review-window]
review_status: pending
reviewed_at: null
---

# Review window on macOS: a UX audit and a lighter concept

## Question

The owner finds the menu bar app's Review window heavy, and sees the same things in several places (2026-11-02). What exactly repeats, what makes the window heavy, and what would a lighter window look like that keeps every safety guarantee and every decision made for a reason? macOS only: Windows and Linux follow once the Mac is right.

## TL;DR

The weight is in reading and choosing, not in clicks: accepting the defaults takes four clicks, which is fine. But one screen shows a group's selected amount three times, stacks up to four banners, repeats each class's meaning on every group of the class, gives one action two names ("Scan Again", "Scan Now"), shows "not scanned" in four shapes, switches between two views of the same rows, and hosts a second job, Whole disk, with its own header, selection and sheets. The sheet puts 120 to 200 words of prose before its item list, and the result draws counts of zero.

The concept removes repeats, not features. One number per fact. One place for what was not scanned, with the Full Disk Access state. The scan's age in view. A group's own action stays in the group, where a person looking at that group needs it, and moves into the group header instead of a row of its own. The way back to the popover stays visible. The sheet keeps every item, action, toggle and warning, user data first, and folds the fixed re-check text. The safety model does not change.

Some changes amend earlier decisions; each is listed under "Open questions for the owner" with the story it comes from and a recommendation. The work splits into six stories, each shippable alone.

## Method

- A designer pass (a subagent briefed as a senior macOS product designer) rebuilt the window from the code on `main` at 225d5f4: every region, control, label, banner, sheet and dialog, with file and line. Nothing was built or run.
- An independent pass checked the main claims against the same code: the group amount drawn in the sidebar and again in the group header (`model.amount(of: entry)` in `ReviewView.swift` and `ReviewGroupsView.swift`); the Select all row's own total; three buttons that open the sheet with different scopes (the footer's `openSheet()`, `SelectAllRow`'s `Wording.cleanFrom`, `PickedRow`'s `Wording.moveFrom`); the segmented Groups/Items picker; Whole disk shown inside the detail (`model.wholeDisk`); the result's tiles drawn at zero (`CleanResultView.swift`). All confirmed.
- A critic pass (`projectstore:critic`) checked the first draft against the code and DDC-002's stories. It found two wrong facts (what the close box does; the order of the sheet's blocks), a proposal that repeated a user's recorded complaint (moving the group action into a menu), and missing attributions. This version corrects them.

## Findings

### What repeats

| Fact or action | Where it shows | Verdict |
|---|---|---|
| A group's selected amount | Sidebar entry; group header (the same string, beside it); the Select all row's summary, which follows the filter and can then disagree | Hurts: three numbers for one fact, and the third reads like a bug under a filter |
| What a class means | Popover class rows; every group header of the class; the User data rule in the sidebar blurb, the group's pick blurb and the sheet | Hurts: once per class is enough. The Trash and "Protected by you" blurbs are facts of their group, not of a class, and stay |
| Not scanned | A banner (your skips); a sidebar line (the background scan's skips); a "Cloud folders" entry; a no-Full-Disk-Access note shown only in the popover | Hurts: four shapes for "these places were not read" |
| Rescan | "Scan Again" in the header and the result; "Scan Now" in the scope banner, the empty state and the popover | Mild: two names for one action; the scope banner's "Scan Now" is pinned by its story and its wording, so the concept keeps both |
| Groups vs Items | A segmented switch between two views of the same rows | Hurts: a mode for what is one more sidebar entry; its value (cross-group sort, ignored rows shown) stays |
| Whole disk | Its own window, and again as a sidebar row inside Review whose Coding Agents "Review…" leads back into Review; the Review header disappears while it shows | Hurts: a second job, with a second selection and its own sheets, inside the cleaning window |
| Start a clean | The footer's "Clean N items…"; in a group, "Clean N from X…" on the Select all row or "Move N from X to the Trash…" on the picked row; the popover's "Clean Selected…" | Mixed. The group action is why two stories exist ("Review window: clean only the selection in this group"; "User data: move a folder's picked files to the Trash from that folder", which came from a user's screen recording where nothing near a picked file offered to move it). Keep it in the group. What hurts is its own extra row and its repeated total |
| Close | The footer button (the way back to the popover); the close box (closes only, by the owner's decision of 2026-10-19); ⌘W; Esc | The button is the only visible way back to the popover. Keep it |
| Lock reason, scope notice, Remove Protection | Inline row line; the scope banner (Clean is off for a report outside the roots); only in "Protected by you" | Helps: each says why something cannot or should be done |

### What makes it heavy

- About 40 to 45 click targets in the default view, five of them bordered buttons.
- Five visual levels before a row: tinted class panel, group entry, group header, Select all row, subgroup, then the row, with three checkbox levels and a fourth seed in the popover (Include Safe, Include Rebuildable).
- Up to four full-width banners above the list, and the frame jumps when Whole disk hides the header and swaps the footer.
- The sheet: a re-check paragraph of about 55 words, always, and class explanations of 9 to 70 words each, before the item list.
- The result always draws four tiles ("deleted", "tool commands", "skipped", "failed"), zeros included.

### What is missing

- The window never says how old the scan is.
- It never mentions Full Disk Access, although the popover does.

## Verdict

Both passes agree on these changes, corrected by the critic pass.

| Change | Why |
|---|---|
| A toolbar band: the filter field (as today: Esc clears it first, Return while typing never opens the sheet), Scan Again, and a subtitle with the scan's age and scope. While a rescan runs, Scan Again shows a spinner and the subtitle says why ("Scanning again after the clean…"). With scans paused, the band says "Scans paused", as today | The missing scan age, and the rescanning banner becomes a state that keeps its reason. "Scan Again" in the band and the result; the scope banner and the empty state keep "Scan Now", the popover's name, as their stories say |
| At most two banners, the scope notice and a protect error, stacked when both apply | The other two banners become the toolbar state and the Not scanned entry |
| The sidebar keeps the one "selected of total"; the group header and the Select all row lose theirs | One number per fact |
| The Select all row becomes a tri-state checkbox at the start of the group header, drawn only where Select all exists today, acting on the rows the filter shows; the group's own action ("Clean 2 from Models…", "Move 2 from Downloads to the Trash…") becomes a button at the right of the same header, shown when it shows today; the picked row's summary ("2 picked · 5.3 GiB") joins the header line | The group keeps its own way to clean, where the person is looking, without two extra bands. A coding agent's group, which has both Select all and picked sessions today, keeps both actions in its header |
| "All Items" at the top of the sidebar replaces the Groups/Items switch; the Items view's heading goes, since the sort header already names the sort | One mode less; cross-group sort stays |
| One "Not scanned" entry, at the foot of the sidebar outside the class sections, gathers your skips, the background scan's skips, cloud folders and the no-Full-Disk-Access line, each with its own action (Settings…, Scan Again, Open Privacy Settings) and the cloud warning text | One place for one fact, with room to explain |
| Whole disk leaves the Review detail; a "Whole Disk…" link at the sidebar's foot opens its window | Different job, own selection and sheets; no loop back |
| The class meaning: a tooltip on the sidebar class name and one facts line per group header ("Safe · deleted · refills on the next install"); the way back stays visible on `review` groups | Said once; what a person needs before ticking a `review` item stays in view |
| Rows: the second line only when it holds something (a lock, a not-selected reason, an incomplete read, protection); "frees ~X" becomes "≈" on the size with a tooltip | Reasons stay inline, never in a tooltip, because they are why a box cannot be ticked |
| The footer: one line ("39 items in 5 groups · 26.6 GiB · frees about 25.1 GiB"), Deselect All and Close (both plain, without a border; their shortcuts and behaviour unchanged), and Clean | "in 5 groups" (sidebar entries holding a selected row) tells what Clean takes; the Trash note moves to the sheet |
| The sheet keeps its blocks in today's order, your own files first (`Confirmation.order`), each with a one-clause explanation, and keeps the warnings, the Delete all permanently toggle (its label unchanged), the per-item toggles, the item list and the Trash sentence; the re-check paragraph and the Time Machine note fold into "What is checked before each item" | Consent stays informed; fixed text read once is folded |
| The result: one line of the non-zero counts and one footnote for APFS and snapshots | Zero tiles carry nothing |

What stays as it is: the empty state, the sort header and the Changed column, subgroup folds with their hint and folder icon, the lock icon in place of a checkbox, the Protect and Remove Protection dialogs, Deselect All, Close and the Esc behaviour, the cleaning view and the stopped view. The designer's proposals to drop the Close button and to move the group action into a split button in the footer were withdrawn after the critic pass, for the reasons in the table above.

## Concept

### Default state

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│ Review and Clean                       [⌕ Filter by path or rule   ]   [↻ Scan Again] │
│ Scanned 2 h ago · your home folder                                                   │
├───────────────────────────────┬──────────────────────────────────────────────────────┤
│  All Items           41.2 GiB │ ☑ Package caches                                      │
│                               │   Safe · deleted · refills on the next install        │
│  ● Safe                       │ ───────────────────────────────────────────────────── │
│    Package caches  9.2 of 13.1 GiB │  Name                      Size ▾     Changed     │
│    Browser caches     3.1 GiB │ ▾ ☑ ~/Library/Caches  5 folders  8.0 GiB   Today       │
│  ● Rebuildable                │     ☑ pip                         3.2 GiB   Today       │
│    Project builds    12.0 GiB │     ☐ Homebrew                    1.9 GiB   Sep 30      │
│  ● Review                     │       Project untouched 3 days · not selected         │
│    Models   42.4 of 46.4 GiB │ ▸ ◩ ~/.npm          2 folders    5.1 GiB   Sep 25      │
│  ● Your files                 │                                                       │
│    Downloads   5.3 of 14.0 GiB│                                                       │
│    Protected by you           │                                                       │
│                               │                                                       │
│  Not scanned (3)              │                                                       │
│  Whole Disk…  ↗               │                                                       │
├───────────────────────────────┴──────────────────────────────────────────────────────┤
│ 39 items in 5 groups · 26.6 GiB · frees about 25.1 GiB   Deselect All  Close  [Clean 39 items…] │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### A `review` group with part of the selection

```
│ ◩ Models                                            [Clean 2 from Models…] │
│   Review · goes to the Trash · not selected for you                         │
│   Way back: ollama pull <model> downloads it again                          │
│  ☑ ~/.ollama/models                     30.0 GiB    Aug 2                   │
│  ☑ ~/.cache/huggingface               ≈ 12.4 GiB    Sep 30                  │
│     Part of it could not be read; it may be skipped                         │
│  🔒 ~/.cache/lm-studio                    4.0 GiB    Today                   │
│     Protected by .ddc-keep                                                  │
```

The group's button opens the same sheet with this group's shown, selected rows only, as "Clean N from X…" does today.

### Your own files

```
│ Downloads        2 picked · 5.3 GiB    [Move 2 from Downloads to the Trash…] │
│   Your files · pick one by one · they go to the Trash, Put Back restores     │
│  ☑ big-video.mov                     4.2 GiB    Sep 2                        │
│  🔒 Photos Library.photoslibrary      8.1 GiB    Today                        │
│     Photos manages it · clean it in Photos                                   │
│  ☑ installer.dmg                     1.1 GiB    Aug 30                       │
│  ☐ Partial.crdownload                0.6 GiB    Today                        │
```

No header checkbox and nothing ticked for you, as today.

### The confirmation

```
┌───────────────────────────────────────────────────────────────────┐
│ Clean 39 items?                                                    │
│ 26.6 GiB selected · frees about 25.1 GiB                           │
│ ■ Your own files · 2 items   to the Trash, Put Back      5.3 GiB   │
│ ■ Safe · 30 items            deleted · refills          12.1 GiB   │
│ ■ Rebuildable · 5 items      deleted · reinstall         6.0 GiB   │
│ ■ Review · 2 items           to the Trash or its tool    3.2 GiB   │
│ ⚠ 1 item will be skipped; the rest is cleaned.   [Check Again]     │
│ ☐ Delete all permanently                                           │
│ ┌────────────────────────────────────────────────────────────────┐ │
│ │ ~/Downloads/big-video.mov                 Trash        4.2 GiB │ │
│ │ ~/Library/Caches/pip                      Delete       3.2 GiB │ │
│ │ ~/.ollama/models        ☐ Permanently    Trash       30.0 GiB  │ │
│ │   Ollama is running · will be skipped                          │ │
│ └────────────────────────────────────────────────────────────────┘ │
│ The Trash keeps the space until you empty it.                      │
│ ▸ What is checked before each item                          (?)    │
│                               [Back to Review]   [Clean 39 items]  │
└───────────────────────────────────────────────────────────────────┘
```

### Not scanned

```
│ Not scanned                                                             │
│   These places were not read, so nothing in them is listed.             │
│  Skipped in Settings     ~/Projects/client-archive         [Settings…]  │
│  Background scan         Downloads, Pictures, Movies       [Scan Again] │
│  Cloud folders           ~/Library/CloudStorage/…                       │
│     A file there may live only in the cloud; the Trash would remove     │
│     it from every device. Use Remove Download in Finder.                │
│  Without Full Disk Access, part of ~/Library    [Open Privacy Settings] │
```

### The order of what a person reads

1. Where the space is and what is ticked: sidebar amounts, rows, the footer.
2. Why a row cannot be ticked, or is not ticked for you: its second line.
3. What a clean does to each item: the sheet.
4. What happened: the result.
5. Everything else, one click away: class definitions, the re-check list, APFS and snapshots, the places not scanned, the whole disk.

## Safety guarantees kept

| Change | Guarantee | How it holds |
|---|---|---|
| The group action moves into the group header | The sheet is the only thing that starts a clean; Return never cleans | The button still calls `openSheet(scope:)`; the footer still calls `openSheet()`; `cleanConfirmedItems()` keeps one caller (`ContractTests.cleanHasOneOwner`) |
| Select all moves into the header | User data is never ticked for you; sessions and builds are picked one by one | Drawn only where `layout.selectAll` is not nil; `toggleAll` keeps skipping locked rows |
| The picked summary joins the header | Your own files only go to the Trash | `Confirmation` decides the action; the sheet title is unchanged |
| The sheet's blocks keep their order | The person sees their own files first (ADR "Cleanup safety model: risk classes, plan and confirmation", amendment of 2026-10-20) | `Confirmation.order` is unchanged |
| Lock and not-selected reasons stay inline | Locked items are visible and explained | The row's lock branch is unchanged |
| The way back stays visible on `review` groups | `review` is never ticked for you, and the cost is known before ticking | `defaultSelected` is unchanged |
| The sheet folds its fixed text | Consent is informed | Items, actions, both permanent toggles, warnings and the Trash sentence stay visible |
| Whole disk leaves Review | Whole disk frees only through its own confirmed sheet | `DiskView` and its sheets are untouched, only no longer embedded |
| One Not scanned entry | Cloud folders are never offered | The entry has no checkbox, and the cloud warning stays |
| The scope notice stays a banner | Clean is off for a report outside the roots | `canClean` is unchanged |
| No class-level checkbox | All `review` items cannot be ticked in one click | None is added |

## Rollout

Six stories in DDC-002, each shippable alone:

1. The toolbar band (filter, Scan Again, scan age, rescanning and paused states), at most two banners, and the "Not scanned" entry with the Full Disk Access line. Amends "User data: pick your own files for the Trash" (DDC-002: "Cloud folders" as the last entry; the background-scan line in the User data section).
2. The group header: the tri-state checkbox, the group's action button and the picked summary in it; the group amount gone from the header; the class meaning as a facts line and a tooltip. Amends "Review window: sorted sidebar, Select all and folder groups" (DDC-002: the Select all row; the header's amount). Rewrites the contract test `userDataPickedRow` to the same intent (no checkbox, no `toggleAll`, only `openSheet(scope:)`, now in the header) and the blurb pins in `ProtectionTests` that cover the User data section blurb.
3. "All Items" replaces the switch, and the Items heading goes. Amends "Review and clean window" and "Review window: sort a group's rows by name, size or last change" (DDC-002).
4. Whole disk leaves the Review detail; a link opens its window. Amends "Whole disk in the app" (DDC-002) and the README.
5. The footer line, the sheet's folds and the result's one line. The result's snapshot line stays conditional, shown only when local snapshots exist (`localSnapshots > 0`), as the safety ADR asks.
6. Lighter class sections, only if the owner wants them (question 5).

Hand-picked ticks across a trip to the popover are not part of these: question 7 needs an answer first, and its answer is a selection rule, not a layout.

## Migration risk

- Mostly view work in `app/DeepDriveCleaner/` (`ReviewView.swift`, `ReviewGroupsView.swift`, `ConfirmSheet.swift`, `CleanResultView.swift`, `ReviewWindowController.swift`, `AppDelegate.swift`).
- DDCKit: `ReviewModel.mode` gives way to an entry kind; `ReviewModel.wholeDisk` goes; the three not-scanned sources (`notScanned`, `backgroundSkipped`, `cloudSkipped`) merge and gain the Full Disk Access fact; `scopeSummary` and `itemsHeading` lose their callers; `pickedSummary` moves to the header. `GroupLayout.selectAll`, `pickScope`, `Wording.cleanFrom` and `moveFrom` stay. About twelve DDCKit test files touch these.
- Pins in tests: `ContractTests.userDataPickedRow` is rewritten (story 2); `cleanHasOneOwner`, `resultScanAgain`, the Remove Protection pins, the Stop button pin and `DockPresenceTests.closeToPopover` keep passing because Close and its closure stay; `ProtectionTests` and `TrashTests` blurb pins stay for the Trash and "Protected by you" blurbs and change only for the User data section blurb; `ReviewSortTests` loses `itemsHeading`; new buttons follow the `titleCase` contract.
- There is no UI snapshot or UI test target, so the look is checked only by the owner: each story adds an owner check to the runbook "Owner checks: manual acceptance across stories".
- No core, schema or CLI change: Windows and Linux are not touched.

## Conclusion

Lighten the window by removing repeats, not features: one number per fact, one place for what was not scanned, the scan's age in view, reasons inline, the group's own action kept in the group but without its own row, and Whole disk back in its own window. Start with story 1: it fills two gaps and amends one story's layout without changing any rule.

## Open questions (for the owner)

The recommendation is the joint one. Each row names the story whose decision it changes.

| # | Question | Earlier decision | Recommendation |
|---|---|---|---|
| 1 | Take Whole disk out of the Review sidebar, leaving a "Whole Disk…" link to its window? | "Whole disk in the app" (DDC-002): it opens from the popover and the Review window; the README says so | Yes |
| 2 | Move the group's action ("Clean N from X…", "Move N from X to the Trash…", a coding agent's sessions) from its own row into the group header? | "Review window: sorted sidebar, Select all and folder groups" (DDC-002, board 9, 2026-10-20: the Select all row and the header's amount); "Review window: clean only the selection in this group" (DDC-002, placement B, 2026-10-21); "User data: move a folder's picked files to the Trash from that folder" (DDC-002, 2026-10-28); "Review window: pick a coding agent's sessions one by one" (DDC-002) | Yes: same button, same scope, one band less |
| 3 | Replace the Groups/Items switch with an "All Items" entry, and drop the Items heading? | "Review and clean window" (DDC-002, the toggle); "Review window: sort a group's rows by name, size or last change" (DDC-002, the heading names the sort) | Yes |
| 4 | Merge the not-scanned banner, the background-scan line and the "Cloud folders" entry into one "Not scanned" entry, with the Full Disk Access line? | "User data: pick your own files for the Trash" (DDC-002) | Yes |
| 5 | Lighter class sections: a colour dot and the name, a tint only on the selected entry, instead of four tinted panels? | Board 9, approved 2026-10-20, in "Review window: sorted sidebar, Select all and folder groups" (DDC-002) | Later, and only after stories 1 to 4: it is taste, not structure |
| 6 | Show the scan's age and the Full Disk Access state in the window? | None: a gap | Yes |
| 7 | Keep hand-picked ticks when the window is reopened through "Review…" or a biggest find? "Clean Selected…" keeps opening with exactly the popover's selection ("Review and clean window", `CleanRunTests.cleanSelected`), and the Include switches would then apply only to rows the person has not touched | Left open in "Review window: Deselect All, then pick across groups and clean only that" (DDC-002) | Yes, as its own story with that rule |
| 8 | No class-level checkbox in the sidebar? | None | Confirm: none |

Owner answers, 2026-11-02:

- **1, Whole disk:** the owner finds Whole disk useful and wants it kept, and adds a requirement: in Whole disk, too, a person must be able to clean and remove what it finds. Where it lives (the Review sidebar or its own window) is left to the story that takes it, with that requirement.
- **2, the group action in the group header:** yes.
- **3, "All Items" instead of the Groups/Items switch:** yes.
Owner answers, 2026-11-04, each the recommendation:

- **4, one "Not scanned" entry:** yes. The not-scanned banner, the background-scan line and the "Cloud folders" entry merge into one entry with the Full Disk Access line (story 1).
- **5, lighter class sections:** later, after stories 1 to 4. Story 6 waits for the owner to see the window without the repeats.
- **6, the scan's age and the Full Disk Access state in the window:** yes (story 1).
- **7, keep hand-picked ticks on reopen:** yes, as its own story with the rule in the question: "Clean Selected…" keeps opening with exactly the popover's selection, and the Include switches apply only to rows the person has not touched.
- **8, no class-level checkbox in the sidebar:** confirmed, none.

Not decided by the owner but open for the stories:

- The filter field moves into the toolbar band as the same control it is today, so the HIG pass's keyboard rules (Esc clears it first; Return while typing never opens the sheet) keep holding. Moving to `.searchable` would need to know the field's focus, which SwiftUI reports only from macOS 15; story 1 checks the app's deployment target before choosing.
- What the footer's "in N groups" counts when a coding agent's data sits in two classes: sidebar entries holding a selected row.

## References

- [[epics/DDC-002/epic|Menu bar app]]
- Stories in DDC-002: "Review and clean window", "Review window: sorted sidebar, Select all and folder groups", "Review window: clean only the selection in this group", "Review window: Deselect All, then pick across groups and clean only that", "Review window: pick a coding agent's sessions one by one", "Review window: sort a group's rows by name, size or last change", "User data: pick your own files for the Trash", "User data: move a folder's picked files to the Trash from that folder", "Protected by you: say what it is, and Remove Protection", "Confirmation sheet: Delete permanently for every item at once", "HIG pass: keyboard, colour, empty states and menus", "Whole disk in the app"
- ADR "Cleanup safety model: risk classes, plan and confirmation", amendment of 2026-10-20 (the "Your own files" block first)
- Code on `main` at 225d5f4: `app/DeepDriveCleaner/ReviewView.swift`, `ReviewGroupsView.swift`, `ReviewWindowController.swift`, `ConfirmSheet.swift`, `CleanResultView.swift`, `PopoverView.swift`; `app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift`, `ReviewItems.swift`, `Confirmation.swift`, `PopoverModel.swift`; `app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift`
- README: "The menu bar app", "Risk classes"

---

*Last updated: 2026-11-02*
