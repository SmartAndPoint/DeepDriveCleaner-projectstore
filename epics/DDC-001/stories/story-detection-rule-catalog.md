---
type: story
id: "story-detection-rule-catalog"
epic: "DDC-001"
title: "Detection rule catalog"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-18
external_refs: {}
tags: [rules, safety]
code_refs: ["crates/ddc-core/src/rules/", "crates/ddc-core/src/class.rs", "crates/ddc-core/tests/rules.rs", "crates/ddc-core/tests/common/mod.rs", "bench/make_baseline.py", "bench/check_baseline.py"]
specs: []
started_at: "2026-10-17T09:38:04.628Z"
closed_at: "2026-10-17T11:48:02.924Z"
plan_updated_at: "2026-10-17T09:38:04.628Z"
---

# Detection rule catalog

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The semantics of `ddc`: which directory is what, which risk class it belongs to and how to clean it. Based on the findings of [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] and the model in [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]] (classes, removal methods, rule priority, conditions for `rebuildable`). A rule recognizes an artifact by its marker and context, not only by its name. Rule ids are stable: they go into the JSON, the log and `bench/baseline-2026-10-17.tsv`.

### Project rules (specialized)

| id | Marker | Class | Method | Selection condition |
|---|---|---|---|---|
| `python.venv` | directory with `pyvenv.cfg` (any name) | `rebuildable` | `delete` | project lock file (`uv.lock`, `poetry.lock`, `requirements*.txt`, `Pipfile.lock`), otherwise `review` |
| `python.tox` | `.tox`, `.nox` + `tox.ini`/`noxfile.py`/`pyproject.toml` in the parent | `safe` | `delete` | — |
| `python.pycache` | `__pycache__` | `safe` | `delete` | — |
| `python.tool-cache` | `.pytest_cache`, `.mypy_cache`, `.ruff_cache` | `safe` | `delete` | — |
| `node.node_modules` | `node_modules` + `package.json` in the parent | `rebuildable` | `delete` | lock file (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lock*`), otherwise `review` |
| — (not a finding) | `node_modules` of global packages: `.nvm/versions/node/*/lib`, `.npm-global`, `.config/yarn/global`, `.bun/install/global`, `.volta`, `.fnm` | — | — | rules are not applied inside, the remainder goes into the directory total |
| `node.node_modules-orphan` | `node_modules` without `package.json` in the parent | `review` | `trash` | — |
| `js.build-cache` | `.next`, `.nuxt`, `.svelte-kit`, `.turbo`, `.parcel-cache`, `.angular` + `package.json` in the parent | `safe` | `delete` | — |
| `rust.target` | `target` + `Cargo.toml` in the parent | `rebuildable` | `delete` | manifest (build output) |
| `maven.target` | `target` + `pom.xml` in the parent | `rebuildable` | `delete` | manifest |
| `gradle.build` | `build` + `build.gradle*` in the parent **and** `intermediates/`, `classes/` or `tmp/` inside | `rebuildable` | `delete` | manifest |
| `gradle.project-cache` | `.gradle` + `build.gradle*`/`settings.gradle*` in the parent | `safe` | `delete` | — |
| `cmake.build` | directory with a conventional name (`build*`, `cmake-build-*`, `out`, `_build`, `.build`), with `CMakeCache.txt` **and without** `CMakeLists.txt`; `CMAKE_HOME_DIRECTORY` from the cache exists and lies **outside** the directory (an in-source build and `-S src -B .` are not found) | `rebuildable` | `delete` | — |
| `cocoapods.pods` | `Pods` + `Podfile` in the parent | `rebuildable` | `delete` | `Podfile.lock`, otherwise `review` |
| `ddc.tombstone` | the name contains `.ddc-deleting-`; checked **first** among the project rules, whatever markers of other rules lie inside; the general restrictions (protected folders, `.git` at the top level) apply to it too | `safe` | `delete` | — |
| — (not a finding) | `*.app` directory: the application bundle is sealed, rules are not applied inside; its bytes go to "outside findings" (`unowned_bytes`), large files inside are listed | — | — | — |
| `generic.cachedir-tag` | valid `CACHEDIR.TAG` signature; does not fire on ancestors of known paths | `review` | `trash` | — |

### Known-path rules (exact path from `home`, only inside the scan roots)

| id | Paths (from `~`) | Class | Method |
|---|---|---|---|
| `cache.npm` | `.npm/_cacache`, `.npm/_npx` | `safe` | `delete` |
| `cache.yarn` | `Library/Caches/Yarn`, `.yarn/berry/cache` | `safe` | `delete` |
| `cache.bun` | `.bun/install/cache` | `safe` | `delete` |
| `cache.pnpm` | `Library/pnpm/store`, `.pnpm-store` | `safe` | `delete` (node_modules from pnpm are clones and do not break) |
| `cache.uv` | `.cache/uv` | `safe` | `native` `uv cache clean` if `uv cache dir` = the finding's path; otherwise `delete` |
| `cache.pip` | `Library/Caches/pip` | `safe` | `delete` |
| `cache.poetry` | `Library/Caches/pypoetry/cache`, `Library/Caches/pypoetry/artifacts` | `safe` | `delete` |
| `cache.homebrew` | `Library/Caches/Homebrew` | `safe` | `delete` (`brew cleanup` is not used: it removes things outside the finding) |
| `cache.go-build` | `Library/Caches/go-build` | `safe` | `native` `go clean -cache` if `go env GOCACHE` = the finding's path; otherwise `delete` |
| `cache.go-mod` | `go/pkg/mod` | `safe` | `native` `go clean -modcache` if `go env GOMODCACHE` = the finding's path; otherwise `delete` with `u+w` |
| `cache.cargo` | `.cargo/registry/cache`, `.cargo/registry/src`, `.cargo/git/checkouts` | `safe` | `delete` |
| `cache.gradle` | `.gradle/caches` | `safe` | `delete` |
| `repo.maven` | `.m2/repository` | `review` (may contain locally installed artifacts) | `trash` |
| `cache.cocoapods` | `Library/Caches/CocoaPods` | `safe` | `delete` |
| `browsers.playwright` | `Library/Caches/ms-playwright` | `review` | `trash` |
| `browsers.puppeteer` | `.cache/puppeteer` | `review` | `trash` |
| `browsers.cypress` | `Library/Caches/Cypress` | `review` | `trash` |
| `xcode.derived-data` | `Library/Developer/Xcode/DerivedData` | `safe` (owner: Xcode) | `delete` |
| `xcode.simulator-caches` | `Library/Developer/CoreSimulator/Caches` | `safe` | `delete` |
| `xcode.device-support` | `Library/Developer/Xcode/iOS DeviceSupport` | `review` (needs the device) | `trash` |
| `xcode.archives` | `Library/Developer/Xcode/Archives` | `user-data` (dSYM) | `none` |
| `vm.docker-desktop` | `Library/Containers/com.docker.docker` | `review` | `native-only`: `docker --context desktop-linux builder prune -f`, `docker --context desktop-linux image prune -f`; no context — instruction only |
| `vm.podman` | `.local/share/containers` | `review` | `native-only`: `podman image prune -f` |
| `vm.colima`, `vm.lima`, `vm.orbstack`, `vm.rancher-desktop` | `.colima`, `.lima`, `.orbstack`, `Library/Application Support/rancher-desktop` | `review` | `native-only` without a command: instruction only |
| `cache.colima` | `Library/Caches/colima` | `safe` | `delete` |
| `toolchain.nvm` | each `.nvm/versions/node/<v>`, except protected versions (below) | `review` | `trash` |
| `toolchain.pyenv` | each `.pyenv/versions/<v>`, except protected versions | `review` | `native` `pyenv uninstall -f <v>` if `pyenv root` = `~/.pyenv`; otherwise `trash` |
| `toolchain.rustup` | each `.rustup/toolchains/<t>`, except protected versions | `review` | `native` `rustup toolchain uninstall <t>` if `rustup show home` = `~/.rustup`; otherwise `trash` |
| `model.ollama` | `.ollama/models` | `review` | `trash` (hint: `ollama rm` one model at a time) |
| `model.huggingface` | `.cache/huggingface` | `review` | `trash` |
| `model.lmstudio` | `.lmstudio/models` | `review` | `trash` |
| `model.torch` | `.cache/torch` | `review` | `trash` |
| `ide.jetbrains-cache` | `Library/Caches/JetBrains` | `safe` (owners: JetBrains IDEs) | `delete` |
| `ide.jetbrains-old-data` | `Library/Application Support/JetBrains/<Product><version>` if a newer version of the same product exists | `review` | `trash` |
| `ide.editor-cache` (split on 2026-10-18 into `ide.vscode-cache`, `ide.cursor-cache`, `ide.windsurf-cache`; see Addendum) | `Library/Application Support/{Code,Cursor,Windsurf}/{Cache,CachedData,CachedExtensionVSIXs,GPUCache,logs}` | `safe` (owner: the application) | `delete` |
| `ide.workspace-storage` | `Library/Application Support/{Code,Cursor,Windsurf}/User/workspaceStorage` | `user-data` (chat history and workspace state in `state.vscdb`) | `none` |
| `app.chrome-cache` | `Library/Caches/Google/Chrome` | `safe` (owner: Chrome) | `delete` |
| `app.brave-cache` | `Library/Caches/BraveSoftware` | `safe` (owner: Brave) | `delete` |
| `app.firefox-cache` | `Library/Caches/Firefox` | `safe` (owner: Firefox) | `delete` |
| `app.spotify-cache` | `Library/Caches/com.spotify.client` | `safe` (owner: Spotify) | `delete` |
| `app.slack-cache` | `Library/Application Support/Slack/Cache`, `Library/Application Support/Slack/Service Worker/CacheStorage` | `safe` (owner: Slack) | `delete` |
| `app.adobe-camera-raw-cache` | `Library/Caches/Adobe Camera Raw 2` | `safe` (owners: Lightroom, Photoshop) | `delete` |
| `trash` | `.Trash` | `review` | `none` (hint "empty in Finder") |
| `user.pictures`, `user.movies`, `user.music` | `Pictures`, `Movies`, `Music` | `user-data` | `none` |
| `user.downloads` | `Downloads` (region: rules work inside) | `user-data` | `none` |
| `user.ios-backups`, `user.imazing-backups` | `Library/Application Support/MobileSync/Backup`, `Library/Application Support/iMazing/Backups` | `user-data` | `none` |
| `user.telegram`, `user.whatsapp` | `Library/Group Containers/6N38VWS5BX.ru.keepcoder.Telegram`, `Library/Group Containers/group.net.whatsapp.WhatsApp.shared` | `user-data` | `none` |
| `user.cloud` (not walked) | `Library/CloudStorage`, `Library/Mobile Documents` | `user-data` | `none` |

### Protected toolchain versions

A version does not become a finding if:

- it is the default or pinned version: for pyenv — an exact name or a prefix (`3.12` → the highest installed `3.12.x`), a virtualenv name (`versions/<env>` → `<version>/envs/<env>`), and also any version with a non-empty `envs/`; for nvm — `.nvm/alias/default`, resolved the way nvm does it (exact version or prefix: `20` → the highest installed `v20.*`; `node`/`stable` → the highest installed; `lts/*` → the contents of `.nvm/alias/lts/*` (a name like `lts/iron`) → the contents of `.nvm/alias/lts/<name>` (a version), at most two steps); for pyenv — the versions from `.pyenv/version`; for rustup — `default_toolchain` from `.rustup/settings.toml`;
- a project file inside the scan roots refers to it: `.nvmrc`, `.node-version`, `.python-version`, `rust-toolchain`, `rust-toolchain.toml`;
- it is the only installed version.

If the default alias points to `system` or does not resolve, the rule produces no findings at all.

For rules with the `delete` method, an effective class of `review` (no lock file; a hidden top-level directory or `~/Library`) changes the method to `trash` (ADR §1); `--permanent` restores `delete`.

### Where project rules and `CACHEDIR.TAG` do not fire

Specialized rules and `CACHEDIR.TAG` never claim a scan root, `$HOME`, the protected folders of ADR §6 inside `$HOME`, or a directory with `.git` at its top level; system protected paths (`/`, `/Users`, `/System`, etc.) are rejected by the executor. Rules that recognize an artifact by its contents rather than its name (`python.venv`, `cmake.build`, `generic.cachedir-tag`) also do not claim a directory with a project manifest (`pyproject.toml`, `setup.py`, `package.json`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle*`, `Gemfile`, `composer.json`). The manifest check does not apply to name-based rules (`.next`, `__pycache__`, `node_modules`, tombstone, etc.): Next.js writes its own `{"type":"commonjs"}` to `.next/package.json`. This way a stray `CMakeCache.txt` in `~/Projects` or `python -m venv .` in a project does not turn the project into a finding, while `.next` stays a finding. The re-check in the executor (`Catalog::verify`) applies the same restrictions.

### Exclusions

`--exclude PATH` protects the path, everything under it and any finding that contains it (the whole finding becomes `user.kept`). An exclusion above the scan root protects the whole root. `.ddc-keep` works the same way: a marker in `~/Downloads` itself protects the whole region (like any `.ddc-keep` — the directory and everything under it); a marker deeper inside the region protects only the directory where it lies and the finding it falls into, but not the region. Version pins (`.nvmrc`, etc.) inside protected projects are taken into account.

### Hidden directory rule

All project rules with class `rebuildable` (`python.venv`, `node.node_modules`, `rust.target`, `maven.target`, `gradle.build`, `cmake.build`, `cocoapods.pods`) inside a hidden top-level directory of `$HOME` (`~/.vscode/extensions/…`, `~/.local/pipx/…`, `~/.local/share/nvim/lazy/…`, `~/.opencode/…`) or inside `~/Library` yield `review`: these artifacts belong to applications and tools (for example, the `target/` of an nvim plugin contains a loadable library).

## Decomposition

- [x] Rule model: `id`, category, risk class, removal method (`native`/`delete`/`trash`/`native-only`/`none`), match kind, native command, owner process (by name or bundle path), list of lock files, hint on how to restore — owner processes are matched by process name; matching by bundle path is a follow-up
- [x] Matching order per the ADR: exclusions → known paths (before the directory is read) → ancestors of known paths are only walked → specialized → `CACHEDIR.TAG`
- [x] Project rules from the table, with environments downgraded to `review` without a lock file
- [x] Known-path rules from the table; `~` resolved from `Env.home`, not from the real `$HOME`
- [x] The `--exclude PATH` flag and the `.ddc-keep` marker
- [x] `Catalog::verify(rule, path)` — the same matching code for the re-check in the executor
- [x] For each rule — a test that it fires and a test for a false positive (a table-driven test generated from the catalog) — firing per rule: `every_rule_is_covered_by_a_fixture`; false positives are tested for the ambiguous names (`ambiguous_names_without_markers_are_not_found`, `stray_cmake_caches_and_in_place_venvs_never_claim_projects`), not per rule

## Implementation Plan

Route follows the `projectstore:planner` plan (section "Rules") with the critic's corrections; contracts — ADR "Cleanup safety model: risk classes, plan and confirmation", §1–§3.

1. **`class.rs`** — `RiskClass { Safe, Rebuildable, Review, UserData }`, `Category`, `Removal { Delete, Trash, Native { cmd, fallback }, NativeOnly(&[cmd]), None }`, `NativeCommand { program, args, location }` (the "tool reports this path" check). Data and predicates only; execution is in the story "Safe cleanup: ddc clean".
2. **`rules/mod.rs`** — `Rule { id, category, class, removal, matcher, environment, lockfiles, owners, restore_hint, listed }`, `Matcher` (`KnownPaths`, `KnownChildren`, `Name{names, ctx}`, `ChildFile`, `CmakeBuild`, `NameContains`, `CacheDirTag`, `GlobalPackages`), `Catalog::builtin(&Env)` (known paths are resolved from `Env.home`), `Catalog::verify(rule, path)` — the same matching code for the re-check in the executor.
3. **`rules/catalog.rs`** — static tables for both sections of the story (ids and classes as in the tables and in `bench/make_baseline.py`).
4. **`rules/visitor.rs`** — `RuleVisitor: scan::Visitor`, priority per ADR §2: `before_read` — `--exclude` and `faccessat(.ddc-keep)` → `user.kept`; cloud folders → `Skip`; exact known path → `Claim` (`Downloads` → `Region`); children of `KnownChildren` → `Claim`; ancestors of known paths are marked. `after_read` — ancestor of a known path → `Descend`; specialized rules by name and by listing contents, siblings via `faccessat` in the parent; `CACHEDIR.TAG` (signature check) last. `on_file` collects `.ddc-keep` inside findings and version pin files (`.nvmrc`, `.node-version`, `.python-version`, `rust-toolchain*`).
5. **`rules/classify.rs`** — after the walk: effective class (lock file for environments, hidden directory rule → `review`), `.ddc-keep` inside a finding → `user.kept`, protected toolchain versions (nvm aliases, pyenv global version, rustup `default_toolchain`, pins from projects, the only version) are removed from findings, `listed: false` (global packages) are not shown.
6. **Tests** `crates/ddc-core/tests/rules.rs` — table-driven positive and negative cases, priority, lock files, hidden directories, `.ddc-keep`, `--exclude`, toolchains, catalog invariants (unique ids, VM without `delete`/`trash` and without forbidden commands, `user-data` → `none`), known paths only inside roots.
7. **Check on the reference machine** — a temporary `ddc scan` stub is not needed: the hidden `ddc du --rules` prints findings (id, class, path, size); comparison with `bench/baseline-2026-10-17.tsv` by the script `bench/check_baseline.py`.

Files touched: `crates/ddc-core/src/{class.rs,rules/}`, `crates/ddc-core/tests/rules.rs`, `crates/ddc-cli/src/cmd/du.rs`, `bench/check_baseline.py`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] False-positive tests: `env/` without `pyvenv.cfg`, `target/` without `Cargo.toml`, `build/` next to `*.xcodeproj` without Gradle signs, `src/` with `CMakeLists.txt` and `CMakeCache.txt` (in-source build) are not found — evidence: `crates/ddc-core/tests/rules.rs` `ambiguous_names_without_markers_are_not_found`
- [x] Priority tests: a uv venv with `CACHEDIR.TAG` → `python.venv`; `target/` with `Cargo.toml` and `CACHEDIR.TAG` → `rust.target`; `.tox/py311/pyvenv.cfg` is absorbed by the `.tox` finding; a directory with only `CACHEDIR.TAG` → `generic.cachedir-tag` (`review`); `~/.cargo/registry` with `CACHEDIR.TAG` yields findings `registry/cache` and `registry/src` (`safe`), not a single `review` finding — evidence: `rules.rs` `rule_precedence`
- [x] Test: a venv and `node_modules` without a lock file get `review`; `rust.target` without `Cargo.lock` stays `rebuildable` — evidence: `rules.rs` `environments_without_lock_file_need_review`, `lock_file_at_the_git_root_counts`
- [x] Whole-catalog test: ids are unique; no rule in the VM category has method `delete`/`trash`; the native command list of VM rules contains no `system prune`, `container`/`rm`, `volume` or `-a`; all `user-data` rules have method `none` — evidence: `rules.rs` `catalog_invariants`
- [x] Test: known-path rules do not fire outside the scan roots; protected toolchain versions do not become findings — including when `alias/default` = `node`, `lts/*` (via `alias/lts/*` → `lts/<name>` → version; the other installed versions are still found), a numeric prefix, a version from `.nvmrc`/`.python-version`/`rust-toolchain.toml` in a project, and a single installed version; `node_modules` of nvm global packages is not found; a venv in `~/.local/pipx/venvs/x`, `node_modules` in `~/.vscode/extensions/x` and `target/` next to `Cargo.toml` in `~/.local/share/nvim/lazy/x` get `review` — evidence: `rules.rs` `known_paths_only_match_inside_scan_roots_and_home`, `known_paths_do_not_match_outside_home`, `nvm_default_alias_resolution`, `single_toolchain_version_is_never_a_finding`, `pyenv_and_rustup_keep_global_and_pinned_versions`, `global_packages_and_cloud_folders_are_not_findings`, `app_owned_artifacts_need_review`
- [x] Test: `.ddc-keep` in an ancestor, `.ddc-keep` in the finding itself (known path `~/.cache/uv/.ddc-keep` and specialized rule `node_modules/.ddc-keep`), `.ddc-keep` deeper inside the finding (`node_modules/foo/.ddc-keep`) and `--exclude` turn the finding into `user-data` with the label "protected by user"; `node_modules` inside `Downloads` is found — evidence: `rules.rs` `keep_markers_and_excludes_protect_findings`, `a_keep_marker_inside_downloads_protects_only_its_finding`
- [x] Test: `*.ddc-deleting-*` with `pyvenv.cfg`/`CMakeCache.txt` inside → `ddc.tombstone` `safe`; `node_modules` inside `*.app` is not found; `CMakeCache.txt` in `~/Projects`, `$HOME`, the scan root, and with the source inside the build directory (`-S src -B .`) is not found; `pyvenv.cfg` in a directory with `.git` or a manifest is not found; `.next` with its own `package.json` is found; `--exclude` above the root gives `user.kept` for the root; `.nvmrc` inside an `--exclude` project protects the nvm version; pyenv: prefix `3.12`, virtualenv symlink `versions/<env>` and a version with a non-empty `envs/` do not become findings; every catalog rule is covered by a fixture (table-driven test) — evidence: `rules.rs` `a_tombstone_is_a_tombstone_whatever_it_contains`, `rules_never_fire_inside_app_bundles`, `stray_cmake_caches_and_in_place_venvs_never_claim_projects`, `project_artifacts_are_found_with_their_class` (`.next`), `excludes_inside_a_finding_or_above_the_root_protect_it`, `pins_inside_protected_projects_still_protect_toolchains`, `pyenv_prefixes_virtualenvs_and_envs_are_protected`, `every_rule_is_covered_by_a_fixture`
- [x] On the reference machine, from a terminal with Full Disk Access: for every still existing path from `bench/baseline-2026-10-17.tsv` there is a finding with the same rule and a path equal to or nested in the baseline path (or the other way round); paths from `inaccessible[]` do not count as misses; in total ≥ 90% of the baseline volume, recomputed with `du -sk` on the day of the check (result in the final summary) — evidence: 2026-10-17, `deepdrivecleaner du --rules ~` run by launchd as the binary holding Full Disk Access, then `python3 -I bench/check_baseline.py bench/baseline-2026-10-17.tsv <findings>`: covered 289.5 of 289.5 GiB = 100.0%; 1 miss of 16.4 MiB, so the script exits 1; accepted because rules deliberately do not apply inside `.app` bundles and the baseline row predates the generator fix (see final summary)

## Final Summary

**What changed.** The built-in rule catalog (`crates/ddc-core/src/rules/`): the rule table in `catalog.rs` (project artifacts, package caches, Xcode, VM runtimes, toolchains, ML models, IDE and app caches, user-data regions, `generic.cachedir-tag`), matchers in `matcher.rs` (including the CMake build-directory convention and `CMAKE_HOME_DIRECTORY` outside the directory), the `RuleVisitor` that claims subtrees during the walk (`visitor.rs`), classification with lock-file and app-owned downgrades, keep markers and `--exclude` (`classify.rs`), toolchain protection for nvm, pyenv and rustup (`toolchain.rs`), and `Catalog::claimable`/`content_matcher_allowed`/`verify`, which the executor reuses for its re-check. Risk classes, categories and removal methods live in `class.rs`.

**Why.** The catalog encodes what the research note "Where a developer's disk space goes (macOS)" found on the reference machine and the classes of ADR "Cleanup safety model: risk classes, plan and confirmation". Specialized rules never claim a scan root, `$HOME`, a protected folder or a directory with `.git`; content-based rules also never claim a directory with a project manifest.

**Tests.** `make check`: 107 tests, fmt and clippy clean (2026-10-17). For this story: `crates/ddc-core/tests/rules.rs` (25 tests, including `every_rule_is_covered_by_a_fixture`). `projectstore:reviewer` findings (nested `--exclude`, CMake and `$HOME` claims, the `cache.colima` category) were fixed and covered by tests; two critic passes on the doc deltas followed.

**Coverage on the reference machine (2026-10-17).** `deepdrivecleaner du --rules ~` was started by launchd, so it ran as the binary that holds Full Disk Access: 5,470,843 files, 544.78 GiB in 43.8 s, 2 errors. `bench/check_baseline.py` against `bench/baseline-2026-10-17.tsv`: covered 289.5 of 289.5 GiB = 100.0%, 0 unreadable rows, 0 gone. The one miss (16.4 MiB, `node.node_modules-orphan`) is `~/.zcode/computer-use/ZCode Computer Use.app/Contents/Resources/node_modules`: rules deliberately do not apply inside `.app` bundles, and the baseline row predates the generator fix that skips them.

**Risks and follow-ups.**
- Both errors in the coverage run were `EINTR` from the directory reader. The walker now retries `open` and `getattrlistbulk` on `EINTR` (`crates/ddc-core/src/sys/bulk.rs`); the installed binary was not rebuilt yet, because a rebuild of the unsigned binary drops its Full Disk Access grant.
- Owner processes are matched by name only (not by bundle path); false-positive tests exist for the ambiguous names, not for every rule. *Lifted on 2026-10-18; see Addendum.*
- Orphaned CMake build directories (source deleted, `CMAKE_HOME_DIRECTORY` missing) are not found.
- Private size is measured only where APFS clones are expected (uv, pnpm, bun); the "frees" total is an estimate.
- Regenerate the baseline from a process with Full Disk Access so it no longer lists rows inside `.app` bundles.

## Technical Notes

- Ambiguous `build/`, `dist/` without a reliable marker are not reported at all: otherwise there are hundreds of false findings (`dist` occurs 361 times).
- `node_modules` inside `node_modules` is not a separate finding: the outer one's `Claim` fires.
- `Pods` are sometimes kept in git; a `git ls-files` check is a follow-up.
- Poetry keeps project venvs in `~/Library/Caches/pypoetry/virtualenvs`: they are not part of the poetry cache rule and are found by the venv rule (project root unknown → `review`).
- A separate finding "`brew cleanup`: old formula versions" with the size from `brew cleanup -n` is a follow-up.
- Abandoned CMake build directories (sources deleted, `CMAKE_HOME_DIRECTORY` does not exist) are not found — follow-up.
- APFS private size is measured only where clones are expected: uv, pnpm and bun caches; uv venvs; `node_modules` from pnpm or next to `bun.lock*` (ADR "Rust core: library + CLI with JSON output").
- The reference machine baseline has no examples for these rules: `ide.jetbrains-old-data`, `python.tox`, `python.tool-cache`, `python.pycache` (excluded on purpose), `gradle.build`, `gradle.project-cache`, `maven.target`, `cocoapods.pods`, `generic.cachedir-tag`, `ddc.tombstone`, `toolchain.nvm`, `toolchain.pyenv`, `toolchain.rustup`, `repo.maven`, `cache.gradle`, `cache.cocoapods`, `browsers.cypress`, `xcode.device-support`, `xcode.archives`, `vm.orbstack`, `model.lmstudio`, `trash`, `user.cloud`. Only unit tests cover them, and the 90% criterion will not catch a regression in them.

## Addendum (2026-10-18)

The first real `ddc clean` showed the cost of matching owners by process name: macOS's `CursorUIViewService` matched `Cursor` and held back every editor cache. The story [[epics/DDC-001/stories/story-safe-cleanup-ddc-clean|Safe cleanup: ddc clean]] now matches owners by app bundle (`OwnerApp`, `rules::owns`), and splits `ide.editor-cache` into `ide.vscode-cache`, `ide.cursor-cache` and `ide.windsurf-cache`, one owner each. The catalog table above and the limitation in Technical Notes are kept as they were at close, marked where they no longer hold.

## Dependencies

- [[epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner|Workspace skeleton and parallel filesystem scanner]]

## Attachments

-

---

*Last updated: 2026-10-18*
