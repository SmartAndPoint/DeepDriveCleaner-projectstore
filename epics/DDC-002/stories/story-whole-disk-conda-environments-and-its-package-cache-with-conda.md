---
type: story
id: "story-whole-disk-conda-environments-and-its-package-cache-with-conda"
epic: "DDC-002"
title: "Whole disk: Conda environments and its package cache, with conda"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-29
external_refs: {}
tags: [core, app, whole-disk, conda, native-command, adr-amendment, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Whole disk: Conda environments and its package cache, with conda

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner asked on 2026-10-29 for an action on every Whole disk row that shows a size. The Conda row says "conda clean --all removes the package cache; conda env remove removes an environment" and offers nothing to press. The analysis is in the research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 5).

**Where conda keeps its space.**

- The Conda area (`disk/areas.rs`) covers only `/opt/{miniconda3,anaconda3,miniforge3}`, the all-users installs.
- The installers' default prefix is in the home folder: `~/miniconda3`, `~/anaconda3`, `~/miniforge3`.
- `envs_dirs` and `pkgs_dirs` are empty by default. Conda then uses the first writable location: `<root_prefix>/envs` and `<root_prefix>/pkgs`, then `~/.conda/envs` and `~/.conda/pkgs`. So a user of a root-owned `/opt` install keeps their environments and packages in `~/.conda`, inside the scan roots.
- The catalog has no conda rule, so none of this is a finding today.
- Out of scope: environments created with `-p` anywhere else.

**The tool.** A finding is cleaned by the conda that owns it, as safety ADR §6 asks ("only if the tool itself reports that path as its own"). The known installs are the three home prefixes and the three `/opt` prefixes, in that order. The tool for a finding is the first install whose `bin/conda info --json` lists, compared by device and inode, the finding's parent in `envs_dirs` (an environment: `envs_dirs` holds the parent folder such as `~/miniconda3/envs`, not each environment) or the finding itself in `pkgs_dirs` (the cache). With none, the finding has no native method and takes its fallback. Each install's `conda info --json` runs at most once per clean and its answer is reused for every finding (conda takes seconds to start).

This does not fit today's native command. Go's is a static `program: "go"` found on the tool path (`exec/mod.rs`) with one `LocationProbe` (`crates/ddc-core/src/class.rs`: one program, `Equals` or `Contains` on its stdout). Conda needs a new probe kind: several candidate absolute paths, a JSON list compared by device and inode, and the chosen install becoming the command's program.

**Scoping each command to one folder.** `conda clean --all` cleans every package folder it can write, so with a home install's `pkgs` and `~/.conda/pkgs` both present, one command would empty both. Every `conda clean` therefore runs with `CONDA_PKGS_DIRS=<that one folder>` in its environment (conda reads `CONDA_<setting>` over its config). `conda env remove -p <path>` names its one environment. It also writes `~/.conda/environments.txt`, outside the finding.

**Two parts, one tool.**

1. **In the home folder: catalog rules with a native method**, as the catalog already has for Go and others (`native` in `catalog.rs`):
   - `conda.env`: a folder with `conda-meta/history` under an `envs` folder of a known install or under `~/.conda/envs`. Review class. Method: `conda env remove -p <path> --yes`. Fallback: Trash. Group: "Python environments". Never an install's prefix itself (the base environment).
   - `conda.pkgs`: a package cache (`<root_prefix>/pkgs` of a home install, `~/.conda/pkgs`). Safe class. Method: `conda clean --all --yes` with `CONDA_PKGS_DIRS`. Fallback: Delete. Group: "Package downloads". The confirmation says it removes unused packages only: packages linked into environments stay, so less may be freed than the folder's size.
   - Each gets the plan, the re-check, the in-use check of safety ADR §5 (a running process whose `CONDA_PREFIX` is the environment), and the journal of every finding.
2. **In /opt: area actions through `ddc disk free`**, in the sheet that runs `brew cleanup` today (story "Whole disk: free space with the owning tool"):
   - a list of the install's environments (`conda env list --json`) with their measured sizes, and its own package folder. When that folder is not writable for the user, the user's packages live in `~/.conda/pkgs`, a `conda.pkgs` finding in Review; the sheet says so in place of the cache line;
   - removal of one environment at a time, and `conda clean --all --yes` with `CONDA_PKGS_DIRS=<prefix>/pkgs`, each after a dry run: the environment's measured bytes, `conda clean --all --dry-run --json` for the cache;
   - the action refuses an environment or cache in the home folder: that one is the catalog rule's;
   - its location probe replaces §5's re-checks, so the `CONDA_PREFIX` in-use check is added to it explicitly;
   - a root-owned install's command fails with a permission error; the sheet shows the existing "needs an administrator" wording;
   - the exclusive data-dir lock and an `area-action` journal line, as for `brew cleanup`.

**Before it starts (the work-start gate).**

- A safety ADR §6 amendment: `conda env remove` writes `environments.txt` outside the finding, and `conda clean` is scoped by `CONDA_PKGS_DIRS`; the amendment admits both under those terms, and names the tool lookup above.
- The structure test `disk_free_runs_only_its_two_commands` is a list of forbidden substrings in `free/` (`Command::new`, `remove_`, `sudo`, `autoremove` and others). The conda commands need it reworked; names with `remove_` in them stay out of `free/`.
- The story "Whole disk: free the checked areas in one sheet" (p3) assumes only runtimes and Homebrew can free anything. This story amends it: an `/opt` install's cache joins the rows that can be checked.

## Decomposition

- [ ] Safety ADR §6 amendment (critic-reviewed).
- [ ] Core: the tool lookup as a new probe kind (candidate paths, `bin/conda info --json` once per install per clean, by device and inode, the chosen install as the program); `conda.env` and `conda.pkgs` with their native methods, `CONDA_PKGS_DIRS`, fallbacks and the `CONDA_PREFIX` in-use check; goldens and the rules JSON.
- [ ] Core: the /opt area actions in `free/` (list, dry run, remove one environment, clean its cache), refusing home folders; the structure test reworked.
- [ ] App: the Conda row's action sheet with the environment list and the cache line.
- [ ] Vault: amend "Whole disk: free the checked areas in one sheet".

## Implementation Plan

## Acceptance Criteria

- [ ] Test (core): a temp HOME with a fake conda install (a `bin/conda` script that prints a fixed `conda info --json`) and an environment under `envs/` gives a `conda.env` finding in "Python environments"; the base prefix gives none; `~/.conda/pkgs` gives a `conda.pkgs` finding in "Package downloads".
- [ ] Test (core): with two findings of one install, its fake conda answers `info --json` once per clean.
- [ ] Test (core): a finding no known install lists takes its fallback (Trash for an environment, Delete for a cache); an environment that a fake process's `CONDA_PREFIX` names is refused as in use.
- [ ] Test (core): the fake conda receives `CONDA_PKGS_DIRS` set to the one cache folder of the finding (or of the /opt install), and `env remove -p` with the one environment.
- [ ] Test (core): the /opt area action refuses an environment whose path is in the home folder; its dry run reports the measured bytes; it writes an `area-action` journal line; it holds the exclusive lock.
- [ ] Test (core): the reworked structure test passes for `free/` with the conda actions.
- [ ] Manual (owner): on a Mac with conda, an environment shows in Python environments with its size; removing it in Review removes it from `conda env list`.

## Final Summary

## Technical Notes

- Tests never run a real conda: a script in a temp folder stands in for `bin/conda`.

## Dependencies

- Research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 5)
- "Whole disk: free space with the owning tool" (DDC-002): `ddc disk free`, the action sheet, the probe pattern; this story waits for it to close
- Safety ADR "Cleanup safety model: risk classes, plan and confirmation": §5 (in-use check), §6 (native commands), the 2026-10-28 amendment

## Attachments

-

---

*Last updated: 2026-10-29*
