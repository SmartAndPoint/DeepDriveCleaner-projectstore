---
type: research
slug: "what-fills-system-data-on-a-developers-mac"
title: "What fills System Data on a developer's Mac"
status: final
date: 2026-10-28
authors: ["Evgenii Konev"]
tags: [apfs, system-data, whole-disk, agents]
review_status: reviewed
reviewed_at: 2026-10-28
---

# What fills System Data on a developer's Mac

## Question

Disk Utility and System Settings → Storage show a large "System Data" share that the user cannot open or remove. `ddc` scans only the home folder, so it cannot explain that share either. The owner asked on 2026-10-28 for DeepDriveCleaner to show where the whole disk goes and what of it can be freed.

**What "System Data" means here.** System Settings → General → Storage sorts used space into categories: Applications, Documents, Developer, iCloud Drive, macOS, System Data and others. "System Data" is what is left over. Apple does not publish the rule. In practice it holds caches, logs, temporary files, swap, local snapshots, app support files and anything the categories do not claim. Part of it lies inside the home folder, mainly `~/Library`. The Storage pane's figure was not recorded on the reference machine, so this note does not reconcile against it.

This note is scoped to **what lies outside the home folder**, where ddc does not look today. The home folder side is in [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]].

It answers three questions:
- What is in the used space that a home-folder scan does not see?
- Which of it can a user free, and by which means?
- What breaks when the current `ddc scan` walks those places?

## TL;DR

On the reference machine the APFS container uses 794.6 GB. The home folder holds 499.1 GB of it.

Outside the home folder ddc and `du` agree on about 202 GB of readable files:
- applications, 86 GB;
- Homebrew and miniconda in `/opt`, 30 GB;
- macOS assets, 25 GB in `/System` on the Data volume, nearly all in `/System/Library/AssetsV2`, with iOS simulator runtimes at 16 GB of it;
- `/private`, 29 GB, about four fifths of it temporary folders;
- `/Library`, 18 GB;
- `/usr/local`, 12 GB.

Volumes other than Data add 46.5 GB: swap 23.6, the sealed system 12.6, Preboot 9.0 and Recovery 1.3. Separately, 46.8 GB of the Data volume was not counted by either walk. The two figures are near each other by chance.

Almost all of "System Data" is nameable. A good part of it can be freed, but by the owning tool, not by deleting files:
- `xcrun simctl runtime delete` for simulator runtimes;
- `brew cleanup` for old Homebrew versions;
- a restart for swap;
- installing the downloaded macOS update.

The user's temporary folder held 820,775 entries and about 10 million directories. 96% of the entries were test folders that coding agents' runs never removed. That is evidence for an agent-footprint epic, proposed as DDC-003 and not yet in this vault.

Today's rule set misfires outside the home folder:
- `__pycache__` inside Homebrew and miniconda is selected as `safe`, 16,000 findings;
- the global npm folder `/opt/homebrew/lib/node_modules` is reported as an orphaned `node_modules`.

A whole-disk view must report areas, and must not apply the home folder's rules there.

## Method

Reference machine: Apple silicon, macOS 26, one internal 1 TB SSD. Full Disk Access was on for the app and for the terminal. Measured on 2026-10-28 with read-only commands. No `sudo` was used, and nothing was removed.

**Volumes:**
- `diskutil apfs list`;
- `df -k`;
- `tmutil listlocalsnapshots /`;
- `sysctl vm.swapusage`;
- `xcrun simctl runtime list`.

**Files outside the home folder:**
- `du -skx` for each top-level folder of `/System/Volumes/Data`, except `Users/<you>`, then one level deeper in the large ones;
- `ddc scan --json` 0.3.0 with a scratch `--data-dir` over the same folders: `Applications`, `opt`, `private`, `System`, `Library`, `usr`, `Users/Shared`.

**Home folder:** the last scheduled `ddc scan` of the same day, with Full Disk Access on, 0 unreadable folders.

**Temporary folder:** `ls -f "$TMPDIR"`, grouped by name prefix.

`du -k` counts KiB. All sizes below are decimal GB, so KiB × 1024 / 10⁹. Two tools print binary units and were converted:
- `sysctl vm.swapusage` prints MiB: total 22528 MiB = 23.6 GB, used 21110 MiB = 22.1 GB;
- `xcrun simctl runtime list` prints GiB: 14.8 GiB = 15.9 GB.

## Findings

### 1. The container

| Volume (role) | Used |
|---|---|
| Data | 747.8 GB |
| VM (swap) | 23.6 GB |
| System (sealed snapshot) | 12.6 GB |
| Preboot | 9.0 GB |
| Recovery | 1.3 GB |
| Update and smaller volumes | < 0.1 GB |
| **Container in use** | **794.6 GB of 994.7 GB** (the rows sum to 794.3; the rest is rounding and container metadata) |

Two iOS simulator runtimes are mounted from their own disk images as separate APFS containers (18.2 GB and 18.6 GB). Their images are files in `/System/Library/AssetsV2`, so they count once, on the Data volume. `simctl` reports 14.8 GiB (15.9 GB) for the two images; `du` gives 16.0 GB for their AssetsV2 folders.

The machine had no local Time Machine snapshots. Swap was 22.1 GB used of 23.6 GB (the VM volume's size), and `/private/var/vm/sleepimage` held 2.1 GB.

### 2. The Data volume, by area

| Area | Size | What it is | Who can free it |
|---|---|---|---|
| Home folder | 499.1 GB | what `ddc scan` sees today | the user, through Review |
| `/Applications` | 85.9 GB | app bundles | the user, by removing an app |
| `/opt` | 29.7 GB | Homebrew `Cellar` 15.6 GB, `lib` 3.6 GB; miniconda | `brew cleanup` (old versions, downloads); `conda clean` |
| `/private` | 29.3 GB | temporary folders (`var/folders`, `tmp`), system databases (`var/db`), `sleepimage`; breakdown in §2a | the temporary folders' owners; the rest is the system's |
| `/System` (on Data) | 25.0 GB | `Library/AssetsV2`: simulator runtimes 16.0 GB, Siri 2.7 GB, developer docs 1.7 GB, language data 1.1 GB | runtimes with `xcrun simctl runtime delete`; the rest by macOS |
| `/Library` | 17.7 GB | Application Support 9.6 GB (plug-ins and content of audio and video apps), Command Line Tools 3.0 GB, a downloaded macOS update 2.2 GB, frameworks, Java | the update by installing it; content by its app |
| `/usr/local` | 11.8 GB | software installed outside Homebrew's prefix | its installer |
| `/Users/Shared` | 2.0 GB | shared files | the user |
| **Not counted** | **46.8 GB** | Spotlight index, `.fseventsd`, `.DocumentRevisions-V100`, 315 folders readable only by root (`/private/var/db`, `spool`, `protected`), APFS metadata; and what the home scan does not walk: the cloud folders and the user's skips | macOS, or the user for the home part |

The two counts outside the home folder agree: 201.4 GB by `du`, 201.9 GB by `ddc`. The home folder and outside together make 701.0 GB of the 747.8 GB the volume reports, which leaves 46.8 GB.

"Not counted" holds more than unreadable folders:
- folders only root can read;
- the cloud folders (`~/Library/CloudStorage`, `~/Library/Mobile Documents`), which the home scan does not traverse;
- the user's `skips`;
- local Time Machine snapshots, of which this machine had none;
- APFS metadata.

APFS clones and hard links make a per-folder sum count too much rather than too little. So 46.8 GB is a lower bound of what was not walked.

### 2a. `/private` in detail

`/private` was measured again per folder with `ddc du`, about six hours after the `du` pass. It had grown to 31.4 GB, mostly in the temporary folders:

| Folder | Size | What it is |
|---|---|---|
| `var/folders` | 14.2 GB | per-user temporary and cache folders. Of the user's own: `T` (`$TMPDIR`) ≈ 9.8 GB, `C` 2.5 GB, `X` 1.9 GB; other users' and the system's are near zero |
| `tmp` | 10.5 GB | nearly all of it the per-session scratch folders of one coding agent, 43 project folders; one session's scratch held 9.3 GB |
| `var/db` | 4.3 GB | system databases; partly readable only by root |
| `var/vm` | 2.1 GB | `sleepimage` (swap lives on the VM volume, not here) |
| `var/log` | 0.2 GB | logs |
| the rest of `var` | < 0.1 GB | |

`$TMPDIR` was not walked on its own, because one walk of it takes about ten minutes. Its size is `var/folders` minus the user's `C`, `0` and `X` and the other users' folders.

So about 24.7 of the 31.4 GB are temporary folders, and most of that was written by coding agents.

### 3. The temporary folder is agent debris

`$TMPDIR` (`/private/var/folders/<id>/T`) held 820,775 entries. `ls -f` alone took more than two minutes.

| Prefix | Entries | Source |
|---|---|---|
| `ps*` (`ps-inst`, `ps-home`, `ps-draft`, `ps-hook`, `ps-cli`, …) | 790,104 | test runs of an agent plugin, run by coding agents |
| one other project's test folders | 17,791 | that project's tests |
| `.tmp*` | 505 | Rust `tempfile` |
| `ddckit-*` | 181 | DeepDriveCleaner's own app tests |

Inside these folders lay about 10 million directories. The rule set found 52,779 orphaned `node_modules` among them, all test fixtures.

The same machine also had seven fake-CLI scripts from DeepDriveCleaner's app tests. They had been polling for a gate for up to three days, at about 3% CPU each. The fake now gives up when its folder is gone or after five minutes.

Agents create and abandon temporary state at a rate no person does. It costs in two ways:
- **Space.** `$TMPDIR` held about 9.8 GB, and agents' scratch folders in `/private/tmp` another 10.5 GB (§2a).
- **Entries.** Every walk of the disk pays for each of them in time.

### 4. What the current scan does outside the home folder

`ddc scan` over the folders above took 628 s and walked 10.3 million directories. Two quadratic steps after the walk made it look hung. Both were fixed and committed on 2026-10-28:
- the claim-versus-finding lookup in the totals;
- a project folder's activity, which was read again for each finding.

The walk itself still pays for every directory.

The home folder's rules give wrong answers outside it:
- `python.pycache` selected 16,000 `__pycache__` folders inside `/opt/homebrew` and `/opt/miniconda3` as `safe`, 1.4 GB in all. Homebrew and conda own those trees, and a package's files are not the user's to delete.
- `node.node_modules-orphan` called the global npm folder `/opt/homebrew/lib/node_modules` orphaned. The class is `review`, so it is not selected, but the label is wrong.
- `python.venv` and `node.node_modules` found environments in `/private/tmp`. They are the user's or an agent's work, and the age rules for projects do not apply there.

Unreadable folders and mounts are reported correctly:
- 315 `inaccessible` entries, all owned by root;
- `other-filesystem` skips for the cryptex mounts of the simulator runtimes and for Docker installer images.

## Comparison

| Option | Pros | Cons |
|---|---|---|
| A. Scan `/` with today's rules | one code path | wrong findings outside home (§4), selected by default; 10+ minutes per scan; walks root-only and system-sealed places |
| B. Areas outside home, report only, with tool actions | names every GB; no new delete path; the actions are the owning tools' own | a second kind of row next to findings; the area table is maintained in code |
| C. Leave it to Disk Utility | nothing to build | the owner's problem stays unsolved |

## Conclusion

Option B is recommended.

- **A whole-disk view is a list of areas, not of findings.** It shows the home folder (today's scan), the areas in §2, the other volumes in §1 and what was not counted, each with its size. The sizes add up, to within rounding, to what Disk Utility shows.
- **Outside the home folder nothing is selected and nothing is removed by path.** Space is freed only by an area's owning tool, and only after the user confirms:
  - simulator runtimes, `xcrun simctl runtime delete <id>`;
  - Homebrew, `brew cleanup`;
  - an area with no tool, such as swap or a downloaded update, only gets an explanation of what frees it.
- **Rules from the home folder are not applied outside it.** Areas are measured, and a few named folders inside them can be listed, such as the largest apps with their last use.
- **Walk time is the cost.** A whole-disk walk is far slower than a home scan, mostly because of directories in temporary folders. It runs on demand, not hourly, until a persistent index exists. The index would be a file-backed tree of sizes, kept current from the FSEvents journal instead of a full walk, and needs its own ADR. Directory modification times alone cannot validate it: a file growing inside a folder does not change the folder's time.
- **Agent debris** in temporary folders is evidence for the proposed agent-footprint epic (DDC-003, not yet in this vault): 790,000 abandoned test folders in one `$TMPDIR`.

## References

- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]: the home folder side; its §9 on why `du` is not freed space
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: the scan roots, protected folders, and native tool actions
- `xcrun simctl runtime list|delete`, `diskutil apfs list`, `tmutil listlocalsnapshots /`, `sysctl vm.swapusage`

---

*Last updated: 2026-10-28*
