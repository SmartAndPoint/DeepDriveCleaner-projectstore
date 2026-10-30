---
type: runbook
slug: "cutting-a-release-tag-ci-archives-dist-and-publish"
title: "Cutting a release: tag, CI archives, dist and publish"
status: draft
date: 2026-10-30
authors: ["Evgenii Konev"]
tags: [release, ci, build, tag, ddc-007]
---

# Cutting a release: tag, CI archives, dist and publish

## Purpose

Use this runbook to publish a DeepDriveCleaner release. One release holds:
- the macOS CLI tarball and the app zip, built, signed and notarized on the owner's Mac;
- the Linux and Windows archives that `release.yml` builds on the tag, once their targets are enabled;
- `SHA256SUMS` over all of them.

The release starts with a tag, and the tag comes first. `make dist` and `publish-release.sh` both refuse until origin holds the tag at the commit they build.

The flow comes from story "Build matrix: every target built, tested and packaged from one tag" (epic DDC-007 "Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers"). The macOS signing and the cask are as before.

**Who pushes the tag:** the owner, with `make tag`. Nothing else creates it, and no workflow creates the GitHub release.

**The release is one commit.** The tag, the macOS assets and the published release all name `HEAD` of `origin/main`. Nothing is pushed to `main` between `make tag` and the end of publish: any commit moves `origin/main` off the tag, and publish then refuses.

## Prerequisites

- [ ] An unsandboxed shell. `notarytool` reads the keychain, which the sandbox blocks.
- [ ] A fresh clone of `main`, or the main clone with a clean tree. Untracked files count as dirty.
- [ ] `.env` with the signing identity and the notary profile, loaded into the shell (`set -a && . <code repo>/.env && set +a`).
- [ ] `gh` authenticated, plus `jq` and `python3`, and the Xcode and Homebrew that previous releases used (`build-app.sh`; `ruby -c` and `brew style` in publish).
- [ ] The version is bumped in `Cargo.toml` and in the app's `MARKETING_VERSION`, they are equal, and `docs/releases/<version>.md` exists. All three are pushed to `main`.

## Steps

### 1. Tag

```bash
make tag
```

`scripts/tag-release.sh` refuses unless all of these hold:
- the tree is clean and on `main`;
- `HEAD` equals `origin/main` after a fetch;
- the Gate run for `HEAD` is green (`scripts/ci-green.sh` waits for it if it is still running);
- the Cargo version equals `MARKETING_VERSION`, and the release notes exist;
- `v<version>` exists neither locally nor on origin.

It then creates the annotated tag `v<version>` at `HEAD` and pushes it. If the push fails, it removes the local tag, so a rerun starts clean.

A tag push starts no Gate run (`gate.yml` runs on branch pushes only). It starts `release.yml`.

### 2. Wait for release.yml

`make dist` (step 3) does not depend on this run and can start right away. Publish (step 4) waits for it when targets are enabled.

```bash
gh run list --workflow release.yml --limit 3
gh run watch <id> --exit-status
```

- The `plan` job checks that the tag equals `v<Cargo version>` and reads `RELEASE_TARGETS`.
- While the list is empty (`'[]'`), the `build` job is skipped and the run is green. That is the state until a catalog story enables a target.
- With targets enabled, each `build` job:
  1. waits for the Gate run;
  2. builds;
  3. smoke-tests (`--version` equals the tag, and `scan --json` validates against the schema);
  4. packs a flat archive and checks it for runner paths;
  5. attests its provenance and uploads it as a workflow artifact named after the archive.

### 3. Build the macOS assets

```bash
make dist
```

- `make dist` first runs `scripts/require-tag.sh`: origin must hold `v<version>` at `HEAD`, so the commonest mistake costs seconds, not a whole gate.
- Then `ci-green`, the full `gate` and `app-test` run.
- Then `build-dist.sh` builds, signs and notarizes the CLI tarball, and `build-app.sh --dist` does the same for the app zip.

### 4. Publish

```bash
./scripts/publish-release.sh --dry-run
./scripts/publish-release.sh
```

A pre-release (a version such as `0.8.0-rc.1`, in `Cargo.toml` and `MARKETING_VERSION`) takes the same steps. It is how the build matrix story's stage-3 criteria are exercised.

`publish-release.sh`:
1. Requires the tag on origin at `HEAD`.
2. Reads `RELEASE_TARGETS` from `release.yml`. A missing or malformed line is an error, never an empty list.
3. With targets enabled, it also:
   - requires a green `release.yml` run for the commit;
   - downloads each archive with `gh run download` (a missing one means it refuses);
   - checks each archive's listing;
   - runs `gh attestation verify` with the repository, the signer workflow and the tag commit.
4. Writes `SHA256SUMS`: one line per asset, `<64 hex>  <name>`, sorted by name with `LC_ALL=C`, with no entry for itself. It checks the file with `shasum -a 256 -c`.
5. Runs `gh release create v<version> --verify-tag` with every asset, then compares each uploaded asset's digest.
6. Writes the tap's cask. A pre-release (a version with a suffix such as `0.8.0-rc.1`) is published as a pre-release and stops before the cask and the tap.

The dry run lists every asset and `SHA256SUMS`, and stops before anything is written.

## Verification

- [ ] `gh release view v<version> --repo SmartAndPoint/DeepDriveCleaner` lists the tarball, the zip, every enabled CI archive and `SHA256SUMS`.
- [ ] In an empty folder, `gh release download v<version> --repo SmartAndPoint/DeepDriveCleaner && shasum -a 256 -c SHA256SUMS` reports every asset OK.
- [ ] For each CI archive: `gh attestation verify <archive> --repo SmartAndPoint/DeepDriveCleaner` passes.
- [ ] `brew update && brew info --cask smartandpoint/tap/deepdrivecleaner` shows the new version (not for a pre-release).

## Rollback

The tag must survive once a release names it. Never move or delete the tag of a published release.

**Before deleting a tag, check that it has no release:**

```bash
gh release view v<version> --repo SmartAndPoint/DeepDriveCleaner
```

It must print `release not found`. A draft counts as a release: delete the draft first (below). With `RELEASE_TARGETS: '[]'`, publish does not look at `release.yml`, so a release can exist under a red run.

- **release.yml failed, or the fix needs a commit:** after the check above, fix it on `main`. Then delete the tag on origin (`git push origin :refs/tags/v<version>`) and locally (`git tag -d v<version>`), and run `make tag` again. Bumping the version also works. While targets are enabled, publish refuses a red run.
- **make dist or publish failed for an outside reason** (credentials, network, Apple's notary service): fix it and rerun. The tag stays, as long as nothing was committed.
- **The release exists but publish stopped later** (for example at the tap): first run `gh release view v<version> --repo SmartAndPoint/DeepDriveCleaner --json isDraft`. If it is published, rerun `publish-release.sh`. It resumes at the cask with the release's own assets after checking their digests, and never rebuilds them. Resume has three conditions:
  - `HEAD` descends from the tag;
  - only `packaging/homebrew/` or `scripts/publish-release.sh` changed since the tag;
  - the release holds `SHA256SUMS`, so resume works only for releases published by this flow.
- **A draft release was left behind:** `publish-release.sh` refuses any draft, complete or not. Delete it and keep the tag: `gh release delete v<version> --repo SmartAndPoint/DeepDriveCleaner --yes`. Then rerun.

## Common Issues

| Message | Cause | Fix |
|---|---|---|
| `origin has no tag v<version>: run 'make tag' first` | `make dist` before `make tag` | Run step 1 |
| `tag v<version> on origin points at <sha>, not HEAD` | This clone is behind, or a commit landed after the tag | `git pull` if behind. A checkout of the tag cannot be published (publish needs `HEAD` = `origin/main`). Otherwise bump the version, or, after the no-release check, delete the tag and tag again |
| `tag v<version> exists here: remove it with 'git tag -d …'` | A local tag left from an earlier attempt | `git tag -d v<version>` once origin's tag is gone |
| `tag v<version> exists on origin: bump the version, or …` | Rerunning `make tag` after a failed `release.yml` | Run the no-release check, delete the tag on origin, rerun |
| `release v<version> is a draft: …` | `gh release create` was stopped before it published | Delete the draft, keep the tag, rerun |
| `since v<version>, files outside the cask changed: …` | Resuming after a commit that touched more than the cask | Bump the version |
| `HEAD is not origin/main` | Unpushed or unpulled commits | Push or pull, then rerun |
| `the app's MARKETING_VERSION (…) is not the CLI's version (…)` | Only one of them was bumped | Bump both, push, rerun |
| `release.yml holds no single RELEASE_TARGETS line` | `release.yml` was edited by hand | Restore the single `  RELEASE_TARGETS: '<json>'` line |
| `release v<version> is published but holds no SHA256SUMS` | Resuming a release made before this flow | Fix the cask by hand for that release |

## References

- Story "Build matrix: every target built, tested and packaged from one tag" (epic DDC-007).
- `Makefile`, the comment above `tag:`; `scripts/tag-release.sh`, `scripts/require-tag.sh`, `scripts/build-dist.sh`, `scripts/publish-release.sh`, `.github/workflows/release.yml`.
- Runbook "First Sparkle releases: shipping the updater, then proving it", whose release step now starts with `make tag`.

---

*Last updated: 2026-10-30*
