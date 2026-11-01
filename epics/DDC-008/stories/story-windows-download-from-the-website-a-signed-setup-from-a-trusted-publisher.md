---
type: story
id: "story-windows-download-from-the-website-a-signed-setup-from-a-trusted-publisher"
epic: "DDC-008"
title: "Windows download from the website: a signed setup from a trusted publisher"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-02
updated: 2026-11-02
external_refs: {}
tags: [windows, website, download, signing, smartscreen, release]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows download from the website: a signed setup from a trusted publisher

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

A person on Windows opens the product website, presses **Download for Windows**, and runs a setup that Windows can tell comes from a known developer (the owner, 2026-11-02: "download installer from project site and also sign it as trusted developer"). This story owns the website side and the honest description of what a person meets. The signed setup and its release gates are story "Windows installer: one signed setup with the CLI and the tray"; the signing route and the publisher identity are story "Windows signing and distribution route: research and ADR".

**Where the publisher shows on Windows.** A per-user setup raises no administrator prompt, so the familiar "Verified publisher" line of that dialog never appears. The signature shows in:
- the SmartScreen warning, while the file has little reputation: its "More info" names the publisher instead of "Unknown publisher";
- Smart App Control, which lets an app signed by a certificate from a CA in Microsoft's Trusted Root Program run, and checks every executable, so all five files are signed (the setup, its uninstaller, the app, both CLI executables);
- the file's Properties → Digital Signatures, and `Get-AuthenticodeSignature`.

**Reputation.** SmartScreen reputation builds from download volume on the file and its signing identity; Microsoft offers no submission for it on consumer machines (Microsoft Learn, "SmartScreen reputation for Windows app developers", read 2026-11-02). So every release is signed with one consistent identity, and the page says what a first download shows until reputation builds. If Microsoft Defender flags a release (a program that walks the home folder and deletes caches is a typical heuristic hit), the owner submits it at the Microsoft Security Intelligence portal as its developer: an incident step, not a per-release one.

**On the website.** The product site (deepdrivecleaner.app) gets the Windows setup in its install panel, with a platform switch; the CLI site (ddc.sh) gets it on its Windows tab, beside `install.ps1` and `winget install` (DDC-007 story "Windows install: PowerShell script, winget and a signed binary"). Both languages, as DDC-004's pages are.
- **Release facts at runtime, one source for every platform.** `site.js` reads `https://api.github.com/repos/SmartAndPoint/DeepDriveCleaner/releases/latest`, which allows cross-site reads and gives each asset's name, size, download address and SHA-256 digest. It picks the asset ending in `-windows-x86_64-setup.exe` and fills the button, the version, the size and the hash; the same call fills the macOS download and the CLI archives, so no platform shows a hand-pinned version beside a live one. The answer is kept for the session. This replaces the DDC-004 README rule that release facts are edited in `src/content.mjs` per release.
- **Without JavaScript, or when the API fails or the latest release has no Windows setup** (a macOS-only hotfix, a pre-release), the page shows a plain link to the releases page instead of a broken button.
- The default platform follows `navigator.userAgentData?.platform` (the user agent where that is missing), with the existing `?os=` override.
- **What you will see:** the browser's download warning (Edge's SmartScreen, Chrome's "isn't commonly downloaded"), the SmartScreen warning with the publisher under "More info", no administrator prompt, the setup's pages. Screenshots come from a virtual machine with a neutral account, cropped to the dialog, never from the owner's machines: an install-folder page or a bubble shows the account name.
- **Check the file yourself:** `Get-FileHash` against the shown SHA-256, and `Get-AuthenticodeSignature` showing `Valid` and the publisher (it can say `UnknownError` where Microsoft's identity-verification root is missing; the page says so).
- **Tested on** Windows 11, x86_64; ARM64 Windows 11 runs it under x64 emulation. The minimum is taken from the app ADR and the scanner story, not restated here.

## Decomposition

- [ ] `site.js`: the release facts from the releases API for macOS, Windows and the CLI, the no-JavaScript and failure fallback, the platform default; the Windows section, "What you will see" and the verify commands in both languages; the DDC-004 README's release-facts rule amended.
- [ ] Build-time release facts removed: `version`, `checked`, `release` and `cli.*.filename`/`url` leave `src/content.mjs`; the static pages (`pages.mjs`) and `llms.txt`/`llms.zh.txt` (`package.mjs`) carry no version, only the `/releases/latest` link; the JSON-LD drops `softwareVersion` and `downloadUrl`, or `site.js` fills them.
- [ ] An offline website test (Node), run by a CI job of its own, as the `app` job runs for `app/` (a `make gate` recipe may not read a rust-free path): the Windows section and its fallback link exist in both languages, the asset picker chooses the setup from a recorded API answer and falls back when none is listed, and the built output holds no version-like string.
- [ ] Live checks in `scripts/publish-release.sh` after upload, beside the appcast check: the setup's address resolves to the versioned asset, and the API's digest equals its `SHA256SUMS` line.
- [ ] Screenshots from a clean Windows 11 virtual machine.
- [ ] The release runbook "Cutting a release: tag, CI archives, dist and publish": the Defender false-positive step, and the rule that the signing identity stays the same across releases (Microsoft: a consistent identity carries the publisher trust signal).
- [ ] The first signed release with the website live.

## Implementation Plan

## Acceptance Criteria

- [ ] With JavaScript, the Windows button links the latest release's setup and shows its version, size and SHA-256 from the API; without it, or when the latest release has no setup, the page links the releases page (the offline test, and a manual look).
- [ ] After a release, `publish-release.sh` fails when the setup's address does not resolve or its digest differs from `SHA256SUMS`.
- [ ] No release fact on the site is typed by hand: macOS, Windows and the CLI read the same API answer, and the offline test finds no version in the built pages or `llms.txt`.
- [ ] Manual (owner, on a clean Windows 11 virtual machine with Smart App Control **On**, not Evaluation; recorded in the owner-checks runbook, in the same run as the installer story's browser check): the setup downloaded from the website with Edge and with Chrome shows the warnings the page lists, the SmartScreen warning (when shown) names the publisher, Smart App Control lets it run, no administrator prompt appears, and Properties → Digital Signatures shows the identity. The release-job signature gates are the installer story's.

## Dependencies

- Story "Windows signing and distribution route: research and ADR": the route, the publisher identity, and whether reputation survives the route's certificate rotation.
- Story "Windows installer: one signed setup with the CLI and the tray": the signed setup and its release gates.
- DDC-004 story "English and Chinese product website with development runbook": the sites.
- DDC-007 story "Windows install: PowerShell script, winget and a signed binary": the `install.ps1` and winget lines.

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-02*
