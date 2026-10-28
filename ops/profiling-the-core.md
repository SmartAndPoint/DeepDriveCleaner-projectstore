---
type: runbook
slug: "profiling-the-core"
title: "Profiling the core"
status: draft
date: 2026-10-28
authors: ["Evgenii Konev"]
tags: [performance, memory, cpu, leaks, dhat, load, ddc-005]
---

# Profiling the core

## Purpose

Use this runbook when the load harness reports something:
- `make load` fails over a memory or CPU limit;
- `bench/compare_load.py` flags a metric more than 15% worse;
- `make leaks` fails.

Use it also before optimizing anything in `ddc-core`.

It covers:
- telling a regression from a stale baseline;
- a CPU flamegraph;
- a heap profile;
- the leak check and its exclusions;
- re-baselining the limits.

The harness and its rules are in story "Memory and CPU under load: peak memory, leaks and CPU time" (epic DDC-005 "Production-grade Rust core: mandatory gate and load testing"). Story "Complexity guard: operation counts and scaling exponents" adds a section on its operation counters later.

All commands run from the repository root.

## Prerequisites

- [ ] The toolchain from `rust-toolchain.toml`, with `make tools` run once per clone.
- [ ] Xcode command line tools (`/usr/bin/leaks`, `hdiutil`, `plutil`) and `python3`.
- [ ] For step 3: `cargo install --locked samply`, or Xcode.app for `xcrun xctrace` (the command line tools alone do not have it).
- [ ] About 2 GB free for an M disk image (L needs about 15 GB).
- [ ] For CPU numbers: a quiet machine, plugged in, Low Power Mode off (`pmset -g | grep lowpowermode` shows 0). The 1-minute load average (`sysctl -n vm.loadavg`, second field) must be at most a quarter of the CPU count (`sysctl -n hw.ncpu`). The load check does not see a lowered clock: on battery or when hot, CPU seconds rise. Memory numbers depend on none of this.
- [ ] No other load run in progress on this machine. A run in another worktree keeps its own image, but it skews CPU.
- [ ] Never `sudo`. Nothing here needs a privacy (TCC) grant, except Instruments in step 3, which may ask for Developer Tools access the first time. That dialog is the owner's to answer.

## Steps

### 1. Read what moved

```bash
make load                      # M, every profile, enforce
cat target/load/M-<profile>.json   # "failures": the over-limit lines, "cpu_judged"
head -1 target/load/load-M.tsv     # the "#" line: model, ncpu, load_start, mode
```

Each over-limit line names the command, the profile, the median and the limit:

```
scan on wide-root: peak memory … bytes over the limit …
```

`compare_load.py` prints a `WORSE:` line per metric, with the old and new medians.

`clean-items` runs once per profile, so it is judged on memory alone. No CPU failure for it means nothing.

### 2. Regression or stale baseline?

Check these before profiling. Each one makes the old numbers stale without any regression in the code. The baseline commit is on the `#` line of `bench/load-limits.tsv`. If it ends in `-dirty`, use `git log -1 --format=%h -- bench/load-limits.tsv` instead.

- **The tree changed.** `git diff <baseline-commit> --stat -- crates/ddc-loadgen/src`: a change to the generator or a profile gives a different tree. To compare the trees themselves, take `"hash"` from `target/load/<size>-<profile>.json` and, in a worktree at the baseline commit, `cargo run -q -p ddc-loadgen -- manifest <profile> --entries 100000 --seed 1 | plutil -extract hash raw -o - -`. These caps match an APFS disk image.
- **The rule catalog changed.** More rules matching means more work per entry. `git log <baseline-commit>.. -- crates/ddc-core/src/rules/ crates/ddc-core/src/report/project.rs` (the project signals are the largest allocation traffic).
- **The report changed.** A new field per finding grows `last-scan.json`. Its buffer is the global heap peak (`store.rs`, see the story's heap profile), so peak memory rises for a legitimate reason.
- **The toolchain or a dependency changed.** `git log <baseline-commit>.. -- rust-toolchain.toml Cargo.lock`. New codegen moves both CPU and memory.
- **macOS changed.** An update changes libmalloc and APFS, which moves RSS. The `#` line does not record the OS version: compare `sw_vers` with the date of the baseline.
- **The machine changed.** Compare `model=` and `ncpu=` on the `#` lines. The limits come from the reference machine.
- **The machine was busy or slowed down.** `mode=enforce-memory` means CPU was recorded and not judged, and nothing was published to `bench/`. Battery, Low Power Mode or heat raise CPU seconds without raising the load (Prerequisites).
- **The seed changed.** Only seed 1 is published and compared.
- **The baseline is provisional.** The first `bench/load-limits.tsv` (commit `56bef23`) has CPU limits from a busy machine; its first `#` line says so. Its memory limits stand.
- **`wide-root` CPU after story "Complexity guard: operation counts and scaling exponents".** That story removes the `dir_has_any` re-read (an 18 s scan at M), so its CPU limit is re-baselined.

If one of these explains the change, re-baseline (step 7). Otherwise it is a regression: profile it (steps 3 and 4) and fix the code.

### 3. CPU flamegraph

Build the CLI in the `profiling` profile, which is release plus line tables. It goes into `target/profiling`, so `target/release/deepdrivecleaner` is never replaced. The generator is the release build from `make load`.

```bash
cargo build --profile profiling -p ddc-cli --locked
cargo build --release -p ddc-loadgen --locked
dir=$(target/release/ddc-loadgen mktemp | plutil -extract path raw -o - -)
target/release/ddc-loadgen build deep --entries 10000 --into "$dir" --brief >target/profiling/tree.json
home=$(plutil -extract home raw -o - target/profiling/tree.json)
data=$(plutil -extract data raw -o - target/profiling/tree.json)
tree=$(plutil -extract tree raw -o - target/profiling/tree.json)
```

An S tree in a marked temporary folder is the largest the user's volume should hold. At S the slowest profiles are `deep` and `deep-path-max`; for a longer recording, raise `--rate`.

Record with [samply](https://github.com/mstange/samply). samply runs inside `env -i`, so the CLI gets the same environment as in a harness run. samply needs its absolute path there, because `env` looks it up in the new `PATH`. `-o` keeps the recording in `target/profiling`; without it samply writes into `$HOME`, which is the tree's home and is removed with it.

```bash
env -i HOME="$home" PATH=/usr/bin:/bin NO_COLOR=1 "$HOME/.cargo/bin/samply" record --rate 4000 \
  --save-only -o target/profiling/scan.json.gz \
  target/profiling/deepdrivecleaner scan --json --data-dir "$data" "$tree" >/dev/null
samply load target/profiling/scan.json.gz    # opens the Firefox Profiler locally
```

samply launches the command itself, so `samply setup` (which signs samply to attach to a running process) is not needed. No dialog is expected. If one appears, stop and record it in the story.

Instruments' Time Profiler works on the same build. `--env` sets the same environment for the launched CLI:

```bash
xcrun xctrace record --template 'Time Profiler' --output target/profiling/scan.trace \
  --env HOME="$home" --env PATH=/usr/bin:/bin --env NO_COLOR=1 \
  --launch -- target/profiling/deepdrivecleaner scan --json --data-dir "$data" "$tree"
open target/profiling/scan.trace
```

`HOME` is the tree's own home and `--data-dir` is the tree's data folder, as in every harness run. So the owner's settings, `last-scan.json` and home folders are never read or touched.

Remove the folder with `make loadgen-clean` afterwards. It removes every marked folder of this user, including one that a run in another worktree is using, so run it only when no other run is in progress.

**Limit:** this records at S only. The disk image of `make load` is detached when its script exits, so a cost that appears only at M (a superlinear one, like `wide-root`'s `dir_has_any`) cannot be recorded with this runbook yet. Story "Complexity guard: operation counts and scaling exponents" finds such costs with its operation counters.

### 4. Heap profile (dhat)

```bash
make heap-profile                                  # mixed at M
make heap-profile HEAP_PROFILE=names HEAP_SIZE=L   # another profile, or L
```

What it does:
- builds the CLI with `ddc-core/dhat-heap` into `target/dhat`, with the `profiling` profile;
- builds the tree on a disposable disk image;
- runs one isolated `scan --json`;
- keeps `target/dhat/dhat-<size>-<profile>.json`.

`bench/dhat_top.py` then prints two tables of the top 10 sites, each also per entry:
- by bytes live at the global peak (peak RSS follows it roughly);
- by total bytes allocated (allocator traffic, which is CPU time).

dhat counts the heap alone. For `mixed` at M, 7.2 MB is live at the heap peak, against about 24 MB peak RSS for the same scan in `make load`. The rest is thread stacks, the binary and allocator slack. A dhat build's own RSS is larger again, from dhat's bookkeeping, so never compare it with the limits.

How sites are charged:
- A site is the innermost frame in `crates/ddc-core` or `crates/ddc-cli`.
- A bare `ddc_core::…` name without a line is code inlined where dhat could not find a line. Set `debug = true` in `[profile.profiling]` locally for more lines.
- For the whole stacks, open the JSON in the [DHAT viewer](https://nnethercote.github.io/dh_view/dh_view.html).

The reference top 10 (`mixed` at M) is in the story.

`dhat-heap` installs a global allocator, and `op-stats` will install one too (story "Complexity guard: operation counts and scaling exponents"). A build with both fails with a `compile_error!`.

### 5. Leak check

```bash
make leaks
```

How it works:
- For every profile at S, in a marked temporary folder, it runs `leaks -quiet -atExit -- target/release/deepdrivecleaner scan --json --data-dir <data> <tree>`.
- `-atExit` sets `MallocStackLogging=lite`, so a leak report carries its stacks.
- `/usr/bin/leaks` holds `com.apple.system-task-ports.read`, so the ad-hoc-signed cargo build needs no `sudo` and raises no dialog. If a Developer Tools Access dialog appears anyway, stop and record it in the story: something changed.

A profile passes on exit 0, a `0 leaks for 0 total leaked bytes` line and a `last-scan.json` (the scan finished its walk). The outputs are `target/load/leaks-<profile>.txt` and `leaks-<profile>.stderr.txt`. They mean:

- **Exit 1 with a summary of `n leaks`:** a leak. The report's stacks follow the summary line in `leaks-<profile>.txt`.
- **An exit above 1, `sig<N>`, or no summary line:** `leaks` or the scan failed, which is not a leak report. Read `leaks-<profile>.stderr.txt`.
- **`scanned: false`:** the scan stopped before its walk ended, so "no leaks" proves nothing.

`leaks -atExit` passes the environment on to the command: without `HOME` the CLI exits with "HOME is not set", so a passing profile shows it did.

The first `make leaks` on the reference machine is the owner's. `make leaks` joins `make gate` after it. Record in the story whether excluded blocks still count in the summary line. If they do, an exclusion (step 6) can never make a profile pass, and `leaks::passed()` must read the report body instead of the summary.

### 6. Exclusions for `leaks`

An OS update can make `leaks` report a block allocated by a system library: the stack holds only system frames (`libsystem_*`, `libobjc`, `Foundation`, `CoreFoundation`) and no frame of ours. Add it to `EXCLUSIONS` in `crates/ddc-loadgen/src/leaks.rs`:

```rust
pub const EXCLUSIONS: &[(&str, &str)] = &[
    ("_some_system_symbol", "macOS 26.2: <n> bytes in <library>, stack <one-line summary>; not ours"),
];
```

- `-exclude` drops every block whose stack includes the symbol. So choose the system frame nearest the root that never appears under a frame of ours, such as a library initializer. Never choose an allocator or constructor symbol (`malloc_zone_malloc`, `_CFRuntimeCreateInstance`, an NS or CF constructor): it would hide our own leaks too.
- Every entry carries its written reason, with the macOS version and the stack summary.
- Never exclude a symbol of ours or of a crate we depend on: that is a leak to fix.
- Review the list after each major macOS release and remove what no longer reproduces.

### 7. Re-baseline the limits

Do this only after step 2 has shown the baseline is stale, never to make a regression pass.

```bash
sysctl -n vm.loadavg hw.ncpu      # load at most ncpu/4; quit other heavy apps
make load-baseline                # S and M, every profile, 3 runs, record mode
git diff bench/load-limits.tsv
```

What it does:
- It writes `bench/load-limits.tsv`.
- Each limit is base + k per entry: the base from S, k from the slope between S and M. Both get 25% headroom.
- `check` adds 1 MiB and 20 ms of slack.
- The `#` line records the date, commit, model and both start loads. The file is regenerated whole, so a hand-written line such as the "provisional" one goes away by itself. Add one back only if this machine was not the quiet reference machine.

Explain every limit that rises in the commit message. Then run `make load` once more. On a quiet machine it writes `bench/load-<date>-M.tsv` (`mode=enforce`) and compares it with the newest earlier one. Commit both files.

That `make load` can exit 1 although every limit passed: when the stale baseline came from slower code, `compare_load.py` flags each metric more than 15% worse than the previous table. The table is already written. Read the `WORSE:` lines and explain them in the commit, as for the limits.

## Verification

- [ ] `make load` on the quiet reference machine exits 0, and the `#` line of `target/load/load-M.tsv` says `mode=enforce`.
- [ ] `bench/compare_load.py` prints `no metric more than 15% worse`, or every flag is explained in the commit.
- [ ] `make leaks` exits 0, with `"passed": true` for every profile.
- [ ] After a heap fix: the site's per-entry bytes fell in `make heap-profile`, and peak RSS fell in `make load`.

## Rollback

- **Limits:** `git checkout <commit> -- bench/load-limits.tsv` restores the previous baseline.
- **A published table that should not be there:** `git rm bench/load-<date>-<size>.tsv`. `compare_load.py` then uses the newest earlier one.
- **Disk images:** each run discards its own image. One left behind by an interrupted run is discarded by the next `make load`, `make load-image` or `make heap-profile`, once the process that made it has ended.
- **Marked temporary folders:** step 3 leaves one; `make leaks` leaves one only when it is killed. `make loadgen-clean` removes them (see step 3 for when it is safe). It builds the generator in the dev profile, so the first run is slow.
- **Profiling output:** `target/profiling` (with `tree.json`, `scan.json.gz`, `scan.trace`) and `target/dhat` are build output; delete them to reclaim space. Nothing in step 3 writes to the repository root, so `make dist`'s clean-tree check is not affected.

## Common Issues

### Every profile fails with `limits: bench/load-limits.tsv: No such file`
`make load` runs from the repository root. A run from elsewhere needs `--limits`.

### `scan-background on <profile>: no limit`
A command or profile was added after the baseline. Re-baseline (step 7).

### `make load-baseline` fails with `only in the large run` or `only in the small run`
A command ran for a profile at one size only. `clean-items` runs only when the scan lists items, so a profile with items at M and none at S has no S row. Profiles are deterministic, so this points at a generator change: check it before anything else.

### Nothing was published to `bench/`
Only a full run (`LOAD_PROFILE` unset) of seed 1 with CPU judged is published. CPU is judged only with `LOAD_REPEAT` 3 or more and a quiet start, so a busy machine writes `mode=enforce-memory`. Memory was still judged.

### `bench/load-<date>-M.tsv: File exists`
A published table is never replaced, so a second full enforce run on the same day fails after it has passed. The run's table stays in `target/load/load-M.tsv`. Keep the first table, or `git rm` it (Rollback) and run again.

### `no dhat-heap.json … is it built with ddc-core/dhat-heap?`
The CLI passed to `ddc-loadgen heap` is the plain release build. Use `make heap-profile`, which builds the right one.

### `hdiutil: Resource busy`
- `hdiutil create` and `hdiutil attach` are retried three times, 5 s apart.
- The detach at the end is not retried. When it fails, the script prints `could not detach …; … is kept for the next run`.
- Find what holds the volume with `hdiutil info` and `lsof +D <mount point>`.
- The next `make load`, `make load-image` or `make heap-profile` discards the kept image.

## References

- Story "Memory and CPU under load: peak memory, leaks and CPU time" (epic DDC-005 "Production-grade Rust core: mandatory gate and load testing").
- Story "Complexity guard: operation counts and scaling exponents" (same epic): scaling exponents and the counters section of this runbook.
- Code:
  - `crates/ddc-loadgen/src/{run,limits,leaks,heap,measure,isolate}.rs`
  - `crates/ddc-core/src/heap.rs`
  - `scripts/load-image.sh`
  - `bench/compare_load.py`
  - `bench/dhat_top.py`
- Makefile targets: `load`, `load-baseline`, `leaks`, `heap-profile`, `asan`, `bench-test`.
- `man leaks`, `man xctrace`, `man env`; [samply](https://github.com/mstange/samply).

---

*Last updated: 2026-10-29*
