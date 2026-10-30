---
type: story
id: "story-build-matrix-every-target-built-tested-and-packaged-from-one-tag"
epic: "DDC-007"
title: "Build matrix: every target built, tested and packaged from one tag"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [ci, release, build, linux, windows]
code_refs: []
specs: []
started_at: "2026-10-30T07:10:03.214Z"
closed_at: null
plan_updated_at: "2026-10-30T07:10:03.214Z"
---

# Build matrix: every target built, tested and packaged from one tag

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This is the epic's backbone, the build. Today one script builds one target (`aarch64-apple-darwin`) on the owner's Mac. The story grows in three stages, so the build exists from the start and does not wait for the port:

1. **Thin slice (after DDC-005's `gate.yml` exists).** If this epic starts first, this story creates a minimal `gate.yml` that DDC-005 then extends.
   - DDC-005's gate workflow on macOS.
   - Jobs that run `cargo check --workspace --target x86_64-unknown-linux-gnu` and `x86_64-pc-windows-msvc`, marked `continue-on-error`.
   - One `cargo build --release` on macOS, so release-only breakage (LTO, codegen units) is caught before `make dist`.
2. **Native test matrix (after story "Move OS calls behind the platform module").** The check jobs become required. The gate runs natively on `ubuntu-24.04` and `windows-latest`: fmt once, clippy and tests per OS. On an unported platform only the platform-neutral tests run, because the CLI refuses scans there (seam ADR).
3. **Release archives (per platform, after its catalog story).** On a release tag, `release.yml` builds the Linux and Windows archives on native runners. It requires the `gate.yml` run for the same commit to be green, so every packaged binary passed the test suite on its own OS. It smoke-tests each built binary, attests its provenance, and uploads the archives as workflow artifacts. `publish-release.sh` collects them with the owner's locally signed macOS archive and the app zip into one release.

Targets:

| Target | Built on | Archive | Trust |
|---|---|---|---|
| `aarch64-apple-darwin` | owner's Mac (as today) | `deepdrivecleaner-<v>-aarch64-apple-darwin.tar.gz` | Developer ID signature, notarized |
| `x86_64-unknown-linux-musl` | `ubuntu-24.04` | `deepdrivecleaner-<v>-x86_64-unknown-linux-musl.tar.gz` (static) | provenance attestation |
| `aarch64-unknown-linux-musl` | `ubuntu-24.04-arm` | `deepdrivecleaner-<v>-aarch64-unknown-linux-musl.tar.gz` (static) | provenance attestation |
| `x86_64-pc-windows-msvc` | `windows-latest` | `deepdrivecleaner-<v>-x86_64-pc-windows-msvc.zip` | provenance attestation; signing per story "Windows install: PowerShell script, winget and a signed binary" |

`SHA256SUMS` covers every archive in the release. A locally built macOS archive cannot carry a CI attestation; its signature and notarization are its proof.

### Release flow

Today `publish-release.sh` creates the tag itself (`gh release create --target <built commit>`). It refuses a tag that already exists on origin without a release (`STRAY_TAG`), and it treats an existing release as "resume". A workflow that creates the release itself would collide with both.

The flow becomes:
1. The owner (or `make tag`) pushes an annotated `v<version>` tag at `HEAD` of `origin/main`.
2. `release.yml` runs on the tag. It builds, smoke-tests and attests, and uploads the archives as **workflow artifacts**. It never creates the GitHub release.
3. `make dist` builds and notarizes the macOS archive and the app zip from the same commit. `build-dist.sh` checks that `git ls-remote origin refs/tags/v$VERSION` points at `HEAD`. It asks origin, as `publish-release.sh` already does, because local tags are not fetched in this clone.
4. `publish-release.sh` checks:
   - the tag exists on origin and points at `HEAD`;
   - the `release.yml` run for that commit succeeded, when the `enabled` list is non-empty; otherwise only the tag check applies.
   
   It fetches the archives with `gh run download`, writes `SHA256SUMS` over everything, and creates the release as today. The `STRAY_TAG` refusal becomes the checks above.
5. If `release.yml` fails, the tag exists without a release, which is the state `STRAY_TAG` used to prevent. Recovery: fix on `main`, delete the tag on origin (it has no release yet), and tag again, or bump the version. While targets are enabled, `publish-release.sh` refuses while the run is red.

## Decomposition

- [ ] Stage 1: `gate.yml` gains the `cargo check` jobs for Linux and Windows (`continue-on-error`) and a macOS `cargo build --release` job.
- [ ] Stage 2: make the Linux and Windows jobs required, and run the gate natively on both. Test harness portability is done in story "Move OS calls behind the platform module".
- [ ] `release.yml` on `v*` tags, created in stage 1 with an empty `enabled` list, so the catalog stories only add to it. A `plan` job emits the matrix and the build job carries an `if:` on a non-empty list, because an empty matrix fails the run while a skipped job does not. One job per enabled target:
  - waits for, or requires, a green `gate.yml` run for the same commit;
  - `cargo build --release --locked --target <t>` with `--remap-path-prefix` as in `build-dist.sh`;
  - smoke test: `--version` equals the tag, and `scan --json` on a small generated tree validates against the schema;
  - package with `LICENSE.md`, `DISCLAIMER.md` and `README.md`, flat as the macOS tarball;
  - check that no runner path or home folder appears in the binary;
  - `actions/attest-build-provenance`;
  - upload as a workflow artifact.
- [ ] A per-target `enabled` list in `release.yml`: Linux and Windows are added by their catalog stories, in the same commit that flips the platform's `SUPPORTED` switch.
- [ ] `make tag`: refuses unless the tree is clean, `HEAD` equals `origin/main`, and the version in `Cargo.toml` matches. Then creates and pushes the annotated tag.
- [ ] `build-dist.sh`: require the tag on origin at `HEAD` (`git ls-remote`).
- [ ] `publish-release.sh`: the checks and download above, `SHA256SUMS` over all assets, and refusal when an enabled target's artifact is missing.
- [ ] The release runbook (or the Makefile header): tag, wait for `release.yml`, `make dist`, publish, and the recovery from a failed `release.yml`. Who pushes the tag is stated.

## Implementation Plan

Stage 1 lands now; stages 2 and 3 follow the stories they depend on.

1. `scripts/build-cli.sh <target>`: the release build with `--remap-path-prefix`, then the unsigned checks (architecture, `--version`, no home, checkout or `CARGO_HOME` path, the `nm` and `DDC_OP_STATS` checks). `build-dist.sh`, the gate's release build and `release.yml` all call it.
2. `gate.yml`: push trigger on branches only, so a tag push does not start a second Gate run. A `check` matrix (`x86_64-unknown-linux-gnu` on `ubuntu-24.04`, `x86_64-pc-windows-msvc` on `windows-latest`, native `cargo check --workspace --locked`, `continue-on-error`). A `release-build` job on `macos-26` running `build-cli.sh aarch64-apple-darwin`.
3. `release.yml` on `v*` tags with `RELEASE_TARGETS: '[]'`. A `plan` job checks the tag against the Cargo version and emits the matrix and `any`; the `build` job runs only when `any` is true: wait for the Gate run, build, smoke test (`--version`, `scan --json` against the schema), flat archive, runner-path check, provenance attestation, upload as a workflow artifact.
4. `ci-green.sh` takes an optional workflow file (default `gate.yml`).
5. `scripts/tag-release.sh` and `make tag`: clean tree, on `main`, `HEAD` equals `origin/main`, Cargo version equals the app's `MARKETING_VERSION`, release notes exist, tag absent locally and on origin; then an annotated tag, pushed.
6. `build-dist.sh` requires the peeled tag on origin at `HEAD`.
7. `publish-release.sh`: "tag on origin at `HEAD`" replaces `STRAY_TAG`; reads `RELEASE_TARGETS` fail-closed; with targets enabled, requires a green `release.yml` run, downloads each archive (missing means refuse), checks its listing and attestation. `SHA256SUMS` (`<hex>  <name>`, sorted by name with `LC_ALL=C`, no entry for itself) over every asset; `gh release create --verify-tag`; a digest check per asset. A version with `-` is a pre-release (no cask, no tap).
8. The Makefile and README give the order: `make tag` → `release.yml` → `make dist` → publish. The vault runbook "Cutting a release: tag, CI archives, dist and publish" gives the steps and the recovery.

## Acceptance Criteria

- [ ] Stage 1: on `main`, the macOS gate job is green; the Linux and Windows check jobs run (red is allowed until stage 2); the macOS release build job is green.
- [ ] Stage 2: every push runs the gate on macOS, Ubuntu and Windows, and all three are required and green on `main`.
- [ ] Stage 3: a pre-release test tag produces the enabled targets' archives, each smoke-tested on its runner; `gh attestation verify <archive> --repo SmartAndPoint/DeepDriveCleaner` passes for each CI-built archive.
- [ ] `publish-release.sh` on that tag produces one release with the CI archives, the macOS archive, the app zip and `SHA256SUMS`, and refuses when an enabled target's artifact is missing or the run failed (tested on a throwaway pre-release).
- [ ] No archive contains a runner path or a home folder (checked in the workflow).

## Final Summary

## Technical Notes

- Native runners rather than cross-compiling (`cargo-zigbuild`, `cargo-xwin`) let each binary run its tests on its own OS. Cross-compiling stays a fallback for building without CI.
- `ubuntu-24.04-arm` is generally available for public repositories (GitHub changelog, 2025-08-07). Re-check runner labels and terms when implementing.
- musl's allocator is slow under multi-threaded allocation. If Linux load checks miss by more than 2×, consider `mimalloc` for musl targets (story "Linux scanner and rule catalog").
- Signing stays with the owner. The workflow holds no Apple credentials.

## Dependencies

- DDC-005 story "Mandatory gate before every commit" (the workflow it extends).
- Story "Move OS calls behind the platform module" (stage 2).
- Stories "Linux scanner and rule catalog" and "Windows scanner and rule catalog" (stage 3 per platform).

## Attachments

-

---

*Last updated: 2026-10-30*
