---
type: adr
id: "rust-core-library-and-cli-with-json-output"
title: "Rust core: library + CLI with JSON output"
status: accepted
date: 2026-10-17
authors: ["Evgenii Konev"]
tags: [stack, rust, cli]
external_refs: {}
supersedes: null
superseded_by: null
review_status: reviewed
reviewed_at: 2026-10-17
---

# Rust core: library + CLI with JSON output

| Field | Value |
|---|---|
| **Status** | accepted |
| **Date** | 2026-10-17 |
| **Authors** | Evgenii Konev |

---

## Context

DeepDriveCleaner starts as a terminal utility, `ddc`: find, group and carefully remove dev junk (see [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]). Later it will grow into a Mac app for distribution. Hence the requirements:

1. **Speed.** Walking hundreds of gigabytes and millions of files (on the reference machine, `__pycache__` alone occurs 7,830 times) must take seconds or tens of seconds, not minutes. The user explicitly asked for "blazingly fast".
2. **One binary with no runtime.** A user of the future app must not have to install Python or Node.
3. **The core is reused in the GUI.** The scanning logic, the rules and the cleanup model must not be rewritten when we move to the Mac app.
4. **Stable machine-readable output.** Background mode and the future GUI read the scan result programmatically.

The decision is needed now: all the code depends on the language.

## Decision

We write the core in **Rust**. A Cargo workspace of two crates:

- `crates/ddc-core` (library `ddc-core`): parallel filesystem walk, rule catalog, findings model, cleanup plan construction, plan executor. It does not print to the terminal and does not read stdin.
- `crates/ddc-cli` (package `ddc-cli`, binary `deepdrivecleaner` with a symlink `ddc`, `clap`): commands `scan`, `clean`, `schedule`; a hidden debug command `du` for cross-checking against `du` and for measurements; later `dupes`. Human-readable output by default; the `--json` flag gives a versioned schema (`schema_version`). The schema is generated from the types (`schemars`) into `docs/scan.schema.json`; a test compares the committed file with the generated one.

The filesystem walk is our own parallel walker on `rayon` on top of `getattrlistbulk(2)`: one system call returns a batch of directory entries with name, type, `fileid`, `devid`, link count, allocated size and (with `FSOPT_ATTR_CMN_EXTENDED`) `ATTR_CMNEXT_PRIVATESIZE`. Prototype on `~/Projects` (2.24 million files): `getattrlistbulk` — 6.8–8.8 s, `std::fs` + `lstat` — 14.3 s, `du -skx` — 47 s; the total matches `du` byte for byte. Off-the-shelf walkers (`jwalk`, `ignore`) also offer a per-directory hook with subtree pruning, but they are built on `std::fs::read_dir` + `lstat` and give neither the speed of `getattrlistbulk` nor the APFS private size. That is the reason for our own walker.

For each finding the core computes **allocated** (the sum of `ATTR_FILE_ALLOCSIZE`). **Frees at least** (the sum of `ATTR_CMNEXT_PRIVATESIZE`, field `private_bytes`) is computed only where APFS clones are expected: the uv, pnpm and bun caches; venvs created by `uv venv` (`uv =` in `pyvenv.cfg`); `node_modules` installed by pnpm (`.pnpm`/`.modules.yaml` inside) or by bun (`bun.lock*` next to it). Private size triples the cost of reading a directory: with it enabled in all removable findings, `ddc scan ~` took 68–80 s instead of 35 s (2026-10-17, three runs). In the other findings clones are not expected, `private_bytes` is absent, and the freed amount is taken to equal the allocated amount. The "frees" total in the report and in the plan is an estimate (`~`), not a bound: it is too low when both sides of a clone are selected (the venv and the uv cache each show ≈ 0), and too high where clones exist but are not measured (for example, `uv pip install` into a venv not created by uv; copies made with `cp -c`/`ditto`). Hard links (`nlink > 1`) are only recorded during the walk, in a `(dev, ino) → owners` table, and are resolved deterministically after the walk: if all links of an inode are found inside one finding, the bytes go to its `bytes`; otherwise they go to the `shared_bytes` of each involved finding. In the grand total an inode is counted once, as in `du`. Two consecutive runs give the same result.

A directory is opened by path with `O_DIRECTORY | O_NOFOLLOW`, and then the descriptor's `fstat` is checked against the `(dev, ino)` from the parent's listing. On a mismatch (a swap between listing and opening) or a different `st_dev` (a mount point), the directory is not walked. With materialization turned off, opening a dataless directory returns `EDEADLK`; this is "skipped (dataless)", not an access error. Entries with the `SF_DATALESS` flag in `ATTR_CMN_FLAGS` are not opened.

Data paths are not hard-coded in the core: the `Env` struct carries the home directory, the data directory (`--data-dir` / `DDC_DATA_DIR`) and the log directory. Tests always pass temporary directories and never touch the real `$HOME`.

The integration with the future Mac app is recorded as a direction, not as a decision: as a first step SwiftUI runs `ddc --json`; later, if needed, it links `ddc-core` through FFI (UniFFI). We will record that choice in a separate ADR when we start the app.

Toolchain: rustup stable, with the version pinned in `rust-toolchain.toml`. The Homebrew `rust` package on the reference machine is broken (`cargo` cannot find `libgit2.1.8.dylib`) and comes before rustup in `PATH`.

## Rationale

1. **Performance without compromise.** Native code with no GC; `rayon` gives work-stealing parallelism across directories; the bottleneck is system calls, and `getattrlistbulk` cuts their number by an order of magnitude (measurement above).
2. **One binary.** `cargo build --release` produces a self-contained executable; we will get a universal build for arm64 and x86_64 with `lipo`.
3. **The core is a separate library.** The `ddc-core` / `ddc` boundary forces us to keep the logic free of any tie to the terminal, so the GUI will get the same core.
4. **Ecosystem.** Mature `clap`, `rayon`, `serde_json` and `trash` exist; FFI to Swift is UniFFI.
5. **The user chose Rust** (2026-10-17).

## Alternatives Considered

### Alternative A: Swift (Swift Package: library + CLI)

**Pros**:
- The same language as the future Mac app; no FFI needed.
- Direct access to Foundation and macOS APIs (notifications, Trash, `getattrlist`).

**Cons**:
- A filesystem walk through `FileManager` is slower; to catch up with Rust we would have to drop down to POSIX APIs by hand.
- The core is tied to Apple platforms.

**Rejected because**: the user chose Rust for speed; Swift's advantage (no FFI) pays off only at the app stage, and it is solved by UniFFI or by calling the CLI.

### Alternative B: Python

**Pros**:
- The fastest way to write the first version.

**Cons**:
- Needs an interpreter; on a clean macOS, `python3` is a stub that offers to install the Command Line Tools.
- Slow startup and slow walk; for the GUI the core would have to be rewritten, or the interpreter embedded.

**Rejected because**: it does not meet requirements 1–3.

### Alternative C: Go

**Pros**:
- Fast, one binary, simple concurrency.

**Cons**:
- FFI to Swift through cgo is awkward; there is a GC and a runtime.

**Rejected because**: Rust gives the same speed and embeds into Swift more easily.

## Consequences

**Positive**:
- Fast scanning, one binary, a reusable core.
- The JSON schema becomes the contract for background mode and the GUI.

**Negative / trade-offs**:
- Mac-specific parts (`getattrlistbulk`, `setiopolicy_np`, `libproc`, notifications, LaunchAgent) are written through `libc`/`unsafe`, `osascript` and `launchctl`, not through Foundation. Parsing the `getattrlistbulk` buffer is unsafe code that must be isolated in one module and covered by tests against `lstat`.
- The core is tied to macOS (APFS). For tests and future platforms, a fallback walker on `std::fs` stays behind the same interface.
- The GUI will later need an FFI layer or a process boundary.
- JSON schema versioning policy: new optional fields do not change `schema_version`; renaming or removing a field, changing a type and adding a new enum value (`category`, `class`, `removal`, `notes`, `not_selected`, `skipped[].reason`) bump it, because strict decoders such as Swift `Codable` do not accept unknown values. Rule identifiers are strings: a new rule does not change the version. A reader that sees a different version treats the report as missing (`read_last_scan`).
- The binary is named `deepdrivecleaner` (this is the name shown in Full Disk Access, Activity Monitor and background items); `ddc` is a symlink for the terminal (`make install`, later a Homebrew formula).

**What changes in code / process**:
- The repository gets `Cargo.toml` (workspace), `crates/ddc-core`, `crates/ddc-cli`, `rust-toolchain.toml`, `docs/scan.schema.json`, `bench/`.
- Build and checks: `cargo build --release`, `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt --check`. On the reference machine Homebrew Rust 1.81 comes before rustup in `PATH`, and even `~/.cargo/bin/cargo` picks up its `rustc`/`clippy-driver`. So all calls run with `PATH="$HOME/.cargo/bin:$PATH"`, and the `Makefile` sets an absolute `CARGO` and exports that `PATH`.
- `unsafe` is allowed only in the core's `sys` module (`getattrlistbulk`, `setiopolicy_np`, `libproc`, `sysctl`, `statfs`); the rest of the code is under `unsafe_code = "deny"`. The core does not depend on terminal crates and forbids `print!`/`eprint!` through lints.
- Speed measurements: the median of three runs after one warm-up.

## References

- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- rayon, clap, UniFFI — crate documentation

---

*Last updated: 2026-10-17*
