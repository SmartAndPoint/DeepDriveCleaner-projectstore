---
type: research
slug: "removal-wherever-a-size-is-shown-coding-agents-protected-folders-and-the-whole-disk"
title: "Removal wherever a size is shown: coding agents, protected folders and the whole disk"
status: final
date: 2026-10-29
authors: ["Evgenii Konev"]
tags: [idea, review, whole-disk, agents, user-data, homebrew, conda, user-report]
review_status: reviewed
reviewed_at: 2026-10-29
---

# Removal wherever a size is shown: coding agents, protected folders and the whole disk

## Question

On 2026-10-29 the owner looked at app 0.4.1 and found three places that show a size but give no way to act on it:

1. **Coding Agents.** This section of the Whole disk view says what coding-agent tools keep in the home folder. It has a Refresh button and nothing else. The owner wants it in the Review window's sidebar, beside node_modules and Python environments, with a way to delete.
2. **"Protected by you"** under User data. It listed 74 folders such as `/private/var/folders/…/T/…` with disabled checkboxes and no button. The owner says a user will not understand what this is, and that it needs a delete button like every other group.
3. **The Whole disk rows** other than the simulator runtimes and Homebrew (Applications, Library for all users, Conda, /usr/local, System files in /private). They say what frees their space but offer nothing to press. The owner asks for an action. Where DeepDriveCleaner cannot act itself, it should at least send the user to Finder, or to the owning program. The owner's example is Homebrew: list what is installed and how big it is with its dependencies, and offer removal, reusing brew itself.

The owner's rule: removal is offered everywhere a size is shown. This note says what each of the three needs, what the safety model allows, and what goes into the backlog.

## TL;DR

- **The 74 temp folders were not the user's data.** The saved report was a scan of `/private`, made from the command line. The folders were the DDC-005 load trees in `$TMPDIR`, which plant `.ddc-keep` markers on purpose. Any Scan Now in the app replaces that report. The popover already says "found in private", but the Review window says nothing and treats the report as the home folder. First story: the Review window and the popover say plainly when the report's roots are not the app's, and Clean and Protect are off when a root lies outside the app's.
- **"Protected by you" gets its own words and a way out, through the CLI.** The safety model never removes a protected folder (safety ADR §2, §5). The way out is **Remove Protection…**, which removes the *protection*, never the folder: a new `ddc keep --remove`, after two ADR amendments, deletes the marker with an identity check, the data-dir lock and a journal line, and an exclusion or a skip is removed from Settings after a confirmation, then Scan Now. The next scan lists the folder in its own group, where it can be cleaned. That needs a report field that names what keeps each finding: today the report says only "kept by a marker" or "excluded".
- **Coding agents in Review need catalog rules, and the owner has to choose where they come from.** DDC-003's ADR "Agent footprint" is on a branch about 58 commits behind main, has no liveness rule for a clean and no rule for `~/.cursor`. The choice: rebase DDC-003 and amend its ADR, or a narrow DDC-002 amendment of the safety ADR now, which DDC-003 takes over later. Either way, the note lists what in the group can be deleted and what cannot.
- **Whole disk rows get the honest action each one has**, from a per-row table: the owning tool in a sheet with a dry run (new for **Conda** and for **Homebrew packages**), **Show in Finder** where a folder can be acted on, the Software Update pane for a downloaded update, and the explanation alone where macOS manages the space. DeepDriveCleaner still removes nothing by path outside the home folder (safety ADR amendment of 2026-10-28), and never hands the user a `sudo` command.
- **Homebrew packages: brew does the removal, ddc does the preview and the sizes.** `brew uninstall` removes unused dependencies by default and has no dry run, and brew's JSON has no sizes. So ddc runs it with `HOMEBREW_NO_AUTOREMOVE=1`, as `brew cleanup` runs today, measures `Cellar/<name>` itself, and computes from brew's JSON which dependencies the removal would orphan. Formulae only in the first version. It needs a safety ADR amendment first.

## Method

- The code at 0.4.1 (commit a1f010f) was read: `agents/mod.rs` (`HARNESSES`), `AgentsSection.swift`, `rules/catalog.rs`, `listing.rs` and `classify.rs`, `report/recheck.rs` (`same_roots`), `disk/home.rs`, `disk/areas.rs`, `disk/model.rs`, `free/{brew,simctl,probe}.rs`, `Formatting.swift` (`Wording.scope`), `PopoverView.swift`, `FoldersSettingsView.swift`, `ReviewItems.swift`, `ReviewView.swift`, `ReviewGroupsView.swift`, `AreaActions.swift`, `AreaActionSheet.swift` and `DiskView.swift`.
- The vault was read: the safety ADR with its amendments; the menu bar ADR §7; the DDC-002 stories "Coding agents in the Whole disk view", "Whole disk: free space with the owning tool", "Whole disk: free the checked areas in one sheet" and "User data: move a folder's picked files to the Trash from that folder"; DDC-003's ADR and stories on their branch; the research note on coding-agent harnesses.
- The saved report behind the screenshot was summarised (its roots, version and finding counts only). No paths from it are recorded here.
- Homebrew's behaviour comes from its manual (`HOMEBREW_NO_AUTOREMOVE`) and its source (`cmd/info.rb`); Conda's from its documentation. Neither was run.

## Findings

### 1. The "Protected by you" folders came from a scan of another root

- The saved report had `scan_roots: ["/private"]`. It held about 88,700 findings, and 74 of them were `user.kept`, all kept by a marker.
- All 74 were under `$TMPDIR/ddc-load-*/…/tree/projects/…`: trees made by `ddc-loadgen`, whose projects profile plants a `.ddc-keep` in about 5 % of the rebuildable projects (`profiles/projects.rs`).
- The app's settings have no extra roots and no exclusions. Someone ran `ddc scan /private` without `--data-dir`, and `scan` always saves to the default data folder. Loadgen's measured runs refuse to run without `--data-dir`, so this was a manual run.
- **What already checks the roots:**
  - the popover: `Wording.scope(roots:home:)` (`Formatting.swift`) shows "found in private";
  - the core: `disk/home.rs` refuses a non-home report for the Whole disk home row, and `report/recheck.rs::same_roots` gates the clean's report edits; `clean --items` refuses items outside the current roots.
- **What does not:** the Review window. It shows any `last-scan.json` as the home folder, and offers Clean and Protect on it. Clean would end in refusals; Protect would write a marker into the foreign tree. "found in private" is also too weak to explain what happened.
- The report does not say where it came from (`ScanReport` has no origin field). A root added in Settings after the last scan gives the same mismatch. So the app can say *what* the report covers, not *who* made it.

### 2. What "Protected by you" is, and what removing a protection needs

- `user.kept` is a synthetic rule: class user-data, method `none`. Three things make a finding kept:
  - a `.ddc-keep` marker in it, at any depth inside its claim, or above it up to the scan root;
  - an exclusion: `--exclude`, or one in Settings → Folders;
  - a skip: since the 2026-10-28 amendment, a skipped folder is added to the excludes, "so a finding that holds a skipped folder is kept whole", and the settings file is the only place to delete one. A skip can also come from `ddc scan --skip` or `ddc clean --skip`; `settings.rs` merges both into one list, so the report must carry the skip's origin.
- The safety ADR (§2, §5, the 2026-10-20 amendment) never removes a kept finding.
- **The report does not say which marker or exclusion keeps a finding.** `classify.rs` records only `Note::KeptByMarker` or `Note::Excluded`. One marker or one exclusion can keep many findings. Under a marker the walker classifies nothing (`rules/visitor.rs` claims the folder as kept the moment it sees the marker), and the rule it displaced is dropped.
- **Undo today:** an exclusion can be removed in Settings → Folders (Remove exclusion). A marker cannot be removed from the app. The app only creates one (`KeepMarker.protect`).
- **Deleting a marker is a deletion.** The menu bar ADR §7 makes the confirmation sheet's Clean the only destructive call the app makes. An app-side `unlink` would have no (dev, ino) check, no journal line and no bound to the scan roots. On the very report behind this note, it would delete loadgen's markers in `$TMPDIR`. So the channel is the CLI: `ddc keep --remove <path>`, which checks the marker folder's identity against the report, refuses outside the current roots, takes the data-dir lock, and writes a journal line (story 5 below).
- **A marker in a git checkout** may be committed by someone else: then it is not "protected by you", and deleting it changes the checkout. The confirmation says so when git tracks the file.
- The rows are locked, so their checkboxes are drawn disabled. The User data section's shared blurb promises "pick files one by one", which is true only for the folders whose entries can be picked (Downloads, Pictures, Movies, Music and device backups).

### 3. What Coding Agents shows, and what a Review group needs

- The section reads `ddc disk agents --json`: each harness's folders and allocated sizes, its live and stale sessions (Claude Code and Codex), and its records and transcripts. The paths come from `agents::HARNESSES`.
- The story "Coding agents in the Whole disk view" (DDC-002, in progress) made the section read-only, and a contract test enforces that it sends no destructive command. Its bytes count in "Rest of the home folder", not as findings. Its Dependencies say of DDC-003's "Harness data rules": "not a dependency in either direction". A Review group reverses that, and the story is amended to say so.
- The Review sidebar is built from findings, one section per risk class, with groups from `GROUPS` in `rules/listing.rs`. A "Coding agents" group needs catalog rules.
- **DDC-003's ADR "Agent footprint" §3** (proposed, on branch `ddc-003-agent-footprint`, last commit 2026-10-18, about 58 commits behind main):
  - scratch (debug logs, shell snapshots, todos): safe, deleted;
  - history, file history, Codex sessions, Codex runtimes, opencode data: review, to the Trash;
  - transcripts: report only, because `--resume` and `/rewind` read them; DDC-003 rejected `safe` for them;
  - memory: never claimed;
  - **no liveness rule** for a clean, and **no rule for `~/.cursor`** (the desktop app's caches are already `ide.cursor-cache`).
- **Where the bytes are** (research note on coding-agent harnesses, of about 11 GB measured): `~/.cursor` about 2.8 GB; transcripts about 2.3 GB of Claude Code's 2.7 GB. As DDC-003 stands, the group would show its largest rows locked and leave Cursor out, which is the complaint the owner made about "Protected by you".
- **The one real risk:** a move to the Trash does not count open files (safety ADR, 0.2.1 amendment). A live session's transcript could be moved while it is written. The liveness checks of `disk agents` exist and can run again at clean time.
- **An option DDC-003 did not weigh:** a transcript of a *stale* session, moved to the Trash after a liveness check at clean time, as a review item. `--resume` of that session then fails until it is put back.

### 4. Whole disk rows and their honest actions

- The areas come from a fixed table (`disk/areas.rs`, 16 rows). Each has a title, an explanation, a "frees" text, and at most one action. Two actions exist: deleting a simulator runtime and `brew cleanup`. Both run through `ddc disk free` with a dry run, a location probe by device and inode, the clean lock, an `area-action` journal line, and measured bytes.
- Disk rows have no context menu and no Show in Finder. The action sheet already has a Copy Command button.
- A structure test (`disk_free_runs_only_its_two_commands`) forbids `sudo` and `autoremove` in `free/`.

| Row | Action | Why |
|---|---|---|
| Applications | Show in Finder | The user removes an app there or with its uninstaller |
| Library for all users | Show in Finder | The apps own it; the user can see what is large |
| macOS downloaded data | — | macOS removes it on its own |
| /opt | Show in Finder | The Conda and Homebrew rows cover the known tools |
| /usr/local | Show in Finder | Installers' uninstallers vary; no generic command |
| System files in /private | — | macOS manages them |
| Other users | — | Each user cleans their own home folder; their homes are not readable |
| Crash dumps | Show in Finder | Finder asks for an administrator's password itself; ddc gives no `sudo rm` line |
| Downloaded macOS update | Open Software Update | Installing the update frees it |
| Simulator runtimes | Delete with simctl (exists) | — |
| Homebrew | brew cleanup (exists); the packages list (story 3) | — |
| Conda | conda (story 2) | — |
| Temporary folders | — | Quitting the owning app or restarting; Finder would show noise |
| /tmp | — | Restarting clears what is older than three days |
| Sleep image | — | macOS manages it |
| Shared folder | Show in Finder | Ordinary files |

- **Terminal.** A `.command` file opened with `NSWorkspace.open` runs in Terminal without the Automation prompt. It runs the moment Terminal opens, though, with no pause to read it, which conflicts with the consent model. So where a command is the action, the sheet shows it with Copy Command; no row needs a Terminal button.

### 5. Conda

- The Conda area covers only `/opt/{miniconda3,anaconda3,miniforge3}`, the all-users installs. The installers' default prefix is in the home folder (`~/miniconda3`, `~/anaconda3`, `~/miniforge3`). `envs_dirs` and `pkgs_dirs` are empty by default, and conda uses the first writable location: `<root_prefix>/envs` and `<root_prefix>/pkgs`, then `~/.conda/{envs,pkgs}`. So a user of a root-owned `/opt` install keeps their environments and packages in `~/.conda`, inside the scan roots. The catalog has no conda rule.
- So conda has two parts:
  - **in the home folder** (a home install, and `~/.conda` for any install): catalog rules with a native method, as the catalog already has for other tools (`native` in `catalog.rs`): an environment (`conda env remove -p <path> --yes`, review class, grouped with Python environments) and the package cache (`conda clean --all --yes`, safe). They get the plan, the re-check, the in-use check of safety ADR §5 (a process whose `CONDA_PREFIX` is the environment) and the journal of every finding;
  - **in /opt**: area actions through `ddc disk free`, with the same commands. Such an action refuses an environment or cache that `conda info --json` places in the home folder; that one belongs to the catalog rule. Its location probe replaces §5's re-checks, so the in-use check is added to it explicitly.
- Either way: a probe that the install's own `bin/conda` matches `conda info --json` (`root_prefix`, `envs_dirs`, `pkgs_dirs`); never `base`; the preview is the environment folder's measured bytes, and `conda clean --dry-run` for the cache.

### 6. Homebrew packages

- `brew info --json=v2 --installed` gives each formula's `installed[]` entries: version, time, `runtime_dependencies`, `installed_on_request` (the old `installed_as_dependency` field is gone; a dependency is a keg with `installed_on_request` false). It has **no sizes**: `cmd/info.rb` computes sizes only for text output.
- `brew leaves --installed-on-request` lists what the user asked for; `brew uses --installed --recursive <name>` what depends on a package.
- `brew uninstall` refuses while installed formulae depend on its target (and casks that depend on it: `installed_dependents.rb`), has no dry run, and **removes unused dependencies by default**: the manual's `HOMEBREW_NO_AUTOREMOVE` says calls to `brew cleanup` and `brew uninstall` will not automatically remove unused formula dependents.
- `brew autoremove -n` lists what is orphaned now, not what one removal would orphan.
- So the story:
  - runs `brew uninstall <formula>` with `HOMEBREW_NO_AUTOREMOVE=1`, as `brew cleanup` runs today;
  - measures `Cellar/<name>` itself (the ADR's "measured, never assumed"). The preview says it is the Cellar only: the formula's data in `var/` and `etc/` stays after `brew uninstall`;
  - names a running `brew services` job for the formula and refuses while it runs, because brew does not stop it;
  - computes the would-be orphans from the JSON, recursively: the formula's runtime dependencies, and theirs, that were installed as dependencies and that no remaining installed formula, and no installed cask's `depends_on formula:`, needs. They are listed in the sheet, and removing them is a second, separately confirmed action;
  - says in the confirmation that brew cannot know about users outside brew (a virtualenv linked against brew's openssl);
  - offers **formulae only** in the first version. A cask uninstall removes an app from `/Applications` (the Applications row), and a cask's `uninstall` stanza can call `sudo` or `launchctl`, which fail on a closed stdin. No `--zap`.

## Comparison

| Option | Pros | Cons |
|--------|------|------|
| Delete a protected folder directly | One click, as asked | Removes what the user asked to keep; contradicts safety ADR §2 and §5 |
| **Remove Protection… through `ddc keep --remove`, then clean it in its own group** | Keeps the ADR; identity check, lock and journal line; the confirmation names the findings the marker keeps | Two steps with a scan between; a new report field, a new command and two ADR amendments |
| Remove Protection by an app-side `unlink` | No CLI change | A destructive call outside the CLI's safety model (menu bar ADR §7); no identity check or journal |
| Coding agents group from the `disk agents` report, no rules | Fast | Not findings: no classes, plan or re-check; a second removal path |
| **Coding agents rules: rebase DDC-003 and amend its ADR** (liveness at clean time, Cursor, stale transcripts) | One design for agents, owned by its epic | A branch about 58 commits behind; waits for that ADR |
| **Coding agents rules: a narrow DDC-002 amendment of the safety ADR** (harness rules from `agents::HARNESSES`, the liveness checks reused; DDC-003 takes them over later) | Answers the owner sooner | Two epics touch the same rules; DDC-003's ADR must absorb them |
| Brew packages through our own package model | Full control | Re-implements brew |
| **Brew packages removed by brew, previewed and measured by ddc** | brew's own dependency refusal; what the owner asked | Orphans computed by us from brew's JSON; an ADR amendment |
| A `.command` file opened in Terminal | No prompt | Runs at once, no pause to read |
| **Copy Command in the sheet** | The user sees and runs the exact command | One paste |

## Conclusion

These stories go to the DDC-002 backlog. Story 4 (Show in Finder, no ADR work) is done second, right after story 1; the rest follow in the order listed.

1. **"Review window: say when the saved scan is not of the home folder"** (p1, a defect).
   - One DDCKit helper, used by `PopoverModel` and `ReviewModel`, computes the roots the way the CLI does (`settings.rs`, `scan/walker.rs`): the Settings roots as the whole list (the home when the list is empty), without the ones that do not exist now (an unplugged disk), each through `realpath(3)` (not Foundation's `resolvingSymlinksInPath`, which strips a leading `/private`), sorted, deduplicated, nested ones dropped. It compares that list with `report.scanRoots`, and the app's canonical home with `report.home`, as `same_roots` does. It can mirror `disk/home.rs`'s cancelled-report check for one notice; an age limit would be new for the Review window, and is a choice.
   - **A root outside the app's roots** (the `/private` case): the Review window and the popover say "This report is of /private, not of your home folder and the folders in Settings. Scan Now replaces it." (or "of 2 folders …", as `Wording.scope` counts them), with the Scan Now button. Clean and Protect are off: Clean would end in refusals.
   - **Roots inside the app's** (`ddc scan ~/Projects`, a documented use): the notice only. Clean works there today (`clean_items.rs` takes a full walk, the plan accepts the items), and stays on.
   - The CLI's `ddc scan <root>` with roots other than the settings', and without `--data-dir` or `DDC_DATA_DIR`, says on stderr that the app will show this report until its next scan.
2. **"Whole disk: Conda environments and its package cache, with conda"** (p2): the home-folder rules and the /opt area actions of Finding 5.
3. **"Whole disk: Homebrew packages, their sizes, and brew uninstall"** (p2), after a safety ADR amendment for "uninstall a formula with its own tool": Finding 6.
4. **"Whole disk: Show in Finder and the honest action for each row"** (p2): the table of Finding 4, plus Show in Finder on the Coding Agents section's folders, which is not destructive and does not wait for story 6. The table already gives Finder to every row with a folder, tool rows included, so it does not wait for 2 and 3.
5. **"Protected by you: say what it is, and Remove Protection"** (p2). It starts after a safety ADR amendment (§2: removing a marker; §7: a new journal `kind`, with the schema version bump its policy asks for) and a menu bar ADR amendment: §7 naming the one caller of `keep --remove`, with a contract test like the Coding Agents section's, and §6 its `--json` as a new contract.
   - The group's own blurb ("Folders you protected with a .ddc-keep file, an exclusion or a skip. DeepDriveCleaner never removes them.").
   - A report field per kept finding: what keeps it (each marker's folder with its device and inode; a Settings exclusion; a command-line `--exclude`; a skip), and the rule that would have claimed the folder, when there is one. The walker must record the displaced rule wherever it claims a folder kept: `claim_unless_kept` and the exclusion branch in `before_read`, the marker check in `after_read` (which must still run the matchers), and the re-tag in `classify.rs`. Nothing beneath a marker was classified, so nothing more can be known before a scan.
   - Each row shows how it is protected.
   - **Remove Protection…** for a marker runs `ddc keep --remove`. It takes the data-dir lock exclusively (a running clean re-checks markers as it acts, `exec/guard.rs`), opens the marker's folder with `O_NOFOLLOW`, compares its device and inode with the report, and unlinks `.ddc-keep` only if it is a regular file: the mirror of `KeepMarker.protect`. A marker of another kind (a symlink or a folder also protects) is refused with its path, and the row says to remove it in Finder. Its journal line has the marker's path, the folder's identity and the findings it kept. The confirmation names the kept findings and the rule that would claim each, says that what lies beneath appears at the next scan, and warns when git tracks the marker (`git ls-files --error-unmatch`). That contract is in DDC-003's proposed ADR, so the story restates the flags it needs (`--no-optional-locks`, `-c core.fsmonitor=false`) rather than waiting for it. After it, the app runs Scan Now, so the folder moves to its own group.
   - For a Settings exclusion, and for a finding that holds a skip, it is the Settings edit (`FolderList.removeExclusion`) after a confirmation, then Scan Now; for a command-line exclusion the row says "Excluded by a command-line flag. The next scan without it lists the folder."
   - Off for a report whose roots lie outside the app's (story 1).
   - The User data section's shared blurb stops promising picking where nothing can be picked.
6. **"Review window: coding agents' data as a group"** (p2), after the owner's choice between the two rule sources above. It lists what will be deletable (scratch; history and sessions to the Trash; Cursor's agent data, not its extensions or worktrees; stale transcripts if chosen) and what will not (live sessions, memory). It amends "Coding agents in the Whole disk view", whose Dependencies change.

Story "Whole disk: free the checked areas in one sheet" (p3) says only runtimes and Homebrew can free anything; stories 2 and 3 amend it. Removal of a user-data folder's own files is the backlog story "User data: move a folder's picked files to the Trash from that folder".

## Open questions for the owner

1. Remove Protection in two steps (protection gone, the app rescans, then clean in its own group), or one flow that also cleans under the underlying rule? The second uses the rule that story 5 adds to the report, and works only where there is one.
2. Coding agents: rebase and amend DDC-003's ADR, or a narrow DDC-002 amendment now?
3. Stale-session transcripts: report only (DDC-003 as written), or to the Trash after a liveness check?

## References

- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]]: §2 (keep markers), §5 (the plan's checks), the 2026-10-20, 2026-10-25 and 2026-10-28 amendments
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §7 (the sheet's Clean is the only destructive call)
- [[research/coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session|Coding-agent harnesses on macOS]]
- DDC-002 stories: "Coding agents in the Whole disk view", "Whole disk: free space with the owning tool", "Whole disk: free the checked areas in one sheet", "User data: move a folder's picked files to the Trash from that folder"
- DDC-003 (branch `ddc-003-agent-footprint`): ADR "Agent footprint: worktrees, harness data and bounded consent for agents", story "Harness data rules"
- Homebrew manual: `HOMEBREW_NO_AUTOREMOVE`; `brew info --json=v2`, `leaves`, `uses`, `uninstall`, `autoremove`; Homebrew source `cmd/info.rb`; Conda: `conda info`, `conda env list`, `conda env remove`, `conda clean`

---

*Last updated: 2026-10-29*
