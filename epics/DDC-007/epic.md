---
type: epic
id: "DDC-007"
title: "Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers"
status: planned
priority: p2
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, cli, linux, windows, macos, distribution, release, ci]
code_refs: []
review_status: reviewed
reviewed_at: 2026-10-28
---

# DDC-007: Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers

| Field | Value |
|---|---|
| **Status** | planned |
| **Priority** | p2 |
| **Created** | 2026-10-28 |
| **Updated** | 2026-10-28 |

---

## Goal

A developer installs the `deepdrivecleaner` CLI (`ddc`) without Homebrew and without the app, on macOS, Linux or Windows, with one command. On every platform it finds that platform's dev junk under the same safety model. Every release produces tested, checksummed CLI archives for all targets from one commit. The build is the backbone of this epic.

Success criteria:

- One release has a CLI archive per target, plus `SHA256SUMS` over all of them:
  - `aarch64-apple-darwin`, signed and notarized by the owner as today;
  - `x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl`, static, each with a build provenance attestation;
  - `x86_64-pc-windows-msvc`, with a build provenance attestation.
  
  Each CI-built binary was packaged only after the gate for its commit passed on its own operating system.
- On macOS and Linux, `curl -fsSL https://ddc.sh/install.sh | sh` gives a working `deepdrivecleaner --version` and `ddc --version`, after the `PATH` line the script prints. Running it again upgrades. The same holds on Windows for the route story "Windows install: PowerShell script, winget and a signed binary" decides (the proposal is `irm https://ddc.sh/install.ps1 | iex`).
- `cargo install --git https://github.com/SmartAndPoint/DeepDriveCleaner --locked ddc-cli` gives `deepdrivecleaner` on every platform. The README gives the one command that adds the `ddc` link.
- `ddc scan ~ --json` on Linux and Windows finds the platform's package caches and project artifacts. It never follows a symlink, junction or mount point, and never plans a protected folder. It states in the JSON which macOS-only features are unavailable.
- Until a platform's catalog story is done, the CLI on that platform runs only `--version`, `--help` and `rules`, and refuses every other command with a stable exit code. A source build cannot bypass that.

## Context

The core is macOS-only today. The repository has no `cfg(target_os)`, `cfg(unix)` or `cfg(windows)` gate, so it does not compile for Linux or Windows. The macOS specifics are spread across the code:
- `sys/` holds the FFI: `getattrlistbulk` (`bulk.rs`), `libproc` (`procs.rs`), `statfs`, verified-parent file operations (`fsops.rs`), I/O policy, rlimits, signals, terminal.
- `libc` and `std::os::unix` are also used outside `sys/`:
  - `scan/walker.rs` and `scan/reader.rs` (errno mapping);
  - `report/backups.rs` (`O_NOFOLLOW`) and `report/contents.rs` (`EACCES`, tests only);
  - `exec/system.rs` (`SIGKILL` of process groups);
  - `exec/delete.rs`, `exec/guard.rs`, `rules/matcher.rs`, `report/after_clean.rs`, `disk/mod.rs`, `disk/visitor.rs`;
  - 13 test files, including `tests/common/`.
- `disk/*` reads APFS, Time Machine, simulators and other macOS areas.
- `env.rs` takes the home from `HOME`, which is normally unset on Windows. The data and log folders are under `~/Library`.
- `ddc-cli` schedules with launchd (`launchd.rs`) and notifies through `/usr/bin/osascript` (`notify.rs`).
- The rule catalog (`rules/catalog.rs`) names `~/Library` paths. Project artifacts (`node_modules`, `target`, `.venv`) and many tool caches (`~/.cargo`, `~/.npm`, `~/.gradle`, `~/.m2`) are the same everywhere.

Releases today:
- `scripts/build-dist.sh` builds only `aarch64-apple-darwin`, a flat tarball with `deepdrivecleaner`, `LICENSE.md`, `DISCLAIMER.md`, `README.md`. It signs with the Developer ID, notarizes, and refuses a dirty tree or a binary containing `$HOME`.
- `scripts/publish-release.sh` creates the tag itself at the built commit. It refuses a tag that already exists on origin without a release.
- The Homebrew cask carries the app with the CLI inside. It requires Apple Silicon and macOS 26 (`depends_on macos: :tahoe`). The CLI tarball's floor is its own deployment target, which starts at macOS 26 (the only version it has run on). A script install becomes the route for older macOS on Apple Silicon as far as the owner lowers that floor after a run on the older version.
- The crates have `publish = false`.

The owner's decisions of 2026-10-28:
- **One-line install script.** `curl … ddc.sh/install.sh | sh` for the CLI on macOS and Linux, and something similar on Windows, possibly a signed installer. This is decision 3 in [[research/distribution-through-agents|Distribution through agents]], which withdrew that note's §5 "not an install script" for the CLI. The concerns of §5 shape the script: checksum verification, no `sudo`, non-interactive, and the manual route shown next to it.
- **Signing stays with the owner.** The macOS Developer ID signature and notarization stay a local step; team signing is off the vault. See [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], §8.

The website story "English and Chinese product website with development runbook" (DDC-004) plans a CLI page for macOS, Linux and Windows. It shows a download button only where a release asset exists. This epic produces those assets and the scripts. That story's acceptance criteria currently name only `brew install --cask` as the install command, so they need the script added once this epic delivers it.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-one-line-install-on-macos-and-linux-ddc-sh-install-sh-and-cargo](./stories/story-one-line-install-on-macos-and-linux-ddc-sh-install-sh-and-cargo.md) | planned | `curl -fsSL https://ddc.sh/install.sh \| sh`: checksum-verified, no sudo, stable path, upgrade in place, refuses next to a Homebrew or app install. Plus `cargo install --git` and cargo-binstall. macOS works with today's tarball |
| [story-build-matrix-every-target-built-tested-and-packaged-from-one-tag](./stories/story-build-matrix-every-target-built-tested-and-packaged-from-one-tag.md) | planned | Starts thin now: the macOS gate and a Linux/Windows `cargo check` in CI. Grows into native test jobs per OS and release archives as workflow artifacts that `publish-release.sh` collects into one release |
| [story-platform-seam-adr-and-capability-model](./stories/story-platform-seam-adr-and-capability-model.md) | planned | ADR: one platform module per OS, declared capabilities, `platform`/`capabilities` in the JSON, app decoder updated, a `SUPPORTED` switch that refuses unported platforms |
| [story-move-os-calls-behind-the-platform-module](./stories/story-move-os-calls-behind-the-platform-module.md) | planned | Every `libc`/`std::os::unix`/macOS call in `ddc-core` and `ddc-cli` moved behind `platform::macos`. Linux/Windows stubs. The test harness ported or marked. `cargo check` for all targets |
| [story-rule-catalog-split-shared-and-per-os-sections](./stories/story-rule-catalog-split-shared-and-per-os-sections.md) | planned | The catalog in a shared section and per-OS sections, with data and log folders per OS. macOS output byte-identical (golden test) |
| [story-linux-scanner-and-rule-catalog](./stories/story-linux-scanner-and-rule-catalog.md) | planned | Research note first. Then: the Linux walker, `/proc` in-use checks, freedesktop Trash, XDG catalog, protected folders. Flips Linux to supported and publishes its archives |
| [story-windows-scanner-and-rule-catalog](./stories/story-windows-scanner-and-rule-catalog.md) | planned | Research note first. Then: a handle-based walker with name-surrogate reparse points never followed, handle-based verified-parent deletes, Job Objects, `USERPROFILE` home, long and case-insensitive paths, Recycle Bin, `%LOCALAPPDATA%` catalog |
| [story-windows-install-powershell-script-winget-and-a-signed-binary](./stories/story-windows-install-powershell-script-winget-and-a-signed-binary.md) | planned | Research and an ADR on Smart App Control, SmartScreen and code-signing options with costs. Then `install.ps1`, a winget portable package, and signing once the owner picks a route |

Suggested order:
1. The macOS half of the install story and the thin slice of the build matrix. Both need only today's code, plus one release that carries `SHA256SUMS` and DDC-005's `gate.yml`.
2. The seam ADR, then the move behind the platform module, then the catalog split.
3. Linux.
4. Windows, then its install story.

## Expected Results

- [ ] `curl -fsSL https://ddc.sh/install.sh | sh` installs the CLI on macOS and Linux, checksum-verified, without sudo.
- [ ] `cargo check --workspace --target <t>` passes for macOS, Linux (gnu and musl) and Windows.
- [ ] CI runs the test suite natively on macOS, Ubuntu and Windows on every push.
- [ ] A release carries archives for all supported targets, `SHA256SUMS`, and attestations for the CI-built ones.
- [ ] The CLI works on Linux and Windows with a platform catalog and the same safety rules. Unported platforms refuse.
- [ ] Windows has a decided and working install route.

## Dependencies

- [[epics/DDC-005/epic|DDC-005 · Production-grade Rust core: mandatory gate and load testing]]: its gate workflow is what the build matrix extends. Its property and load tests guard the seam refactor. Its load thresholds are macOS numbers; each platform sets its own.
- [[epics/DDC-006/epic|DDC-006 · Code shape gate: measurable conventions that guide humans and agents]]: new platform code is born inside the shape limits. Its burn-down story runs after the move behind the platform module.
- [[epics/DDC-004/epic|DDC-004 · Product website and agent-first contribution guide]]: serves the install scripts at `ddc.sh` and the per-target download buttons.
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: applies unchanged. Each platform adds its protected folders (§6) and implements its verified-parent operations and in-use checks (§5).

## Open Questions

- [ ] Scheduled scans and notifications on Linux (systemd user timer, `notify-send`) and Windows (Task Scheduler, toast): in a later epic, or here? Proposal: later. Here `ddc schedule` reports "not supported on this platform" with a stable exit code and JSON.
- [ ] Publishing to crates.io (`cargo install ddc-cli` / `cargo binstall ddc-cli`; PolyForm-Noncommercial-1.0.0 is a valid SPDX id) and Linux package managers (`.deb`/`.rpm`, AUR, Homebrew on Linux): later epics?
- [x] The `ddc` short name off macOS: no second `[[bin]]`. A second binary on macOS would break the one-binary Full Disk Access design, and macOS TCC keys CLI grants by path. `cargo install` produces `deepdrivecleaner` on every platform. `ddc` is a symlink (Unix) or a hard link (Windows), created by the install scripts, `make install`, the cask, and the README's manual steps. Decided 2026-10-28 at authoring, on the critic's evidence.
- [x] An Intel Mac (`x86_64-apple-darwin`) asset: no. The cask and every measurement are Apple Silicon only. The script states that Intel Macs are unsupported. Decided 2026-10-28 at authoring; the owner can reopen it.

## Related

- [[epics/DDC-001/epic|DDC-001 · Disk space cleaner CLI for developers]]
- [[epics/DDC-004/epic|DDC-004 · Product website and agent-first contribution guide]]
- [[research/distribution-through-agents|Distribution through agents]]: decision 3, the install script.
- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]: the Linux and Windows catalogs need the same kind of evidence.

---

*Last updated: 2026-10-28*
