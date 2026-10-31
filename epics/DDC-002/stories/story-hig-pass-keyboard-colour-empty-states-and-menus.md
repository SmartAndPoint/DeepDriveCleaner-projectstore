---
type: story
id: "story-hig-pass-keyboard-colour-empty-states-and-menus"
epic: "DDC-002"
title: "HIG pass: keyboard, colour, empty states and menus"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-21
updated: 2026-10-31
external_refs: {}
tags: [app, hig, accessibility]
code_refs: ["README.md", "app/DeepDriveCleaner.xcodeproj/project.pbxproj", "app/DeepDriveCleaner/Assets.xcassets/AccentColor.colorset/Contents.json", "app/DeepDriveCleaner/AppDelegate.swift", "app/DeepDriveCleaner/CleanResultView.swift", "app/DeepDriveCleaner/ConfirmSheet.swift", "app/DeepDriveCleaner/MainMenu.swift", "app/DeepDriveCleaner/PopoverView.swift", "app/DeepDriveCleaner/ReviewGroupsView.swift", "app/DeepDriveCleaner/ReviewView.swift", "app/DeepDriveCleaner/ReviewWindowController.swift", "app/DeepDriveCleaner/StatusIcon.swift", "app/DeepDriveCleaner/StatusItemController.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanWording.swift", "app/Packages/DDCKit/Sources/DDCKit/Formatting.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverModelTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CleanRunTests.swift"]
specs: []
started_at: "2026-10-21T11:04:18.029Z"
closed_at: "2026-10-31T10:30:46.565Z"
plan_updated_at: "2026-10-21T11:04:18.029Z"
---

# HIG pass: keyboard, colour, empty states and menus

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner asked how closely the app follows SwiftUI idiom and Apple's Human Interface Guidelines. A read-only audit of the app's sources on 2026-10-21 found it mostly standard:
- a transient popover sized by its content;
- a template status icon with an accessibility label;
- a confirmation sheet whose Clean button is not the default, so Return never cleans;
- ellipses on buttons that open a further step;
- a grouped `Form` for Settings;
- cooperative activation, the standard About panel and a Window menu.

The deviations users meet are cheap to fix. The owner approved fixing them first, then the clean's progress bar, then Settings (2026-10-21).

The same day the owner reported a bug from a second Mac: the popover opened looking inactive. Its glass was dark, its switches grey and its Clean selected button grey. It turned light and active only after a click inside it. That bug is fixed here too.

Bigger structural changes go to the story that owns the window:
- **Review window.** A `NavigationSplitView` sidebar with `List(selection:)`, a toolbar with `.searchable`, a `Table` for the items mode, and text styles in place of about 85 fixed font sizes go to "Review window: sorted sidebar, Select all and folder groups" (DDC-002) or a successor.
- **Settings window.** Toolbar tabs through `NSTabViewController` go to "Settings: folders, types, notifications" (DDC-002).
- **Localization** (plurals, locale-pinned number formats) waits until the app is localized.

## Decomposition

Rough until the plan gate.

- [x] Popover opens active:
  - **Where it goes wrong.** `showPopover` asks for activation with `NSApp.activate()`, shows the popover and clears the first responder (`app/DeepDriveCleaner/StatusItemController.swift:91-98`). Activation is cooperative since macOS 14 and arrives after `show`. Nothing then makes the popover's window key, so it draws inactive until clicked.
  - **Fix.** After showing, make the popover's window key, *then* clear the first responder. The order matters: a window's first key pass installs the gear as first responder, and with `.focusEffectDisabled()` gone that would show a ring. If the app becomes active only later, do it again on `NSApplication.didBecomeActiveNotification`. This happens only while the popover is shown and no other window is key, so a Review window opened by a notification keeps key.
  - **Log.** `AppLog` records whether the app was active and the window key after the click. The second Mac's `~/Library/Logs/DeepDriveCleaner/app.log` then tells which half failed if the fix is not enough.
- [x] Keyboard:
  - **Esc in the Review window.** Esc keeps closing the window and returning to the popover, as the owner decided on 2026-10-19 ("Review and clean window", DDC-002). The one exception: while the filter field has text, Esc first clears the filter.
  - **Return in the Review window.** While the filter field has focus and holds text, Return never opens the Clean sheet. With the filter empty, Return still opens it, as before. The filter may hold the focus as soon as the window opens, and Return should still work then. Today the footer's Close holds `.cancelAction` and "Clean N items…" holds `.defaultAction` (`app/DeepDriveCleaner/ReviewView.swift:130,134`). AppKit tries key equivalents before the field editor's `keyDown:`.
  - **Cleaning sheet.** Its button reads Stop, the word its `.stopping` phase already uses, and Esc triggers it (`ConfirmSheet.swift:207`). This supersedes "Cancel" in the ACs of "Review and clean window". The sheet already refuses dismissal during a clean (`app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift:376-380`); that stays.
  - **Confirmation sheet.** Clean gets `role: .destructive` and stays non-default. On macOS 26 it may render red, which is right for a destructive action.
  - **Popover focus.** `.focusEffectDisabled()` goes from the Settings button (`PopoverView.swift:46`), which gains `.help("Settings")`. With keyboard navigation on, the first focus lands on Review… through `defaultFocus`, not on the gear.
- [x] Colour:
  - **Accent.** An `AccentColor` asset holds the brand blue `#2459B8` with a dark-mode value and is set as the global accent (`ASSETCATALOG_COMPILER_GLOBAL_ACCENT_COLOR_NAME`). `PopoverView.accent` goes:
    - the `.tint` calls, so the user's system accent applies and Multicolor shows the brand blue;
    - the review checkbox (`ReviewView.swift:369`), which uses `Color.accentColor`;
    - the confirmation sheet's tints (`ConfirmSheet.swift:101,188`);
    - the status icon's own copy, `StatusIcon.accent` (`StatusIcon.swift:15,66`), whose badge fill uses `NSColor.controlAccentColor`.

    Class colours (the bar, chips and sidebar tints) stay as they are.
  - **Warnings.** Warning text and glyphs use `ReportClassStyle.ink(.review)`, which has a dark value, instead of the single `ReportClassStyle.review` `#B45F06`. The single colour reads at about 2.3:1 on a dark background.
- [x] Empty states in the Review window use `ContentUnavailableView`:
  - no scan yet, with Scan Now (`ReviewView.swift:33-41`);
  - nothing found;
  - nothing in this group;
  - no match for the filter (`ContentUnavailableView.search(text:)`).

  The popover keeps its own state block.
- [x] Menus:
  - **App menu:** gains Services, Hide Others (⌥⌘H) and Show All.
  - **Window menu:** gains Zoom and Bring All to Front.
  - **Edit menu:** gains Find (⌘F), which focuses the Review window's filter.
  - **Help menu:** a new menu, set as `NSApp.helpMenu` so Help search works. Its "DeepDriveCleaner Help" opens the README on GitHub.
- [x] Wording: buttons, menu items and window titles use title-style capitalization, as macOS does:
  - "Scan Now", "Pause Scans" / "Resume Scans", "Review and Clean";
  - "Reveal in Finder" becomes "Show in Finder";
  - the strings are inline in `PopoverView.swift`, `StatusItemController.swift`, `ReviewView.swift` and `ReviewWindowController.swift`, plus `CleanWording.backgroundSkipped` ("Scan now to list them") in DDCKit;
  - `README.md` follows by hand; nothing checks it;
  - `docs/releases/` stays as published.
- [x] Formatting:
  - the next-scan time follows the user's 12/24-hour setting (`Date.FormatStyle`), not `"%02d:%02d"` (`app/Packages/DDCKit/Sources/DDCKit/Formatting.swift:148`);
  - elapsed timers use `monospacedDigit()`, so they stop shifting every second (`PopoverView.swift:91,203`).
- [x] Accessibility:
  - the status item's turning arc stands still while Reduce Motion is on (`StatusItemController.swift:143-154`);
  - result tiles read as one element;
  - the item checkbox reports the toggle trait.

## Implementation Plan

Planner, 2026-10-21. The work is split into steps; each ends with a check that the main thread runs.

1. **Tests first** (`ContractTests.swift`, `PopoverModelTests.swift`):
   - a contract test for title case over the `Button("…")`, `ClosureMenuItem("…")`, `withTitle:` and `.title =` literals, with a word list for small words and one exception, `Show app.log in Finder`;
   - a contract test that `PopoverView.accent`, `StatusIcon.accent` and the `0x24` literal are gone, that `ReportClassStyle.review` is never a foreground style, and that Stop has `.cancelAction`;
   - `nextScan` cases for `en_GB` and `en_US` (U+202F normalised).

   Check: `make app-test` fails, listing the literals to fix.
2. **DDCKit.** `Wording.nextScan(…, locale:)` uses `Date.FormatStyle` with the calendar's time zone. `CleanWording.backgroundSkipped` reads "Scan Now".

   Check: `make app-test`.
3. **Colour:**
   - `Assets.xcassets/AccentColor.colorset` holds `#2459B8` for light and `#3471D6` for dark;
   - `ASSETCATALOG_COMPILER_GLOBAL_ACCENT_COLOR_NAME` is set in both target configurations;
   - the 9 uses of `PopoverView.accent` are removed: `.tint` calls dropped, the checkbox uses `Color.accentColor`;
   - `StatusIcon` badge uses `NSColor.controlAccentColor`;
   - the 10 warning sites use `ReportClassStyle.ink(.review)`.

   Check: the unsigned build, and `NSAccentColorName` in the built Info.plist.
4. **Popover.** `StatusItemController.keyPopover(cause)` runs `makeKey()`, then `makeFirstResponder(nil)`, then writes an `AppLog` line. It is called after `show`, and once on `didBecomeActiveNotification` when this click found the app inactive. In `PopoverView`, `.focusEffectDisabled()` gives way to `.help("Settings")`, and `defaultFocus` points to Review….
5. **Keyboard:**
   - the filter's clear button takes `.cancelAction` while it exists, and Close takes it only while the filter is empty;
   - Clean N items… drops `.defaultAction` while the filter has focus (`@FocusState`);
   - the cleaning sheet's button is Stop, with `.cancelAction`;
   - the confirmation's Clean gets `role: .destructive` and no tint.
6. **Menus** (`MainMenu.make(about:settings:find:)`):
   - Services, Hide Others and Show All;
   - Zoom and Bring All to Front;
   - Find, which posts `.findInReview` only while the Review window is key, and the view focuses its filter;
   - Help, set as `NSApp.helpMenu`, opening the README.
7. **Views:**
   - `ContentUnavailableView` for no scan yet, nothing found, nothing in this group, and no match, in the groups and items modes;
   - `monospacedDigit` on timers;
   - Reduce Motion, checked on every arc tick;
   - result tiles combined for VoiceOver;
   - the `.isToggle` trait on `CheckBox`.
8. **Title-case literals.** Change them in the app and the README (lines 58 and 271). The literals that name buttons in prose change too. Toggles and checkboxes keep sentence case ("Select all", "Delete permanently").

   Check: the build and a green `make app-test`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Unit test: `Wording.nextScan` gives "1:00 PM" for a 13:00 slot with a 12-hour `en_US` locale, and "13:00" with `en_GB`. — evidence: `PopoverModelTests.nextScan`
- [x] Contract test over the app's sources (`appSources()`, beside `cleanHasOneOwner` in `ContractTests.swift`):
  - `PopoverView.accent` and `StatusIcon.accent` are referenced nowhere, and no `srgbRed: 0x24` colour literal is left;
  - `ReportClassStyle.review` is never a foreground style;
  - the cleaning sheet's Stop button carries `.keyboardShortcut(.cancelAction)`. The confirmation stage's Cancel already has one (`ConfirmSheet.swift:97`), so the test targets the Stop button;
  - every `Button("…")`, `ClosureMenuItem("…")` and `window.title = "…"` literal is in title case, apart from a listed set of sentence-style exceptions such as notes and help text.

  Evidence: `ContractTests.colours` (no `PopoverView.accent` or `StatusIcon.accent`, no `0x24` literal, no `ReportClassStyle.review` foreground, Stop with `.cancelAction`), `ContractTests.titleCase`.
- [x] `make app-test` passes. The unsigned `xcodebuild` build passes. — evidence: the work landed in af8c590 and shipped in 0.2.2, whose `make dist` then required `app-test` and builds the app with `xcodebuild … CODE_SIGNING_ALLOWED=NO` before signing (`scripts/build-app.sh`); there was no CI Gate yet. Today the Gate's `app` job runs `make app-test`, green on main at 2cdae98 (run 41191429282, 2026-10-31).
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): on the second Mac, where the bug was seen: with a terminal frontmost, the first click on the icon opens the popover active, with blue switches and a blue Clean selected. The log line after the click shows the app active and the window key.
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): on the owner's build:
  - **Keyboard:**
    - Esc in a non-empty filter clears it and the window stays open. Esc with an empty filter closes the window and opens the popover.
    - Return while typing a filter opens nothing. With an empty filter it opens the Clean sheet.
    - During a clean, Esc shows "Stopping…".
    - With keyboard navigation on, Tab moves through the popover with a visible ring.
    - ⌘F focuses the filter.
  - **Colour:**
    - With a red system accent, the Review footer's "Clean N items…" is red. With Multicolor it is the brand blue.
    - Warning text is readable in Dark Mode.
  - **Menus and motion:**
    - Help → DeepDriveCleaner Help opens the README.
    - With Reduce Motion on, the menu bar arc does not turn during a scan.

## Final Summary

**What changed.** A pass over the app against Apple's Human Interface Guidelines, shipped in 0.2.2 (af8c590).
- The popover opens active: its window is made key after it shows, and again when the app becomes active later; `app.log` records the app's and the window's state after each click.
- Keyboard: Esc clears a non-empty filter before it closes the Review window, Return never opens the Clean sheet while typing a filter, the cleaning sheet's button reads Stop and takes Esc, the confirmation's Clean is destructive and not the default, the popover's first focus is Review….
- Colour: an `AccentColor` asset holds the brand blue and the system accent applies everywhere; warnings use the class ink, readable in Dark Mode.
- `ContentUnavailableView` for the Review window's empty states; Services, Hide Others, Show All, Zoom, Bring All to Front, Find (⌘F) and a Help menu; title-style capitalization; the next-scan time in the user's 12/24-hour format; monospaced timers; the menu bar arc stands still with Reduce Motion.

**Why.** The owner saw the popover open inactive on a second Mac, and the review of 0.2.0 found keyboard, colour and wording gaps against the guidelines.

**Tests.** `PopoverModelTests.nextScan`, `ContractTests.colours`, `ContractTests.titleCase`; the 0.2.2 release build; the Gate's `app` job on main today. Closed under the owner's rule of 2026-10-30: the two manual checks moved to the runbook "Owner checks: manual acceptance across stories".

**Risks and follow-ups.**
- The popover-activation fix has not been checked on the second Mac, where the bug was seen; that is a manual check in the runbook, and its `app.log` line says which half failed if the fix is not enough.
- The README's title-case wording is kept by hand; no test checks it.

## Technical Notes

- **Audit.** It was read-only (source files only) and ran on 2026-10-21. Its file:line references are in the Decomposition. A critic pass checked the references and the HIG claims:
  - [HIG Buttons](https://developer.apple.com/design/human-interface-guidelines/buttons) asks for title-style capitalization.
  - [`Color.accentColor`](https://developer.apple.com/documentation/swiftui/color/accentcolor): on macOS an app's accent applies only under Multicolor.
  - Stop for an operation in progress follows Apple's apps (Finder, Xcode, Safari); it is not a quoted HIG rule.
- **Return and Esc in the filter.** These come from AppKit's key-equivalent order and were not observed. The manual check confirms them before and after the fix.
- **The Settings window stays as it is.** It is a hand-made `NSWindow`, which is fine with the AppKit lifecycle (`app/DeepDriveCleaner/main.swift`). It closes with ⌘W and, as system windows do, not with Esc. "Settings: folders, types, notifications" rebuilds it.
- **Glass.** The app sets no `glassEffect` and should not. The popover and the sheets get macOS 26 materials from the system. The Review window gets them once it has a real toolbar and sidebar.

## Dependencies

- **"Review and clean window" (DDC-002).** Its cleaning sheet's Cancel becomes Stop, which affects the ACs on stopping a clean. Its Escape rule (owner, 2026-10-19) stands, with the filter-field exception above.
- **No CLI change.**

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-31*
