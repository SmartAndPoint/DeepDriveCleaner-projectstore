---
type: story
id: "story-one-line-install-on-macos-and-linux-ddc-sh-install-sh-and-cargo"
epic: "DDC-007"
title: "One-line install on macOS and Linux: ddc.sh/install.sh and cargo"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [distribution, install, macos, linux, website]
code_refs: ["packaging/install/", ".github/workflows/install.yml", "crates/ddc-cli/Cargo.toml", "scripts/build-cli.sh", "scripts/build-app.sh", "scripts/publish-release.sh", "Makefile", "README.md"]
specs: []
started_at: "2026-10-30T07:37:33.282Z"
closed_at: null
plan_updated_at: "2026-10-30T07:37:33.282Z"
---

# One-line install on macOS and Linux: ddc.sh/install.sh and cargo

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner wants the CLI alone, without the app and without Homebrew, to install with one line on macOS and Linux:

```sh
curl -fsSL https://ddc.sh/install.sh | sh
curl -fsSL https://ddc.sh/install.sh | sh -s -- --prefix "$HOME/bin"   # with flags
```

This is decision 3 of [[research/distribution-through-agents|Distribution through agents]]. On macOS it needs no port: the signed, notarized `aarch64-apple-darwin` tarball already exists, in the layout the script expects. It does need one more release, the first that carries `SHA256SUMS` (decomposition item 1). Releases up to 0.7.0 have nothing to verify against; 0.8.0 is the first that carries `SHA256SUMS`. The Linux half turns on when story "Linux scanner and rule catalog" publishes the Linux archives. Until then the script says plainly that Linux is not released yet.

The script:
- detects the OS and architecture;
- finds the latest release (or `--version <v>`);
- downloads the matching archive and verifies it against the release's `SHA256SUMS`;
- installs `deepdrivecleaner` into a user folder, never with `sudo`;
- creates `ddc` as a symlink to it;
- tells the user how to add the folder to `PATH` if it is missing.

It is non-interactive and fails closed. A run that does not finish installs nothing.

Things the code requires of the script on macOS:

| Fact | Consequence for the script |
|---|---|
| `ddc schedule install` writes the path from `current_exe()` into the LaunchAgent (`ddc-cli/src/cmd/schedule.rs`). | Install to a stable path (default `~/.local/bin/deepdrivecleaner`). Upgrade by downloading next to it and renaming over it, so the LaunchAgent keeps working after an upgrade and a running process is not disturbed. |
| A Developer ID signature keeps a Full Disk Access grant across builds; TCC keys a CLI grant by path. | The same path on every upgrade keeps the grant. The script states that moving the binary loses it. |
| The cask installs the same binary inside the app and links it into Homebrew's `bin`. | If `deepdrivecleaner` or `ddc` already resolves on `PATH` to anything outside `--prefix` (the app, a Homebrew prefix, `~/.cargo/bin` from `cargo install` or `make install`, …), or `/Applications/DeepDriveCleaner.app` exists, refuse (exit 4) and name the existing install with its upgrade or removal command. Two copies would mean two `ddc`s on `PATH` and two schedulers. `--force` installs anyway. |
| `curl` does not set the quarantine attribute. | No Gatekeeper prompt. The script still checks `codesign --verify --strict`, that the signature is a Developer ID Application signature, and that its `TeamIdentifier` equals the project's (a constant in the script, as `publish-release.sh` already pins it; a team id is in every signed binary, so it is not a secret). All three are non-prompting and run before it installs. Without the team pin, a substituted binary plus a substituted `SHA256SUMS` would pass. |
| The cask requires macOS 26; the tarball's minimum is whatever the linker wrote (Rust's default for `aarch64-apple-darwin` is 11.0, which nobody has run). | The floor is a decision, not a default: `build-dist.sh` sets `MACOSX_DEPLOYMENT_TARGET` to the oldest macOS the CLI was run on (26 at first; the owner lowers it only after a run on the older version), the script carries the same constant and compares it with `sw_vers`, and the README states it. An older macOS is refused clearly. |
| Only Apple Silicon is built (epic decision). | `uname -m` = `x86_64` on macOS: check `sysctl -n sysctl.proc_translated` (Rosetta shell on Apple Silicon → install arm64), otherwise refuse with "Intel Macs are not supported". |

Two more install routes need no script:
- **From source:** `cargo install --git https://github.com/SmartAndPoint/DeepDriveCleaner --locked ddc-cli` installs `deepdrivecleaner`. The README gives the one `ln -s` line for `ddc`.
- **cargo-binstall:** `cargo binstall --git https://github.com/SmartAndPoint/DeepDriveCleaner ddc-cli` downloads the release archive instead of compiling. It needs `[package.metadata.binstall]`.

## Decomposition

- [x] `publish-release.sh` attaches `SHA256SUMS` (all archives) to every release, starting with the next macOS release. Today's sidecar `.sha256` files stay local.
- [x] `packaging/install/install.sh`: POSIX `sh` (checked with `shellcheck -s sh` and run under `dash`). Download with `curl` or `wget`; verify with `sha256sum` or `shasum -a 256`.
  - Latest version from the redirect of `https://github.com/SmartAndPoint/DeepDriveCleaner/releases/latest`, with no API token and no rate limit.
  - Flags: `--version <v>`, `--prefix <dir>` (default `~/.local/bin`), `--force`, `--uninstall`, `--modify-path` (off by default; otherwise it only prints the line to add).
  - `DDC_INSTALL_BASE_URL` overrides the download host, for tests only.
- [x] Documented exit codes, in the script header and the CLI guide:
  - 0 installed or already current;
  - 2 unsupported platform;
  - 3 verification failed;
  - 4 existing install found;
  - 5 download failed.

  `--version` of a release without `SHA256SUMS` exits 3 (nothing to verify against) and names the first verifiable version. The CLI guide keeps one table of all exit codes: the install scripts', the CLI's (`exit.rs`: `REFUSED`, `BUSY`) and the platform-not-supported code from story "Platform seam: ADR and capability model".

  Done with one change, decided 2026-10-30: the README keeps two tables, the CLI's and the script's, under the same heading. The codes overlap (2 means a refusal to the CLI and an unsupported platform to the script), so one table would have to say which program each row is for. The script also exits 1 for any other error.
- [x] Upgrade = run again. Same path, atomic rename. It prints old → new version.
- [x] `--uninstall` removes exactly the two files it installed (it checks that `ddc` is its own symlink). If a LaunchAgent of `ddc schedule` exists, it tells the user to run `ddc schedule uninstall` first. It does not run it on their behalf.
- [x] `[package.metadata.binstall]` in `crates/ddc-cli/Cargo.toml`:
  - `pkg-url = "{ repo }/releases/download/v{ version }/deepdrivecleaner-{ version }-{ target }.tar.gz"` (hardcoded: binstall's `{ archive-suffix }` for `tgz` is `.tgz`);
  - `bin-dir = "{ bin }{ binary-ext }"`;
  - `pkg-fmt = "tgz"`, and `zip` for the Windows target;
  - with `--git`, binstall reads the version from `main`. Between a version bump and its release it falls back to compiling. Keep that window short (bump, tag, release in one sitting) and say so in the README. The Windows override uses `.zip` the same way.
- [x] README: three install routes per OS (script, manual download with `shasum -a 256 -c`, cargo), and the `ddc` link line for cargo installs. The manual route notes that a browser download is quarantined and the bare binary is notarized but not stapled, so the first run does an online Gatekeeper check. The cask section says to remove a script install first: the cask cannot refuse, and two `ddc`s would land on `PATH`.
- [x] Provenance of `SHA256SUMS`: checksums prove integrity, not origin. A compromised GitHub account could replace both archive and sums. On macOS the team-pinned `codesign` check covers it. For Linux, decide in this story between checksum only (accepted risk, written down), a minisign signature of `SHA256SUMS` with an owner-held key whose public half is on the site, or `gh attestation verify` when `gh` is present.

  Decided 2026-10-30: checksum only on Linux, an accepted risk written in the script header and the README. The README's manual route names `gh attestation verify <archive> --repo SmartAndPoint/DeepDriveCleaner` for whoever wants to check provenance by hand.
- [ ] Hosting, handed to DDC-004: the website build copies `packaging/install/install.sh` from the tagged release into the site, so `https://ddc.sh/install.sh` serves the exact script of the latest release, with no redirect and no drift. Note on DDC-004's story that its install-command criterion gains this script.
- [x] CI tests (`install.yml`, on macOS and Ubuntu runners), run against a local fake release served by `python3 -m http.server` through `DDC_INSTALL_BASE_URL`:
  - fresh install;
  - upgrade;
  - checksum mismatch;
  - unknown architecture;
  - existing Homebrew-style install;
  - uninstall.
  
  The fake server does not redirect `releases/latest`, so these tests pass `--version <v>`. `releases/latest` also ignores pre-releases: a pre-release is installed only with `--version`.

  On macOS, one more job installs the real latest release.

## Implementation Plan

The macOS half lands now; the Linux half closes after story "Linux scanner and rule catalog" enables the musl archives. `SHA256SUMS` comes from story "Build matrix: every target built, tested and packaged from one tag".

1. `packaging/install/install.sh` (POSIX `sh`, `set -eu`): flags, platform detection (arm64 macOS, Rosetta to arm64, Intel and macOS below 26 refused; Linux x86_64/aarch64 to musl, refused with "not released for Linux yet" while `SHA256SUMS` has no musl line), the version (latest by redirect, or `--version`), the existing-install check (exit 4 naming it; `--force`), download to files, checksum, archive listing, `codesign` against the Developer ID requirement on macOS, `--version` of the new binary, then an atomic replace in `--prefix` and the `ddc` link. Exit codes 0/2/3/4/5 in the header. `--uninstall` stops on the CLI's own LaunchAgent (exit 4). No `sudo`; nothing written outside `--prefix` and a temporary folder except `--modify-path`'s one profile line.
2. Test seam: a loopback base URL plus `DDC_INSTALL_ALLOW_ADHOC=1` together; anything else non-https is refused.
3. `packaging/install/test_install.py` and `stub.c`: a fake release over `http.server`, every case of the acceptance criteria, under `dash` and through `| sh`. `make install-test` joins `make gate`; `make install-lint` runs `shellcheck`.
4. `.github/workflows/install.yml`: fake releases on macOS and Ubuntu on every push; the real latest release through `curl | sh`, `cargo install --git` and `cargo binstall` on a release, weekly and on demand.
5. `MACOSX_DEPLOYMENT_TARGET=26.0` in `scripts/build-cli.sh` and `scripts/build-app.sh`, checked with `vtool`.
6. `ddc-cli`'s `[package.metadata.binstall]`; README's install routes and the exit-code table; release notes name the install line and `shasum -a 256 -c SHA256SUMS --ignore-missing`.

## Acceptance Criteria

- [ ] On a macOS runner without Homebrew's cask, `curl -fsSL <served>/install.sh | sh` installs the latest real release; `deepdrivecleaner --version` and `ddc --version` print it, after the `PATH` line the script prints (CI evidence).
- [ ] Running the script again with a newer fake release upgrades in place: same path, new version, and a `ddc schedule install` agent created before the upgrade still points at a working binary (test).
- [ ] A tampered archive makes the script exit 3 and leaves no file behind (test).
- [ ] With `deepdrivecleaner` or `ddc` resolving into a Homebrew prefix, the app or `~/.cargo/bin`, or with `/Applications/DeepDriveCleaner.app` present, the script exits 4 and names the existing install (test).
- [ ] On Linux before the Linux archives exist, the script exits 2 with "not released for Linux yet"; after, it installs the musl archive (CI, both states; a pre-release needs `--version`).
- [ ] The script never calls `sudo`, never writes outside `--prefix` and a temporary folder, and never edits shell profiles without `--modify-path` (review of the script plus a test that runs it with a read-only `$HOME` except `--prefix`).
- [ ] `cargo install --git … --locked ddc-cli` and `cargo binstall --git … ddc-cli` each give a working `deepdrivecleaner --version` on macOS (CI).
- [ ] Manual (owner): `https://ddc.sh/install.sh` serves the script once DDC-004's site is live.

## Final Summary

## Technical Notes

- `curl | sh` is only as trustworthy as `ddc.sh`'s hosting and TLS. The script verifies the archive. The site also shows the manual route for people who do not pipe into a shell (research note, decision 3).
- Agents run install commands too. Stable exit codes and a non-interactive default matter more than colour.
- Until `ddc.sh` serves the script, the README's one-liner fetches `packaging/install/install.sh` from `main` on GitHub. So `main`'s script must always install the latest published release; the weekly `real-macos` job in `install.yml` checks that, and `real-ubuntu` does for Linux once its archives exist. Once the site is live, `https://ddc.sh/install.sh` is the canonical URL and the README switches to it.
- Hosting handoff (decomposition item 9): DDC-004's story folder is another session's work in progress, not yet in the vault's history, so the note is here instead of on DDC-004's story: its install-command criterion gains `https://ddc.sh/install.sh`, copied from the tagged release's `packaging/install/install.sh`. The owner check in the acceptance criteria covers it.
- `codesign` ships with macOS itself (`/usr/bin/codesign`). If it is missing, the script refuses rather than skipping the check.

## Dependencies

- For the macOS half: one release that carries `SHA256SUMS` (decomposition item 1); nothing else.
- Story "Linux scanner and rule catalog" for the Linux half (archives exist and the CLI is supported).
- Story "Build matrix: every target built, tested and packaged from one tag" for `SHA256SUMS` over all targets.
- DDC-004 story "English and Chinese product website with development runbook" (hosting at `ddc.sh`).

## Attachments

-

---

*Last updated: 2026-10-30*
