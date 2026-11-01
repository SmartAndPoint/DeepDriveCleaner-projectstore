---
type: adr
id: "windows-app-tauri-2-shell-over-the-ddc-cli"
title: "Windows app: Tauri 2 shell over the ddc CLI"
status: accepted
date: 2026-11-01
authors: ["Evgenii Konev"]
tags: [app, windows, tauri, rust, architecture, installer]
external_refs: {}
supersedes: null
superseded_by: null
review_status: approved
reviewed_at: 2026-11-02
---

# Windows app: Tauri 2 shell over the ddc CLI

| Field | Value |
|---|---|
| **Status** | accepted |
| **Date** | 2026-11-01 |
| **Authors** | Evgenii Konev |

---

## Context

[[epics/DDC-008/epic|DDC-008 · Native installers and the Windows tray app]] planned a minimal tray icon on Windows: a status line, Scan now, a schedule and a notification, with review and clean left to the CLI. On 2026-11-01 the owner widened it: the Windows app has the same menu and windows as the macOS app (full parity), and it is built with **Tauri 2**. The owner picked Tauri over Slint, WinUI 3 and a bare `tray-icon` + `muda` menu after this comparison:

- A bare tray menu (the earlier proposal) cannot show what the macOS popover shows: a disk bar, groups with sizes, agents, a Review window with sortable rows, a confirmation sheet.
- WinUI 3 gives the most native Windows 11 look, but brings a second language and toolchain (.NET), a separate gate, and builds only on Windows.
- Slint keeps one toolchain and is small, but has no tray, no positioned popup and no installer of its own, and its royalty-free licence asks for attribution.
- Tauri 2 has a tray, positioner, single-instance, autostart and updater plugins, and its bundler makes a per-user NSIS setup. The web view (WebView2) ships with Windows 11. Its UI also runs on macOS, which helps with layout work; tray placement, toasts, sign-in start and the setup are Windows-only and still need the owner's Windows machine.

The macOS app is the model: [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] (called "the macOS ADR" below). It holds no cleaning logic. It runs the bundled CLI with `--json`, decodes the reports, and keeps its view logic in `DDCKit`, a package unit-tested apart from the UI. This ADR keeps that shape and says what changes on Windows.

The Windows CLI must be supported first: DDC-007 story "Windows scanner and rule catalog" flips `SUPPORTED` for Windows. The app is built against a fake CLI until then.

## Decision

### 1. A Tauri 2 app that runs the installed CLI

- `DeepDriveCleaner.exe` is a Tauri 2 app. It holds no cleaning logic. It runs `bin\deepdrivecleaner.exe` installed beside it, with `--json`, and reads:
  - the saved report `<data-dir>\last-scan.json` directly, for an instant panel;
  - the output of `scan`, `clean`, `disk`, `disk agents` and `disk free` on stdout;
  - progress events on stderr.
- Every CLI child is spawned with `CREATE_NO_WINDOW`, so no console flashes.
- The app and the CLI ship in one setup and one version, so the app reads the `schema_version` it was built for. A mismatch is shown as "the report could not be read", never decoded loosely (macOS ADR §1).
- **One runner, the same lanes.** All CLI runs go through one runner with the queue and lanes of macOS ADR §1 and its amendments: one queue for everything that takes the ddc lock, a lane each for `disk` and `disk agents`, joins by identical arguments, Stop by ticket.
- **Stop is cooperative, as SIGTERM is on macOS** (the CLI also ends the running native command's Job Object, as it forwards the signal to the tool's process group on macOS). Windows has no SIGTERM, and `TerminateProcess` would end a clean like a crash (the journal, but no report). The CLI therefore takes a hidden option `--cancel-event <name>`: it opens that named Win32 event, and a watcher thread sets the same flag the macOS signal handler sets, so a clean stops after the current item and writes its report. The app creates the event per run and signals it on Stop. A `scan` that does not stop within its grace period is ended with `TerminateProcess`: a scan writes nothing before it finishes. The option is added by DDC-007 story "Windows scanner and rule catalog", step 8 (signals and the command runner).
- **The CLI is not linked.** The app does not depend on `ddc-core`. `ddc-app-model` (§2) has its own serde types for the parts of each report it reads, as `DDCKit` has its own `Codable` types. Its tests decode the CLI's golden outputs and the `DDCKit` fixtures, so a schema change that breaks the app fails a test (macOS ADR Alternative A).
- **Why the CLI sits in `bin\`.** Windows paths are case-insensitive: `DeepDriveCleaner.exe` and `deepdrivecleaner.exe` would be the same file, the reason the macOS bundle keeps the CLI in `Contents/Helpers`. `bin\` is also the only folder that goes on `PATH` (§7), so the app's own executable is never on it.

### 2. Where the code lives

```
crates/ddc-app-model/   # member of the root workspace: serde only, no Tauri, no OS calls
app-windows/
  Cargo.toml            # its own [workspace] and Cargo.lock: the Tauri shell only
  src-tauri/            # crate deepdrivecleaner-app
  ui/                   # static HTML, CSS and ES modules, no build step
  deny.toml             # the shell's own licence policy (below)
```

- **`ddc-app-model` is a root workspace member.** It is the counterpart of `DDCKit`: report types, the runner's queue rules, the panel, review, confirmation, disk, settings and scheduled-work models, wording, formatting, the notification text. It has no dependency beyond `serde`/`serde_json`, so it builds and is tested on macOS, Linux and Windows. As a root member it runs in every Rust push's gate (clippy, nextest, coverage floor, `deny.toml` unchanged). Its decoding tests read `crates/` goldens and `app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/` (already off the rust-free list), so a CLI schema change that breaks the app fails the push that makes it.
- **The Tauri shell is a separate workspace.** Tauri pulls hundreds of crates and, on Linux, needs the WebKitGTK system libraries. In the root workspace it would enter `cargo check --workspace` on every target and the core's licence policy. It depends on `ddc-app-model` by path.
- **`deepdrivecleaner-app`** is the shell: windows, tray, process spawning, notifications, sign-in start. It stays thin; a rule that grows past wiring moves into the model. Its command handlers are tested in-process with Tauri's mock runtime (`tauri::test`), not through a browser driver.
- **`ui/`** is plain HTML, CSS and ES modules served from disk (`frontendDist`), with `withGlobalTauri`, so no bundler and **no Node or npm**. The scripts render the view models the shell sends and send back the user's actions; they hold no rules. `cargo tauri` (`tauri-cli`) joins `make tools`.
- **Gate.**
  - `scripts/gate-scope.sh classify` gains a fourth answer, `win-app`, for a push whose paths are all under `app-windows/`. `.githooks/pre-push` runs `make win-app-test` for it; `scripts/tests/test_gate_scope.py` gets its cases.
  - A Gate job `win-app` builds and tests the shell on `windows-2025` on every push, without waiting for scope, as the `app` job does, with its own build cache key. Adding it to the required checks of `main` is the owner's step.
  - `app-windows/deny.toml` holds the shell's licence list. Tauri's tree is expected to bring licences beyond the root list (`MIT`, `Apache-2.0`, `Unicode-3.0`), for example MPL-2.0 through `option-ext`. The shell story runs `cargo deny` on a Tauri hello world first, and the exact list, with the owner's answer on MPL-2.0 (file-level copyleft, in a separate binary of an AGPL product), is recorded here as an amendment before the shell merges.

### 3. UI: a tray icon, a panel and three windows

The macOS app's surfaces map one to one (macOS ADR §2):

| macOS | Windows |
|---|---|
| `NSStatusItem` with a template glyph, paused grey, a corner mark | A tray icon (Tauri `TrayIconBuilder`) in light and dark variants for the taskbar theme, grey while paused, a dot for "needs you" or the badge after a notification. The `.ico` files are committed: `scripts/icons/` is Swift and does not run in the Windows job |
| `NSPopover` with SwiftUI | **The panel**: a frameless, non-resizable window with no taskbar button, placed by `tauri-plugin-positioner` at `TrayCenter` (above the icon, below it when the taskbar is at the top), constrained to the work area for a taskbar at a side; hidden when it loses focus |
| Right-click `NSMenu`: Scan now, Pause or Resume scans, Settings, Quit | A native tray context menu with the same items |
| Review window, Whole disk window (`DiskWindowController`), Settings window | Three normal windows with a taskbar button while open |
| The Dock rule while a titled window is open (macOS ADR §2, 2026-10-20) | Not needed: a normal window has a taskbar button and an Alt-Tab entry |

- A left click on the tray icon toggles the panel; a right click opens the context menu. A click that arrives within 300 ms of the panel hiding on focus loss does not reopen it (the panel hides on the mouse-down that is the same click).
- The look follows Windows 11: Segoe UI Variable, the system light or dark theme and the accent colour, rounded corners, Mica where the window effects API offers it. Fluent's spacing and control shapes are copied in CSS; no web component library is added.
- The panel's web view is created once and kept hidden. If idle memory with a hidden web view is more than the budget in §9, the shell destroys it on hide and recreates it on show; the dogfooding story measures which.
- Accessibility: native HTML controls with labels, read by Narrator through WebView2's UI Automation bridge; Tab order, Enter and Space, Escape closes the panel and dialogs.

### 4. Identifiers and where state lives

- **AppUserModelID and Tauri identifier:** `com.smartandpoint.deepdrivecleaner`, the macOS bundle id. Tauri's NSIS template sets it on the Start menu shortcut. A development build or a hand-copied app registers it instead under `HKCU\Software\Classes\AppUserModelId\com.smartandpoint.deepdrivecleaner` (display name, icon), so toasts work without the setup.
- **The bare CLI** posts no notification on Windows (seam ADR §3, `notifications` absent), so it needs no AUMID.
- **The setup's upgrade key:** the NSIS bundler's per-user uninstall key, derived from the identifier.
- **The URL protocol** `deepdrivecleaner:` (§6), registered per user under `HKCU\Software\Classes\deepdrivecleaner`.
- **The sign-in entry:** the `Run` value named `DeepDriveCleaner`. The setup and `tauri-plugin-autostart` write the same name and data, so Settings shows the true state.
- **App-only preferences** (the badge, permanent by default, working hours, the size in the tooltip; `UserDefaults` on macOS) live in Tauri's app config folder, `%APPDATA%\com.smartandpoint.deepdrivecleaner\preferences.json`. Settings the CLI reads stay in its `settings.json`.

### 5. Scheduling: Task Scheduler through `ddc schedule`

This differs from macOS on purpose:

- The app does not run an hourly tick and has no due function. It calls `deepdrivecleaner schedule install --interval daily|weekly --at <hour>` and `schedule uninstall` (story "Windows scheduled scans through Task Scheduler"). The CLI writes one per-user task whose action is `DeepDriveCleaner.exe --scheduled`, with "run as soon as possible after a missed start". Task Scheduler handles the slots and the catch-up after sleep or shutdown that the macOS due function computes.
- Pause scans disables the task (`schedule pause`), Resume enables it (`schedule resume`). A disabled task does not catch up the slots it missed, so on Resume the app runs one scheduled scan at once if the last scheduled run is older than one interval, as macOS runs a missed slot at the next tick after Resume.
- **Remind me tomorrow** with a weekly schedule: there is no scheduled scan the next day, so `notify snooze` from the app also calls `schedule remind --at <time>`, which adds a one-shot trigger to the task for the schedule hour on the day after the tap (macOS ADR §5's reminder scan).
- The app reads the interval, the hour, the paused state and the next run time from `schedule status --json`. The next run time comes from the Task Scheduler API (`IRegisteredTask::NextRunTime`), never from localized `schtasks` text.
- The app writes no `scheduler` record in `settings.json`: on Windows the CLI owns the task whether the app or a terminal asked for it.

**The `--scheduled` start.** The task starts `DeepDriveCleaner.exe --scheduled`. That process does **not** register the single-instance plugin, so it never holds the instance and never swallows a user launch (macOS ADR §5's invariant):

1. If a resident instance runs (it holds the named mutex `Local\com.smartandpoint.deepdrivecleaner.resident`), `--scheduled` starts `DeepDriveCleaner.exe --run-scheduled` and exits. That start goes through the single-instance plugin, which forwards it to the resident instance. The resident runs the scheduled scan through its own queue, so it never overlaps a Scan now, and posts the toast itself. If the resident quit between the check and the start, `--run-scheduled` finds no instance to forward to and runs exactly as a headless `--scheduled` run (item 2); it never creates the tray.
2. Otherwise `--scheduled` runs headless: no tray, no window, no web view. It reads the schedule from `schedule status --json`, runs `scan --background --notify json`, applies the app's working-hours rule (§6), posts the toast if the decision says so, records it with `notify ack`, and exits. Its toast's buttons work after it exits (§6).

A user launch while a headless `--scheduled` run is going starts a normal resident instance. The two do not race on `last-scan.json`: the CLI's lock makes a second scan wait or fail as busy, and the resident shows the report when it lands.

### 6. Notifications

- A Windows toast with the macOS content: "12.4 GB can be freed", the top categories, and two buttons, **Review** and **Remind me tomorrow**. The decision rule, thresholds, snooze and low-disk rules stay in Rust in the CLI (macOS ADR §5, §6 item 9).
- **Working hours are the app's rule**, as on macOS (`DDCKit` `ScheduledWork`, `AppPreferences.workingHoursOnly`), ported to `ddc-app-model`. A decision made outside working hours is kept as pending in the app's preferences folder. A resident instance posts it at the first moment inside working hours (one timer, no poll); with no resident instance, the next start of the app or the next scheduled run posts it. A newer decision replaces a pending one.
- The toast is posted through the WinRT `ToastNotificationManager` with the AUMID of §4, from toast XML the shell builds itself (`XmlDocument::LoadXml`, then `CreateToastNotifier(aumid).Show`, through the `windows` crate). Neither `tauri-plugin-notification` (its action buttons are mobile-only) nor `tauri-winrt-notification` (it sets no activation type and activates only in-process) can emit protocol activation.
- **Buttons by protocol activation.** Both buttons, and a click on the toast's body (the root's `launch`, which is `deepdrivecleaner:review`), use `activationType="protocol"` with the URIs `deepdrivecleaner:review` and `deepdrivecleaner:snooze`. Windows starts `DeepDriveCleaner.exe "<uri>"` whether or not the posting process still lives, so a click from the Action Center days later still works. That start goes through the single-instance plugin: with a resident instance it is forwarded there; without one, the started process handles it. `review` becomes a normal resident instance with the Review window open; `snooze` runs `notify snooze` (and `schedule remind`, §5) and exits.
- The handler accepts an invocation only when its argv is exactly `[exe, "deepdrivecleaner:review"]` or `[exe, "deepdrivecleaner:snooze"]`, and `--run-scheduled` only as `[exe, "--run-scheduled"]`; anything else is refused whole, which also closes quote-breaking tricks on `%1`. Neither verb removes anything; a web page that invokes them can at most open Review or snooze a notification. The registry entry is `HKCU\Software\Classes\deepdrivecleaner\shell\open\command` = `"<exe>" "%1"`; `tauri-plugin-deep-link` may write and check it instead of hand-written code.
- COM activation (a `ToastActivatorCLSID` and a registered COM server) is not used: protocol activation reaches a closed app without it.

### 7. Setup: Tauri's per-user NSIS bundler, with one template change

- `installMode: currentUser`: no administrator rights.
- **Install folder `%LOCALAPPDATA%\Programs\DeepDriveCleaner\`.** The stock template installs to `%LOCALAPPDATA%\<productName>`, which is `%LOCALAPPDATA%\DeepDriveCleaner`, the CLI's data folder (`last-scan.json`, the journal, `settings.json`). Sharing one folder would make a clean uninstall impossible without deleting the user's reports. The bundle therefore uses a custom `bundle.windows.nsis.template`. The template belongs to `tauri-bundler`, which is compiled into `tauri-cli`, not to the `tauri` crate: `make tools` pins `tauri-cli` (`cargo install tauri-cli --version <x> --locked`), the upstream `installer.nsi` of that version is vendored beside the custom one with its SHA-256, and an offline test diffs the two and fails on any difference but one line, `.onInit`'s `StrCpy $INSTDIR "$LOCALAPPDATA\${PRODUCTNAME}"` for `currentUser` (the top-level `InstallDir` is a placeholder; `RestorePreviousInstallLocation` then keeps the folder on upgrades). Both are bumped together. A second test asserts the install folder and the data folder differ.
- Layout: `DeepDriveCleaner.exe`, the UI resources, `bin\deepdrivecleaner.exe`, `bin\ddc.exe`.
- `installerHooks` (NSIS):
  - **install:** add `...\DeepDriveCleaner\bin` to the user's `Path` (`HKCU\Environment`) and broadcast `WM_SETTINGCHANGE`; register the `deepdrivecleaner:` protocol; write the `Run` value if the person kept "Start at sign-in"; on an upgrade, rename a running `bin\*.exe` to `*.old` before the copy (the bundler's own check closes only `DeepDriveCleaner.exe`), and the app at its next start and the next setup's pre-install hook delete `*.old`;
  - **uninstall:** run `bin\deepdrivecleaner.exe schedule uninstall` first; remove exactly the `Path` element, the protocol, the `Run` value; leave the CLI's data folder, the app's preferences folder and the WebView2 profile unless the person ticks "Remove my settings and reports", which removes all three.
- `webviewInstallMode: downloadBootstrapper`: Windows 11 already has WebView2.
- Silent switches (`/S`) for winget come from NSIS.
- Signing comes from DDC-008 story "Windows signing and distribution route: research and ADR", through the bundler's `signCommand`, for the app, the CLI and the setup.
- An MSIX for the Store, if that ADR picks one, is a second package from the same binaries, with a `StartupTask` and its AUMID from the package.

### 8. Consent and safety

- The confirmation window is interactive consent, as on macOS (macOS ADR §7). The app runs a removing command only after the person confirmed that exact list there, and only with these arguments, as `DDCKit` runs them:
  - `clean --items <file> --json --progress json` (`CleanRun`);
  - `disk free --actions <file> --json`, after its `--dry-run` preview (`DiskFreeRun`); the sheet shows each action's exact command and what it cannot undo (macOS ADR §7, amended).
- Nothing is cleaned on a schedule, at start, from a toast or through the protocol (the owner, 2026-10-28: scans inform, people remove).
- **What can be put back.** As on macOS, `review` items and picked user data go to the Recycle Bin; `safe` and `rebuildable` caches and build outputs are deleted, because they are rebuilt. "Delete permanently" sends `review` items past the Recycle Bin too. An item the Recycle Bin cannot take (over its size limit) fails and is reported; it is never deleted permanently in its place (DDC-007 story "Windows scanner and rule catalog", step 9).
- A test lists every argv the app may build and fails on any other. It is the app's one safety gate, written against the real CLI options.
- The safety rules themselves (risk classes, protected folders, held-open files) are the CLI's; the app shows what the CLI reports, including "held open, skipped".

### 9. Budgets

- Resident, panel hidden: no timer under 60 seconds and no file polling while hidden; the shell wakes on a change notification of the data folder, a CLI event, its working-hours timer or its next-run line. The shell's timers are registered through one model-level registry, and a test asserts this while hidden.
- Memory with the panel hidden is measured in dogfooding against the macOS app's resident memory. If the kept web view costs more than about 30 MB over a bare tray, §3's destroy-on-hide applies.
- Setup size: a few MB without the WebView2 runtime.

### 10. Updates

Out of scope for the first release, as the DDC-008 epic says. The candidate is `tauri-plugin-updater` with a signed feed beside Sparkle's, decided in its own story once the signing route is known.

## Rationale

- One language for the core, the CLI and the Windows shell. A future Linux tray (DDC-010) can reuse the model and most of the shell.
- The macOS app's split, a model package tested apart from a thin UI, carries over: `DDCKit` becomes `ddc-app-model`, and its tests are a checklist for parity.
- The model in the root workspace keeps the app honest against every CLI change; the shell apart keeps Tauri out of the core's gate.
- No JavaScript toolchain: the UI is small, static and rule-free.

## Alternatives Considered

### Alternative A: WinUI 3 (C#)

The most native Windows 11 look. Rejected: a second language and runtime, a separate test stack, builds only on Windows, a tray only through a third-party package.

### Alternative B: Slint

One toolchain and a small footprint. Rejected: no tray, no positioned popup and no installer of its own, and the royalty-free licence asks for attribution.

### Alternative C: `tray-icon` + `muda` with a native menu only

The earlier DDC-008 proposal. Rejected by the owner's parity decision.

### Alternative D: the whole app, Tauri included, in the root workspace

One `Cargo.lock`. Rejected: every core push would build Tauri, Linux CI would need WebKitGTK, and the core's licence policy would have to cover Tauri's tree. Only the Tauri-free model joins the root workspace (§2).

### Alternative E: the model inside the separate workspace too

Keeps all app code in one folder. Rejected after review: the model's drift tests would then run only on app pushes, not on the Rust push that changes a schema, and the gate would need a second coverage and lint setup for it.

### Alternative F: link `ddc-core` into the app

Fewer types to keep in step. Rejected for the macOS ADR's reasons: the CLI is the one place the safety rules run, the JSON schemas are the contract, and a separate process keeps a scan's memory out of the resident app.

### Alternative G: a frontend framework with a bundler (React, Svelte, Vite)

Faster to write large views. Rejected for now: Node in the gate and a dependency tree to audit, for a UI whose logic lives in Rust. A later ADR can revisit it if the Review window outgrows plain modules.

### Alternative H: a long-lived `--scheduled` process that waits for its toast

The first draft. Rejected after review: a button clicked from the Action Center after the process exits would reach nobody, a process that waits would have to promote itself to the full app on Review, and if it held the single-instance mutex it would swallow a user launch. Protocol activation (§6) removes all three.

## Consequences

- Positive: a Windows user gets the macOS app's panel, review and clean, with the same consent and the same reports.
- Positive: the model is tested on every OS and on every Rust push.
- Negative: a second implementation of the `DDCKit` rules, in Rust. A parity inventory (shell story) maps every `DDCKit` source file to a Windows story or a reason it has none; each rule change on macOS needs its counterpart.
- Negative: the look is Fluent-like CSS, not WinUI controls; WebView2 memory while a window is open.
- Negative: one custom NSIS template line to carry across Tauri upgrades, guarded by a test.
- Neutral: two CLI additions for the app on Windows: `--cancel-event` (step 8 of the scanner story) and `schedule pause|resume|remind|status --json` (the Task Scheduler story).

## Open Questions

- [ ] Mica through Tauri's window effects in the first release, or later (shell story).
- [ ] Kept versus destroyed web view (§3, §9), measured in dogfooding.
- [ ] The shell's licence list, including MPL-2.0 (§2): the owner answers before the shell merges.

## References

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[adr/platform-seam-one-platform-module-per-os-with-declared-capabilities|Platform seam: one platform module per OS with declared capabilities]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[epics/DDC-008/epic|DDC-008 · Native installers and the Windows tray app]]
- Tauri 2: the Windows installer guide (install modes, installer hooks, WebView2 modes), the NSIS template (`installer.nsi`, `utils.nsh`), the tray icon, the positioner, single-instance and autostart plugins, the notification plugin; `tauri-winrt-notification` (read 2026-11-01).
