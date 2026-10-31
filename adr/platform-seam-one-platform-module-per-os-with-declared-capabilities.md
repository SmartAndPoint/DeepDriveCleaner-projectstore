---
type: adr
id: "platform-seam-one-platform-module-per-os-with-declared-capabilities"
title: "Platform seam: one platform module per OS with declared capabilities"
status: accepted
date: 2026-10-30
authors: ["Evgenii Konev"]
tags: [rust, architecture, portability, schema, cli, safety]
external_refs: {}
supersedes: null
superseded_by: null
review_status: approved
reviewed_at: 2026-10-30
---

# Platform seam: one platform module per OS with declared capabilities

| Field | Value |
|---|---|
| **Status** | accepted |
| **Date** | 2026-10-30 |
| **Authors** | Evgenii Konev |

---

## Context

[[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]] makes `deepdrivecleaner` run on Linux and Windows. Today the core and the CLI are macOS-only, and nothing in the repository is gated by `cfg(target_os)`:
- `ddc-core/src/sys/` holds the FFI: `bulk` (`getattrlistbulk`), `fsops` (verified-parent operations), `procs` (`libproc`), `statfs`, `iopolicy`, `rlimit`, `signals`, `term`, `usage`, `op_stats`.
- `libc` and `std::os::unix` are also used in `scan/`, `report/`, `exec/`, `rules/matcher.rs` and `disk/`.
- `ddc-cli` schedules with launchd (`launchd.rs`) and notifies through `osascript` (`notify.rs`). `main.rs::init_process` sets the macOS I/O policy and the open-file limit.

Some macOS features have no counterpart elsewhere: APFS private size, TCC (Full Disk Access), dataless cloud files, Time Machine local snapshots, the whole-disk view with simulators, launchd, `osascript`.

Three needs follow:
1. **A seam the move can go behind** without changing macOS behaviour or speed. The walker calls the directory listing once per directory, millions of times a scan (Rust core ADR: 2.24 million files in 6.8–8.8 s).
2. **Reports that say what their platform could not measure.** Without this, a reader takes an absent `private_bytes` as "no clones" and `full_disk_access: false` as a permission to grant. The epic's success criterion: "It states in the JSON which macOS-only features are unavailable."
3. **A half-ported platform must not scan or delete.** This includes source builds (`cargo install --git`, binstall's compile fallback), which never pass our release gate.

Seams that exist today are traits, there for test fakes:
- `exec::{CommandRunner, Trasher, ProcessTable}`, implemented in `exec/system.rs`;
- `report::volumes::StatFs`, `scan::reader::DirReader`, `scan::Visitor`;
- `exec::journal::{Journal, AreaJournal}`;
- in `ddc-cli`, `notify::Notifier`.

Exit codes are 0, 1, 2 and 75 (`ddc-cli/src/exit.rs`; [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §6 item 5). Under `--json` the CLI has no error output: every refusal is a `ddc: …` line on stderr.

## Decision

### 1. One module per OS, chosen at compile time, with one set of signatures

- `platform` has one child module per OS, chosen with `#[cfg(target_os = …)]` and re-exported as `platform::*`. Shared code calls `platform::statfs(path)`, never `platform::macos::…`. Dispatch is static: no `dyn` and no generic parameter on the hot path.
- Every OS module provides the same items with the same signatures. `platform/mod.rs` pins each signature once, for example `const _: fn(&Path) -> io::Result<StatFs> = imp::statfs;`. A module that drifts fails `cargo check --target <t>`, which the gate runs for every target (story "Move OS calls behind the platform module").
- Data types shared by the OS modules are defined once in `platform/` (`platform/mod.rs`, or `platform/types.rs` when it grows), never per OS. These are the listing entry, `StatFs`, process info, `Platform`, `Capability` and `Unsupported`.
- When an OS cannot provide an item, the item still exists there. What it returns depends on the item:
  - An item that backs a capability (§3) returns `io::ErrorKind::Unsupported` (or the type's "unknown" value), and that capability is absent from the list. The absence is stated in the report, never silent.
  - An item in §3's "not capabilities" list that is a no-op on this OS (the I/O policy calls, the open-file limit) returns `Ok`, and the OS module's doc comment says it is a no-op. So `init_process` prints no warning on Linux or Windows, and `ddc rules` writes nothing to stderr there.
- Traits stay only where tests need a fake, as today:
  - `CommandRunner`, `Trasher`, `ProcessTable`, `StatFs`, `DirReader`, `Visitor`, `Journal`, `AreaJournal`;
  - `Notifier` in `ddc-cli`.

  Their `System*` implementations call `platform::`. There is no trait for the platform as a whole.
- A target other than macOS, Linux or Windows does not compile: `compile_error!` in `platform/mod.rs`.

Amended 2026-10-30, story "Move OS calls behind the platform module" (epic DDC-007): the surface is `platform::<module>::<item>` (`platform::statfs::statfs(path)`, `platform::procs::…`, `platform::signals::…`), mirroring today's `sys::` modules, not one flat namespace. The move is then a mechanical rename, and each pin names its module: `const _: fn(&Path) -> io::Result<VolumeStats> = imp::statfs::statfs;`. Everything else in this section holds. Per-OS values such as errno numbers are per-OS items, never shared types: `EDEADLK` is 11 on macOS and 35 on Linux, where 11 is `EAGAIN`.

### 2. Where the code lives

`ddc-core`:
- `src/platform/mod.rs`: the shared types, `Platform`, `Capability`, `SUPPORTED`, `require_supported`, the signature pins and the `cfg` selection.
- `src/platform/macos/`: today's `sys/*`, moved by the story "Move OS calls behind the platform module". Until then `platform/macos.rs` holds only the constants of §3 and §5.
- `src/platform/linux.rs` and `src/platform/windows.rs`, each becoming a folder when it grows.
- `unsafe_code` is allowed only under `platform/`. This takes over the Rust core ADR's rule for `sys`; the move story records it there as an amendment.
- `libc::`, `rustix::`, `windows_sys::`, `std::os::unix`, `std::os::windows`, `cfg(target_os)`, `cfg(unix)` and `cfg(windows)` appear only under `platform/`.

`ddc-cli`:
- `src/platform/{mod,macos,linux,windows}.rs`, crate path `crate::platform`. It holds launchd, the `osascript` notifier and `init_process`'s process setup.
- It is not under `crate::cli`, which is clap's argument definitions. The epic's stories call it "`cli::platform`".
- `ddc-cli` and `ddc-loadgen` reach OS calls only through `ddc_core::platform::*` (the surface `cfg` selects) or `crate::platform`, never through `ddc_core::platform::macos`.

Amended 2026-10-30, story "Move OS calls behind the platform module" (epic DDC-007):
- `ddc-loadgen` gets its own folder, `src/platform/{mod,unix,windows}.rs`. Its tree builder uses `rustix`'s `*at` calls, and every CLI test depends on it, so it must compile on every target.
- The folders are `ddc-core/src/platform/`, `ddc-cli/src/platform/` and `ddc-loadgen/src/platform/`.
- The list of what appears only inside them widens to `libc::`, `rustix::`, `windows_sys::`, `windows::` and `std::os::` (any submodule, including `std::os::macos` and `std::os::fd`), and to every `cfg`, `cfg!` and `cfg_attr` on `target_os`, `target_family`, `unix` or `windows`.
- `unsafe_code` is allowed in `platform/macos/` and in the shared `platform/op_stats.rs` (the allocation counter), not in `platform/mod.rs`, `types.rs` or the stubs.

Amended 2026-10-31, story "Linux scanner and rule catalog" (epic DDC-007): `platform/linux.rs` becomes the folder `platform/linux/`, and `unsafe_code` is also allowed there, for the calls `rustix` does not wrap (`wait4` with its resource usage, signal handlers), each block with a `// SAFETY:` comment.

### 3. Capabilities

`Capability` is a closed enum, kebab-case in JSON, in this declaration order:

| Capability | What the platform does | When it is absent |
|---|---|---|
| `private-size` | Measures the clone-aware private size where clones are expected (APFS) | `private_bytes` is never present. "Frees" equals allocated. |
| `in-use-check` | Before a delete, reads processes' open files, working folders, executables, arguments and `VIRTUAL_ENV`/`CONDA_PREFIX` (safety ADR §5) | See the rule below this table |
| `dataless-files` | Recognises evicted cloud files and never downloads them while scanning | The platform's catalog lists its cloud-sync roots as not walked |
| `full-disk-access` | A permission system (TCC) can withhold parts of the home folder | `full_disk_access` is `true`: nothing is withheld by a permission. Unreadable folders are still listed in `inaccessible`. |
| `local-snapshots` | Counts local backup snapshots that delay freeing space (Time Machine) | A clean report's `local_snapshots` is 0 and has no meaning |
| `whole-disk` | `ddc disk` (the areas, containers, snapshots and simulators report) and the `disk free` actions on its rows: `simctl-runtime-delete`, `conda-env-remove`, `conda-clean` | `ddc disk` refuses with `capability-absent` (§7); so does a `disk free` list naming one of those actions |
| `coding-agents` | `ddc disk agents`: the coding agents' folders and records | `ddc disk agents` refuses with `capability-absent` |
| `homebrew` | `ddc disk brew-packages` and the `disk free` actions on its rows: `brew-cleanup`, `brew-uninstall`, `brew-uninstall-orphans` | `ddc disk brew-packages` refuses with `capability-absent`; so does a `disk free` list naming one of those actions |
| `schedule` | `ddc schedule` (launchd) | `ddc schedule` refuses with `capability-absent` |
| `notifications` | The background scan notifies (`osascript`) | `scan --background` decides but does not show anything; `--notify json` still prints the decision |

- **Per platform.** macOS declares all ten. Linux and Windows are decided by their catalog stories. The expectation is `in-use-check` on Linux (`/proc`), and on Windows whatever its research note supports. `coding-agents` and `homebrew` are separate from `whole-disk` because they do not depend on APFS, Time Machine or simulators: Linux can declare them without the whole-disk report.
- **Linux** (amended 2026-10-31, story "Linux scanner and rule catalog"): declares `in-use-check` only, with the limits the safety ADR's 2026-10-31 amendment names. The other nine follow their "when absent" column.
- **`disk free` is gated per action.** Each action needs the capability in the table. A list that names an action whose capability is absent is refused whole, with `capability-absent` naming that capability (§7). At the terminal, `free::available` offers only the actions whose capability is declared. The command itself is refused only when neither `whole-disk` nor `homebrew` is declared.
- **Where `capability-absent` is checked.** The dispatcher refuses whole commands (`disk`, `disk agents`, `disk brew-packages`, `schedule`). The per-action check for `disk free` lives in the core (`free::run`, `free::dry_run`, `free::available`), so the terminal prompt and the app's list agree. `disk::Sources` (the tool paths: `diskutil`, `tmutil`, `simctl`, `brew`) is macOS-shaped today; each OS module provides its own, so Linux can name Linuxbrew without the others.
- **A Windows check that answers only sometimes** (the Restart Manager) is declared and reports "unknown" per item, or is not declared. The Windows story picks one; it is never probed per run (next bullet).
- **Static, not probed.** `platform::CAPABILITIES: &[Capability]` is a constant of the OS module. It says what this build can do on its OS, not what this run found. Facts about a run already have fields:
  - `full_disk_access`;
  - each volume's `fs_type`;
  - `private_bytes` per finding (absent on a volume that is not APFS even with the capability);
  - `simulators` (absent without developer tools).

  A static list also keeps golden outputs the same on every machine. If a feature depends on the machine, the platform either declares it and reports "unknown" per item, or does not declare it.
- **Not capabilities.** These are present on every supported platform, or a no-op that does not change what a report means:
  - directory listing and the verified-parent operations;
  - allocated size and file identity;
  - free space;
  - the command runner's group kill;
  - the I/O policy (priority and the dataless-materialization setting), the open-file limit, signals, terminal width;
  - the home, data and log folders;
  - the Trash.

  Each OS module says in its doc comment what it does for each. A platform that cannot list directories or do verified-parent operations is not supported (§5); these are never "missing capabilities".
- **`in-use-check` is part of the safety model.** If it is absent, that is no permission to delete without the check. A platform without it can be flipped to supported only together with an amendment of [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]] §5 that names the guard replacing it. On Windows that guard is a sharing violation at delete time, reported as held open and skipped.
- **A new capability is a new enum value.** It bumps the schema version of every report that carries the list (§4), under the Rust core ADR's versioning policy. Splitting the disk commands into three capabilities now is meant to keep that rare.

### 4. The JSON

- Three reports gain two fields, placed right after `ddc_version`:
  - `platform`: `"macos"`, `"linux"` or `"windows"`, equal to the build target's `std::env::consts::OS`;
  - `capabilities`: the declared list, in declaration order.

  The reports are the scan report (`model::ScanReport`), the clean report (`exec::report::CleanReport`) and the disk report (`disk::model::DiskReport`). Both fields are required: no `serde(default)` and no `skip_serializing_if`.
- **Versions:**
  - `SCHEMA_VERSION` goes from 1 to 2;
  - `CLEAN_SCHEMA_VERSION` from 3 to 4;
  - `DISK_SCHEMA_VERSION` from 2 to 3.

  The Rust core ADR keeps the version for new *optional* fields. These fields are required, and they change how other fields read: a version-1 report was macOS by assumption, and a version-2 report is read against its `capabilities` (the "when absent" column of §3).
- **The schemas** get the `$defs` `Platform` and `Capability`. The names are fixed because the app's contract test looks them up.
- **The app** decodes both fields strictly, as the menu bar ADR §1 and §6 require for values ddc assigns itself.
- **`full_disk_access`** keeps its type. Its doc comment and schema description become "true unless a permission system withheld protected data", in `model.rs`, `docs/scan.schema.json` and `ScanReport.swift`.
- **File identity stays `(dev: u64, ino: u64)`** in the scan, clean and items-list schemas. Each OS module defines what the pair holds:
  - macOS: `st_dev` and `st_ino`;
  - Linux: `stx_dev` and `stx_ino`;
  - Windows: the volume serial number and the 64-bit NTFS file id.

  A Windows file whose 128-bit id (ReFS) does not fit in 64 bits is never planned for deletion; the Windows story says how it is reported. The `changed-since-scan` check compares the pair, so its width is part of the safety model and is not widened silently.
- **Unchanged:** the rules, notify, progress, agents, brew-packages and disk-free outputs, the settings file and the journal (the journal has its own version).
  - `NotifyReport` gets no capability list: a background run's decision does not depend on one.
  - `rules --json` gets `platform` in story "Rule catalog split: shared and per-OS sections", which adds `--platform` and bumps the rules schema there. Until then an unported platform prints the macOS catalog, as `rules` is the one command it runs.
    Amended 2026-10-30, story "Rule catalog split: shared and per-OS sections" (epic DDC-007): that story adds `--platform` and the per-OS sections but leaves the rules JSON as it is, with no `platform` field and schema version 1, so macOS output stays golden-equal and the app, which refuses any other rules schema, keeps working. A reader of `--platform` knows which platform it asked for. The field and the bump go with story "Linux scanner and rule catalog": its flip of `SUPPORTED` is the first time two platforms print different real catalogs. Since that story, an unported platform prints its own section (the shared rules).
- **Further bumps in this epic are accepted.** If the Linux story's representation of non-UTF-8 paths changes the scan schema, it bumps it again. This ADR does not decide that representation.
- **After the upgrade,** `last-scan.json` and `last-disk.json` from the previous version read as absent (`store::decode_last_scan`, `read_last_disk`), as after every bump. The app shows "written by another version" until its next scan.

### 5. The `SUPPORTED` switch

- `platform::SUPPORTED` is the OS module's literal constant, and false in a build with `simulate-unported` (§8): `pub const SUPPORTED: bool = imp::SUPPORTED && !cfg!(feature = "simulate-unported");`.
  - macOS: `true`.
  - Linux and Windows: `false`, until each catalog story flips its literal in the same commit that enables its release archives.
  - No environment variable, settings key, command-line flag or feature can make it true.
- **Core.** `platform::require_supported() -> Result<(), Unsupported>` is the first statement of every core entry point that walks the disk or changes anything:
  - `scan::walker::walk_with_reader`, which also covers `scan::walk`, `ddc du`, the disk report's walks and conda;
  - `report::scan_inner`, which covers every `report::scan*`, before the excludes are resolved or a volume is read;
  - `disk::measure_with` (and so `disk::measure`), which now returns `Result<DiskReport, Unsupported>`. Its walks fold errors into partial areas, so without this a refusal would come back as a report.
  - `exec::preflight::preflight`, the dry run the app relies on, as `io::ErrorKind::Unsupported`. Its walk would refuse anyway, but it folds walk errors away and opens folders and runs in-use checks before that;
  - `exec::execute`, before the root check;
  - `free::run`, the executor of `ddc disk free`, as `io::ErrorKind::Unsupported`.

  - **The error type.** `platform::Unsupported { platform: Platform, simulated: bool }` reads "deepdrivecleaner does not support <platform> yet". In a `simulate-unported` build it reads "this build was made with simulate-unported and refuses every scan and clean". It carries `simulated` so the message is never false.
  - **The variants.** The error enums gain `WalkSetupError::Unsupported(Unsupported)`, `ScanError::Unsupported(Unsupported)` and `ExecError::Unsupported(Unsupported)`. Each holds the struct as a field, not as a `#[source]`, so `ddc: {err:#}` prints the message once. `scan_inner` refuses before any walk, so a scan's refusal is always `ScanError::Unsupported`, never `ScanError::Walk(WalkSetupError::Unsupported(_))`.
  - **Not gated in the core.** These only read, and they are reached only through commands the dispatcher already refuses:
    - `agents::report`, which reads the agents' records;
    - the brew listing;
    - `free::available` and `free::dry_run`, which run read-only tools.

    The core gate's promise is therefore "nothing walks a tree or deletes", not "nothing reads".
- **CLI.**
  - `--help`, `--version` and usage errors are handled by clap first (exit 0, 0 and 2).
  - Next, before `init_process` (the process setup is OS code too) and before any `Env`, lock or signal handler, the dispatcher refuses every command except `rules` when `SUPPORTED` is false. That covers `scan` (with `--background`), `clean`, `du`, `disk` and its subcommands, `schedule`, `notify` and `diag`.
  - The match over `Command` is exhaustive, with no wildcard, so a new command has to choose.
  - A core refusal that reaches `main` (a path the dispatcher missed) exits with the same code. `ddc_core::platform::refusal_in(err: &(dyn std::error::Error + 'static))` walks `source()` and recognises `Unsupported` itself, the three variants above, and an `io::Error` whose `get_ref()` downcasts to `Unsupported`. The kind alone is never enough: std maps `ENOSYS` and `EOPNOTSUPP` to `io::ErrorKind::Unsupported`, and the core already returns that kind for other reasons. `preflight` and `free::run` build their refusal with `io::Error::new(ErrorKind::Unsupported, unsupported)`. `ddc-core` has no `anyhow`, so `main` passes `err.as_ref()`, beside its `LockError::Busy` check.

### 6. Exit code 69

The CLI exits 69 (`exit::UNSUPPORTED`; `EX_UNAVAILABLE` in sysexits.h: "a service is unavailable") for `platform-not-supported` and, later, for `capability-absent`.
- **Not 2.** 2 means "refused: change the request", and it is clap's usage error. No other request helps here.
- **Not 1.** A script must tell "this platform cannot" from "this run failed".
- **Why sysexits.** The CLI's 75 (`EX_TEMPFAIL`) already follows sysexits.h, and 69 is the code whose meaning fits. Rejected:
  - 78 `EX_CONFIG`: there is nothing to configure;
  - 71 `EX_OSERR`: that is a failed system call, not a missing port;
  - an unassigned number such as 3: it means nothing outside ddc.
- **No collisions.** 69 is none of 0, 1, 2 or 75, Rust's panic exit 101, the shell's 126 and 127, or 128 + a signal number. The CLI passes no child's exit status through.
- **The app** runs only on macOS, where its embedded CLI is supported. `RunOutcome` maps 69 to `.error(status: 69)` with no new case.
- README's exit-code table and the menu bar ADR's list in §6 item 5 gain 69 (amendment).

### 7. The JSON error envelope

```json
{
  "error_schema_version": 1,
  "error": {
    "code": "platform-not-supported",
    "message": "deepdrivecleaner does not support linux yet: only --version, --help and rules run",
    "platform": "linux"
  }
}
```

- **The type** is `ddc_core::refusal::ErrorReport`: `error_schema_version`, plus `error: ErrorBody { code: ErrorCode, message: String, platform: Option<Platform>, capability: Option<Capability>, action: Option<ActionId>, simulated: bool }`. `ERROR_SCHEMA_VERSION` is 1.
  - `action` (the disk-free schema's strict `ActionId`) names the `disk free` action that needs the absent capability, for example `{"code": "capability-absent", "capability": "homebrew", "action": "brew-uninstall", …}`.
  - `simulated` is true only in a `simulate-unported` build, so an agent reading `code` and `platform: "macos"` there can tell the refusal is a test build's.
- **The schema** is `docs/error.schema.json`, generated by schemars and kept current by a drift test like the other schemas.
- **`code`** is a strict enum:
  - `platform-not-supported`: used from this story on;
  - `capability-absent`: carries `capability`; first used when a supported Linux refuses `ddc disk` or `ddc schedule`.

  A new code bumps `ERROR_SCHEMA_VERSION`. `message` is for people and is never parsed.
- **Readers** branch on the exit code first, then on the `error` key. The version key is named `error_schema_version`, not `schema_version`, so no report decoder can read the envelope's header as its own and then fail on the body.
- **Where it goes.** The envelope goes to stdout, where the report would have gone, when the command's JSON output was asked for:
  - `scan --json`, `scan --background --notify json`;
  - `clean --json`;
  - `disk --json`, `disk agents --json`, `disk free --json`, `disk brew-packages --json`.

  `--progress json` on its own is not JSON output: its events go to stderr. The `ddc: …` line goes to stderr in every case. Commands without a JSON mode (`du`, `diag`, `schedule`, `notify`) print only that line.
- **Scope.** This is the shape for every `--json` refusal from now on. The existing refusals (exit 2) and the busy exit (75) keep their stderr-only output here. Moving them is a later change, reviewed against the menu bar ADR, which documents those paths.

### 8. A test build with `SUPPORTED = false`

- **The feature.** `ddc-core` gets the Cargo feature `simulate-unported`, and `ddc-cli` forwards it (`simulate-unported = ["ddc-core/simulate-unported"]`). It can only turn the switch off: a user who sets it gets a binary that refuses everything, never one that deletes on an unported platform. That is why a feature is acceptable here and an environment variable is not. The rule is "nothing a user or an agent sets can turn it on", and this turns it off.
- **`cfg!()`, not `#[cfg]`,** in `platform/mod.rs`, so both branches type-check in every build.
- **`make unported-check`** builds into `target/unported`, so `target/debug/deepdrivecleaner` stays the normal binary. It runs only the test files written for this build, which compile to nothing otherwise (`#![cfg(feature = "simulate-unported")]`):
  - `crates/ddc-core/tests/unported.rs`;
  - `crates/ddc-cli/tests/unported_cli.rs`.

  It also runs clippy on that build. It is part of `make gate`.
- **Never enabled by accident.** The feature is never in the Makefile's `FEATURES`, and nothing builds with `--all-features` (op-stats and dhat-heap already rule that out). Structure tests assert:
  - no `Cargo.toml` dependency enables the feature, and no build or release script passes it;
  - `platform/mod.rs` reads no environment (`env::var`, `env::var_os`, `option_env!`), and its only `cfg!(feature …)` is `simulate-unported`;
  - in the OS modules, only the folders item (home, data, log) reads the environment, and only these names: `HOME`, `USERPROFILE`, `HOMEDRIVE`, `HOMEPATH`, `LOCALAPPDATA` and `XDG_*`. The test allowlists exactly those. The rule constrains `platform/` only; whether `Env` also carries `APPDATA` or `TEMP` (Windows protected folders, the catalog's temp folder) is decided by the catalog split or Windows story, and the allowlist grows with it;
  - each OS module's `SUPPORTED` is a literal `true` or `false`, and its `CAPABILITIES` a literal slice, never a function call.

  A release binary built with the feature would also fail the release smoke test: `release.yml` (from story "Build matrix: every target built, tested and packaged from one tag") requires `scan --json` on a generated tree to succeed. The seam story adds the same scan of a small temporary tree to `scripts/build-cli.sh` (same origin), so the macOS archive is checked too.

### 9. Tests on a platform that is not supported yet

Core tests that call a gated entry point skip when `!platform::SUPPORTED`, through one helper in `tests/common` that prints why. While a platform is unported, the build matrix's native Linux and Windows jobs therefore run only the platform-neutral tests; they run everything once the catalog story flips the switch. On macOS a normal-build test asserts that `SUPPORTED` is true, so nothing there ever skips.

The rule for choosing between the two mechanisms: a test of something macOS-only by nature (APFS clones, TCC, launchd, `getattrlistbulk`) is compiled only there with `#[cfg(target_os = "macos")]`. A test of a gated entry point that will run on the other platforms after their switch flips uses the runtime skip.

*Amendment, 2026-10-30 (story "Build matrix: every target built, tested and packaged from one tag", stage 2).* The runtime skip has four channels. Each decides from `PLATFORM` at run time, so none ever skips on macOS:
- `unsupported()`: a gated entry point, or unported platform code.
- `lacks(Capability::X)`: a test of a declared capability the platform lacks. It stays meaningful after a switch flips.
- An in-crate `#[cfg(test)] skipped()` beside `require_supported`: for unit tests under `src/`, which cannot reach `tests/common`.
- Tool helpers that return `Unsupported` where a tool the test needs is missing. In-source unit tests use the `#[cfg(test)]` `platform::testing` (`plutil`, symlinks). Integration and CLI tests use their `tests/common` twins (symlinks, a pty driver), which may use an OS `cfg` under the rule below.

Each prints `skipped <test>: <reason>`. The native jobs run under nextest with passing output shown, and compare those lines with a committed per-OS list (`.config/native-skips/`). A new skip is therefore a reviewed diff, not a silent loss of coverage. The `#[cfg(target_os = "macos")]` rule above stays for tests that are macOS-only by nature.

*Amendment, 2026-10-31 (story "Linux scanner and rule catalog", the flip).* With Linux supported, besides `lacks`, three kinds of test stay skipped there by design, and the skip channels name them:
- `no_rule(test, id)` in `tests/common`: a test of a rule this platform's catalog does not list (Xcode, the iOS backups, Telegram, the cloud folders, app bundles). `has_rule(id)` lets a test of the shared rules add a platform's own rule to what it checks, without skipping. Tests of shared rules build their fixtures where this platform's catalog looks (`rule_path(id)`), not at macOS paths.
- `off_macos(test)`, beside the goldens (`golden_cli.rs`) and the pinned trees (`determinism.rs`): they are macOS's recordings, so another platform skips them. Its load trees are built from its own catalog (`ddc_loadgen::fixtures::catalog`), and its load profiles check them against their manifests.
- More tool helpers of the fourth channel: `user_immutable` (no user-settable immutable flag off macOS), `try_clone_file` (no `cp -c`), and a pseudo-terminal driver per OS.

`no_rule` and `off_macos` join the helpers whose labels the structure test checks. A test of something only one platform has by nature is compiled only there, whichever platform that is: the freedesktop Trash under a temporary `HOME` is tested with `#[cfg(target_os = "linux")]`, as the macOS-only tests are with theirs.

## Rationale

1. **No layer that buys nothing.** A binary only ever holds one platform. A `cfg`-chosen free function compiles to a direct call, inlinable under thin LTO. A `dyn Platform` would add a second dispatch layer under the walker's existing `DirReader` and `Visitor` ones, and a generic `P: Platform` would spread a type parameter through `scan`, `report`, `exec` and `disk`, both for no gain.
2. **One signature set, checked by the compiler.** The signature pins plus `cargo check` for every target catch drift between OS modules. That is the guarantee a trait would give, without dynamic dispatch.
3. **Fakes keep working.** The traits tests already use stay where they are. Only their `System*` implementations change what they call.
4. **The reports state their limits.** A static, closed capability list keeps golden outputs the same everywhere, lets the app decode strictly, and keeps run facts in the fields that already hold them.
5. **The switch fails safe.** It is a literal per OS, checked in the core as well as at dispatch. The only build knob turns it off. A source build of a half-ported platform therefore refuses before it walks a tree, runs a preflight or deletes anything, and so does a future caller of the core (the app over FFI, `background`). The few read-only entry points listed in §5 are refused by the dispatcher only.
6. **One refusal shape for agents.** A stable exit code plus a JSON envelope lets an agent tell "unsupported here" from "failed" without parsing prose.

## Alternatives Considered

### Alternative A: a `Platform` trait with one implementation per OS (`dyn` or generic)

**Pros**:
- Any OS could be faked in a test.
- A single type to pass around.

**Cons**:
- A second dispatch layer under `DirReader` and `Visitor`, or a type parameter everywhere.
- The behaviour worth faking is already behind narrower traits.
- One binary never holds two platforms.

**Rejected because**: it costs speed and plumbing and buys only fakes that the narrow traits already provide.

### Alternative B: capabilities probed at run time

**Pros**:
- Can answer "not on this volume" or "this permission is missing".

**Cons**:
- Golden outputs would depend on the machine.
- It duplicates fields that already exist (`full_disk_access`, `fs_type`, per-finding `private_bytes`).
- One meaning ("what can this build do") would turn into two.

**Rejected because**: run facts already have fields; the capability list answers a different question.

### Alternative C: optional fields, no version bump

**Pros**:
- Old `last-scan.json` and `last-disk.json` files stay readable.
- No app release has to accompany the change.

**Cons**:
- The capabilities change how existing fields read (`private_bytes`, `full_disk_access`, `local_snapshots`). A reader that ignores an optional field would read them as before.
- The story asks for the bump.

An absent `platform` itself would be unambiguous: only macOS ever wrote version-1 reports. That is not the reason.

**Rejected because**: a report's meaning changes, and the versioning policy bumps the version for meaning changes. The cost is that every user's saved `last-scan.json` and `last-disk.json` read as absent once.

### Alternative D: an environment variable, a settings key or an enabling feature as the switch

**Pros**:
- Developers could exercise an unported platform end to end.

**Cons**:
- Anything a user or agent sets could let a half-ported platform delete.

**Rejected because**: the story forbids it. The only knob (§8) turns the switch off.

### Alternative E: `cfg` attributes scattered through the code, `sys/` kept for macOS beside a new `sys_linux/`

**Pros**:
- The smallest diff.

**Cons**:
- There would be no single place where the OS signatures are listed.
- "No `libc` outside one folder" could not be enforced by a grep.
- DDC-006's shape gate would see the same functions three times in different folders.

**Rejected because**: it gives no seam to check.

### Alternative F: capabilities as a map `{ "private-size": true, … }`

**Pros**:
- Absent capabilities are explicit.

**Cons**:
- Every reader has to handle unknown keys.
- The schema cannot list the values as an enum.
- The list form already implies the absent ones through the schema's enum.

**Rejected because**: the list with a closed enum is stricter and simpler to decode.

### Alternative G: exit code 2 (refused) or 1 (error)

**Rejected because**: see §6. Neither lets a script tell this case apart.

### Alternative H: an open set of capability strings

**Pros**:
- A Linux- or Windows-only capability would bump no schema and need no app release. The app always ships with its own CLI, so strict decoding of the list buys it little.

**Cons**:
- A typo or a renamed capability goes unnoticed by every reader.
- The schema could not list what a reader must handle.
- Agents reading the JSON could not rely on a fixed vocabulary.

**Rejected because**: the closed enum keeps the vocabulary checked, and splitting the disk commands (§3) keeps the bumps rare.

## Consequences

**Positive**:
- The move story has a target layout and a rule it can grep for.
- Linux and Windows add a module rather than editing shared code.
- Reports say what they could not measure, and the app decodes them strictly.
- An unported or half-ported build refuses before it walks a tree, runs a preflight or deletes anything, whether through the CLI, `background` or a library call. The CLI refuses the remaining read-only commands too.

**Negative / trade-offs**:
- The scan, clean and disk schemas each bump once. Saved `last-scan.json` and `last-disk.json` read as absent until the next scan. Until then:
  - the app shows "written by another version";
  - the first `ddc disk` walks the home folder itself (`disk/home.rs` finds no saved scan), so it is slower once;
  - `clean --items` finds no previous scan and does the full walk once instead of the pruned one.
- The golden files under `bench/golden/` change: `make golden-check` fails until `make golden-record`, and the story checks that the diff adds only the two fields and the version.
- A new capability bumps three schemas.
- `disk::measure` becomes fallible, and three error enums gain a variant.
- Unported platforms skip most core tests until their catalog story flips the switch.
- One more gate step: `make unported-check` is a second build of `ddc-core` and `ddc-cli` into `target/unported` before every push. The story measures what it adds to the pre-push gate (about 2 minutes today).

**What changes in code / process**:
- New: `ddc-core/src/platform/` (this story: `mod.rs`, plus `macos.rs`, `linux.rs` and `windows.rs` holding constants only); `ddc-core/src/refusal.rs`; `docs/error.schema.json`; `exit::UNSUPPORTED = 69`.
- The dispatch gate goes in `ddc-cli/src/main.rs` before `init_process`.
- `make unported-check` joins `make gate`.
- The story "Move OS calls behind the platform module" moves `sys/` to `platform/macos/` and creates `ddc-cli/src/platform/`. It amends the Rust core ADR's `unsafe` rule from `sys` to `platform/`. Its acceptance criterion that names "`cli/platform` (in `ddc-cli`)" is reworded to `ddc-cli/src/platform/` (`crate::platform`) when this ADR is accepted.
- Each catalog story flips its platform's literal, declares its capabilities, and (if `in-use-check` is absent) amends the safety ADR §5. The Windows story records how a file id wider than 64 bits is reported (§4).
- The menu bar ADR §6 item 5 and README's exit-code table gain 69.

## References

- [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]: the versioning policy and the `unsafe` rule.
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §5, the in-use checks and verified-parent operations.
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §1 and §6 strict decoding, §6 item 5 exit codes.
- sysexits.h (`EX_UNAVAILABLE` 69, `EX_TEMPFAIL` 75).

---

*Last updated: 2026-10-30*
