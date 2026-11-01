---
type: story
id: "story-windows-signing-and-distribution-route-research-and-adr"
epic: "DDC-008"
title: "Windows signing and distribution route: research and ADR"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-02
external_refs: {}
tags: [windows, signing, distribution, installer, research, adr]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows signing and distribution route: research and ADR

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner wants a person on Windows to download an installer signed by a verified Windows publisher and install it (2026-10-31). What Windows does with a downloaded file depends on its signature and its reputation, not on the installer's format. So this story first finds out what each route gets on Windows 11. Then an ADR picks one.

This story takes over the research note and the ADR from DDC-007's story "Windows install: PowerShell script, winget and a signed binary". That story keeps `install.ps1` and winget.

Routes to compare:

| Route | What is known (2026-10-31, to verify) | To find out |
|---|---|---|
| Microsoft Store, MSIX | Registration is free for individuals and companies. The Store re-signs the package after certification, so no certificate is bought. Installs from the Store raise no SmartScreen prompt | What a packaged full-trust desktop app may do: whether its writes to `AppData` are redirected, whether it can delete files across the home folder, whether a CLI is available through an app execution alias, whether it can register a Task Scheduler task and start at sign-in (`StartupTask`), how fast certification is, and whether a disk cleaner passes the Store policies |
| Directly downloaded setup, signed through Azure Artifact Signing | About $9.99 a month for the basic tier. Individuals only in the USA and Canada; organizations in a wider list of countries. Signs from GitHub Actions without a key file | Whether the owner's identity is eligible, how identity validation works, and how a new publisher's SmartScreen reputation builds |
| Directly downloaded setup, signed with an OV certificate on a cloud HSM or token | Hundreds of dollars a year. Since 2024 EV gives no immediate SmartScreen reputation, so EV buys nothing here | Vendors and prices, and how CI signs with it without the key leaving the HSM |
| Unsigned | Free | The exact SmartScreen and Smart App Control prompts. This is the baseline the others are compared against |

The installer format is settled (amended 2026-11-02): Tauri's per-user NSIS setup, [[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]] §7. MSIX stays in scope only as the Store route's package, beside the setup. The website download and its "trusted publisher" promise are story "Windows download from the website: a signed setup from a trusted publisher", which this story's ADR feeds.

## Decomposition

- [ ] Research note "Installing a signed and an unsigned app on Windows 11", verified on the owner's lab machine or a VM. Run the critic on it. The note covers:
  - what SmartScreen and Smart App Control show for an unsigned setup and for a signed one from a new publisher, downloaded by a browser (Mark of the Web), by `Invoke-WebRequest` (the `install.ps1` route of DDC-007), and by winget;
  - what winget's validation does with an unsigned package;
  - the Store MSIX limits from the table above. They are tested with a minimal packaged app only if the Store survives the identity question and the owner wants a Store listing at all; otherwise they are recorded from documentation and marked unverified;
  - whether SmartScreen and Smart App Control reputation survives each route's certificate renewal or rotation (Artifact Signing's certificates last days), since reputation is tied to the signing identity;
  - the signing routes: eligibility, whether an individual can onboard today (Q&A reports of paused individual onboarding for Artifact Signing), price, identity validation, and signing in GitHub Actions without exposing a key;
  - Microsoft Defender false positives: a new publisher's binary that walks the home folder and deletes caches is a typical heuristic hit. How to submit a release to Microsoft Security Intelligence, and how long that takes;
  - how the chosen route signs a Tauri NSIS setup (`bundle.windows.signCommand`) and the five files it ships; MSIX only for the Store route.
- [ ] ADR "Windows distribution: signing route and installer format", from the note. Run the critic on it. The ADR:
  - names the route, the format, the file names of the release assets, and what a user sees at each prompt;
  - says where signing runs (CI or the owner's machine) and how the key is held;
  - leaves the publisher identity and account details out of the vault.
- [ ] The owner sets up the chosen identity and signing account. The vault records only that it is done.
- [ ] A test executable signed through the chosen route in the way the release will sign it, and verified.

## Implementation Plan

## Acceptance Criteria

- [ ] The research note and the ADR exist and are critic-approved. The ADR names the route, the format and the exact prompts a user sees.
- [ ] A test executable signed through the chosen route reports a valid, timestamped signature from the chosen publisher: `Get-AuthenticodeSignature` gives `Valid`, and `signtool verify /pa /v` succeeds.
- [ ] Manual (owner, lab machine or VM): the signed test executable, downloaded by a browser, shows the prompts the ADR predicts and no others.

## Final Summary

## Technical Notes

- SmartScreen reputation belongs to the file and to the publisher's history. A new build starts with little or none. The note says what the first users of a new release see, and whether reputation carries over between releases signed by the same identity: the sources disagree.
- Smart App Control is on only after a clean Windows 11 install, so its check runs in a VM. The owner's machines are private, and nothing that describes them goes into the note. See runbook "Native runs: the Linux and Windows legs on the owner's machines".
- Sources checked on 2026-10-31:
  - Microsoft Learn, "Code signing options for Windows app developers" and "SmartScreen reputation";
  - Microsoft Learn, Artifact Signing quickstart (eligibility);
  - Windows Developer Blog, 2025-09-10 (free individual registration) and 2026-05-07 (free company registration);
  - the winget manifest schema (`InstallerType: nullsoft`, `Scope: user`).

## Dependencies

- None to start. The installer story waits for the ADR.

## Attachments

-

---

*Last updated: 2026-11-02*
