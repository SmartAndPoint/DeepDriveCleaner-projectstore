---
type: story
id: "story-docker-desktop-what-is-inside-its-disk-image-and-what-a-clean-frees"
epic: "DDC-002"
title: "Docker Desktop: what is inside its disk image, and what a clean frees"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-08
external_refs: {}
tags: [rules, docker, vm, review-window]
code_refs: []
specs: []
started_at: "2026-11-08T13:45:53.991Z"
closed_at: "2026-11-08T15:03:11.127Z"
plan_updated_at: "2026-11-08T13:45:53.991Z"
---

# Docker Desktop: what is inside its disk image, and what a clean frees

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

A tester (2026-11-01) saw Docker Desktop as one item of 26 GB and could not use it: "what exactly it wants to delete is unclear". The finding is the folder `~/Library/Containers/com.docker.docker` (rule `vm.docker-desktop`, `review`, `native-only`). Cleaning it runs `docker builder prune -f` and `docker image prune -f` after the readiness probe (story "VM images: docker prune runs only while Docker Desktop answers"). Those remove the build cache and dangling images only, yet the item shows the whole folder's size, and nothing says what is inside.

Docker can say it: `docker --context desktop-linux system df -v --format '{{json .}}'` prints one JSON object with every image, container, volume and build-cache record. This story reads it **on demand, never in a scan**, and shows it on the item:

- **Where it is read.** A new read-only query, `ddc disk docker --json`, beside `ddc disk agents`: the readiness probe (`DOCKER_READY`, as the executor runs it), then one `system df -v`. The Review window runs it when the Docker Desktop item is shown, and the confirmation sheet uses its result. No scan runs it: not `ddc scan`, not the background scan, not the clean's re-check, not a dry run. So a scheduled scan never spawns `docker` and never wakes Docker Desktop's VM from Resource Saver, and the scan's "native tools are only looked up, never run" principle stands. The menu bar ADR §6 gets a one-line dated amendment naming this query as the one tool the app runs to read, not to clean.
- **What is inside.** Images, containers, volumes and build cache, each with its count and size, summed from the `-v` records.
- **What a clean frees, as an estimate.** The two prune commands remove less than `docker system df` calls reclaimable: without `--all`, which the safety ADR forbids, `builder prune -f` keeps internal and frontend records and every record shared with an image, and a dangling image's `Size` includes layers shared with tagged images. So the estimate is: build cache = the sum of `Size` over `BuildCache` records with `InUse` false, `Shared` false and `CacheType` neither `internal` nor `frontend`; dangling images = the sum of `UniqueSize` over images with repository and tag `<none>` and no container. Both are sizes docker prints with three significant digits, so the figure is labelled "about".
- **"frees about".** Today a `native-only` item does not count in the sheet's "frees about"; its bytes are listed as "not counted" by runtime. With the query's result, the Docker item counts its estimate and leaves "not counted". Without it (the query failed or has not run), it stays "not counted", and its row says why, with the probe's own detail. The sheet's total is never unknown.
- **What it does not touch,** each with the command to run yourself: unused tagged images (`docker image prune -a`), stopped containers (`docker container prune`), volumes (`docker volume prune`, which deletes their data), and the rest of the build cache (`docker builder prune --all`). `ddc` never runs these.
- **When Docker does not answer,** the row shows the probe's detail and the earlier story's advice: "Docker Desktop is not running or not answering" → "Start Docker Desktop to see what is inside"; "the desktop-linux context does not exist" → "If you no longer use Docker Desktop, its folder is yours to delete in Finder"; no `docker` command → "Docker's command-line tool was not found".
- **After a clean,** the query runs again when the item is next shown; a cached result is never shown after a prune.

`system df` walks every image, container and volume and can take long with large volumes: the query has its own timeout (set at the plan gate: the probe keeps the executor's, `system df -v` gets its own), and a Stop or a closed window cancels it.

## Decomposition

- [x] Plan gate: on the owner's Docker Desktop, the probe and `system df -v` with the VM running, and the probe with Docker Desktop stopped; the owner kept the timeouts (Resource Saver not measured, see the plan)
- [x] Core: a `docker` module beside `brew/` and `conda.rs` behind `CommandRunner`: the probe, one `system df -v`, a tested parser of docker's size strings, the totals and the estimate
- [x] CLI: `ddc disk docker [--json]`, a schema file `docs/docker.schema.json` with its own `schema_version`, the README
- [x] DDCKit: decode, a model that runs the query for the shown item, the row text, the sheet's counted/not-counted change
- [x] Menu bar ADR §6: the dated one-line amendment

## Implementation Plan

Planner run 2026-11-08 against `origin/main` 86905fd. Two code PRs: Rust (core and CLI), then Swift (DDCKit and the app). A vault PR carries this plan and the menu bar ADR line.

**Plan gate, done 2026-11-08.** The owner's Docker Desktop answered the probe and `system df -v` well inside the timeouts with the VM running, and its records carry the fields this plan reads (`InUse`, `Shared`, `CacheType`, `UniqueSize`, `Repository`, `Tag`, `Containers`). (An earlier note said some size fields were empty strings: that came from the measuring script, which printed `""` for a key a section does not have. In docker's formatter no size is empty; an unknown one is `N/A`.) With Docker Desktop not running, docker 29 fails at once with "failed to connect to the docker API", which `DOCKER_READY` does not know yet. Resource Saver was not measured; the owner kept the defaults. So: the probe keeps `READY_TIMEOUT` (5 s, as the executor runs it), and `system df -v` gets `DF_TIMEOUT` 60 s; a Stop on the row ends either. Leaving Resource Saver can take longer than 5 s (Docker's documentation: about 3 to 10 seconds), so a probe that times out reads as "not running or not answering", and the row offers Try Again (a new query). No machine's timings go into this vault.

**What the breakdown shows, corrected against docker** (docker/cli's formatter, moby's summary). Every group counts all its records, as the summary's TOTAL column does. Sizes:
- images: the sum of `UniqueSize`, labelled "unique to each image". The `-v` records' `Size` counts shared layers once per image, and the summary's deduplicated figure needs a third call, which the ACs forbid;
- containers: the sum of `Size`, the writable layer (`-v` never prints the virtual size; a " (virtual …)" suffix is still cut, to be safe);
- volumes: the sum of the sizes given; `N/A` ones are left out, as the summary leaves them out;
- build cache: the sum of `Size` over records with `Shared` false, as the summary counts it ("shared with images is not counted here").
The row says why these do not add up to the folder's size: the sizes exclude shared layers, and Docker's disk image keeps space it freed until Docker Desktop shrinks it. The owner check says what "match" means.

1. **Rust PR.**
   - `crates/ddc-core/src/docker/` (beside `brew/`): `SYSTEM_DF` (the one command), `DF_TIMEOUT`, `contents(commands, cancel) -> DockerContents`: the readiness probe, a cancel check, one `system df -v`, the parse. `DockerContents { schema_version, ddc_version, result }`, `result` a strict enum tagged `status`: `listed { images, containers, volumes, build_cache, estimate }`, `not-installed`, `not-ready { detail }`, `failed { detail }`, `timed-out { seconds }`, `unreadable { detail }`. A group is `{ count, bytes? }`; the estimate `{ build_cache, dangling_images?, bytes? }`. No image, container or volume name is in the output.
   - The parser: docker's decimal sizes as bytes, any number of digits (`Size` prints three significant digits, `UniqueSize`, `SharedSize` and a volume's `Size` four: `1.234GB`, `123.4MB`, `1000kB`, and `1e+03kB`), units `B`, `kB`, `MB`, `GB`, `TB`, `PB`, `EB`; `N/A` is "not given"; anything else makes the result `unreadable`, never zero. Booleans are `"true"`/`"false"` only. The parser table carries the AC's examples and these.
   - The estimate exactly as the Description says: build cache = `Size` over `BuildCache` with `InUse` and `Shared` false and `CacheType` neither `internal` nor `frontend`; dangling images = `UniqueSize` over images with repository and tag `<none>` and `Containers` "0". An `N/A` among the dangling images makes `dangling_images` unknown; `estimate.bytes` is given only when both parts are known. An image kept only by its digest also prints `<none>`/`<none>`, and `image prune -f` keeps it, so the estimate can count it: one more reason it is "about" (Technical Notes).
   - The probe is reused, not copied: `exec::native::ready` gets a `pub(crate)` core (`readiness(probe, commands, cancel)`) the executor's wrapper keeps calling; `DOCKER_READY` becomes public where it is. `DOCKER_READY.known` gains docker 29's "failed to connect to the docker API" → `DOCKER_DOWN`, which also fixes the clean's skip detail on such a Mac (it printed the raw line, with the socket path). A probe that times out is `not-ready` with `DOCKER_DOWN`, not `timed-out`; `timed-out` is the df's.
   - CLI `ddc disk docker [--json] [--data-dir]` (`cmd/disk_docker.rs`, in the shape of `agent_status.rs`: signals installed, a stop ends by its signal), `DiskAction::Docker` in `cli.rs` with `prints_json`, `gate::refusal_for`'s exhaustive match, its text render, `docs/docker.schema.json` with its drift test, README. Exit 0 for every result; 1 only when no report can be printed; 2 on a usage error. Gated on no capability: nothing in it is OS-specific (seam ADR §3), and `vm.docker-desktop` is a shared rule, so a clean may run docker on any supported OS.
   - Tests: a fixture in the shape docker prints, with synthetic names and IDs and docker's real key set, `N/A` values and three- and four-digit sizes; the four totals, the estimate, exactly two recorded calls; the parser table; every failure (probe down with docker 29's and the older texts, no context, no `docker`, a probe timeout, a df timeout, a non-zero exit, garbage) with no third call; gate tests (`EVERY`, `prints_json`); no `docker` call from `scan`, `scan --background`, a dry run or the clean's re-check (a fake that fails on any call, in core and with a logging `docker` on `PATH` in the CLI); structure tests (the disk report's no-cleanup list gains the module; no `prune`, `rm`, `--all`, `Command::new` in it); the schema contract; Windows skips listed.
2. **Swift PR.**
   - `DockerContents.swift` (strict decode) and `DockerContentsModel.swift` (`show()` one run at a time, `stop()` by ticket, `forget()` after a clean; states unknown, reading, listed, unavailable), a lane kind `.dockerContents` (seconds, no lock, writes nothing: the lane rule of ADR §1).
   - The Review row of `vm.docker-desktop` runs `show()` when shown and lists images, containers, volumes and build cache with counts and sizes, what a clean frees (about), and what it does not touch, each with its command; a failure shows the probe's detail and the earlier story's advice; Stop while reading. A closed window stops it; a clean forgets the result.
   - The sheet: the estimate per row in `Sheet` (as `predictions`), filled at open from a fresh result, and again when a query started then finishes while the sheet is open; `Confirmation`'s "frees about" counts the Docker item's estimate and leaves it out of "not counted"; without one it stays not counted, with its reason. `ReviewModel.selectedReclaimable` follows the same rule. The progress bar's total stays as the CLI counts it, and the cleaning note no longer says the Docker item is not counted when the sheet counted it. The result shows docker's own "Total reclaimed space" line from the prune's output, so a clean's effect can be compared with the estimate.
   - Contract tests: the schema version and status enum against the Rust source; `run(.dockerContents` only in its model; no scheduled path names it; the `disk` callers list gains the new model.
3. **ADR.** One dated line in the menu bar ADR §6 item 10: `ddc disk docker --json` is the one `docker` call outside a clean, run by the Review window only, never by a scan, the background scan, a dry run or the clean's re-check; its exit codes; and the sheet's rule: the Docker item's estimate counts in "frees about" only after the query answered. The safety ADR §5 sentence ("for native-only the space … is not included in the total") is about the terminal's plan, which does not change: `ddc clean` and its totals still leave native-only items out.

Code refs, proposed: `crates/ddc-core/src/docker/`, `crates/ddc-core/src/exec/native.rs`, `crates/ddc-core/src/rules/`, `crates/ddc-cli/src/cmd/disk_docker.rs`, `crates/ddc-cli/src/{cli,gate,render}.rs`, `docs/docker.schema.json`, `README.md`, `app/Packages/DDCKit/Sources/DDCKit/{DockerContents,DockerContentsModel,CLIRunner,Confirmation,SkipSummary,CleanProgress,ReviewModel,CleanRun}.swift`, `app/DeepDriveCleaner/{ReviewView,ConfirmSheet}.swift`. The epic DDC-002 footprint gains `docs/docker.schema.json`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (core, fake `CommandRunner` with a hand-made `system df -v` output in the shape Docker Desktop prints, synthetic names only, and a ready probe): the four totals match the records; the estimate is the build-cache sum over unshared, unused, non-internal, non-frontend records plus the `UniqueSize` sum over dangling images with no container; exactly two `docker` calls are recorded, the probe and `system df -v` — evidence: crates/ddc-core/tests/docker.rs `the_totals_and_the_estimate_follow_the_records` (four groups, the estimate, calls == [probe, df], timeouts [READY_TIMEOUT, DF_TIMEOUT]), `the_output_names_no_image_container_or_volume`; PR #146
- [x] Test: the parser reads docker's size strings (`0B`, `512kB`, `1.23MB`, `12.3GB`, `1.1TB`) as decimal bytes; anything else is a parse failure, not zero — evidence: crates/ddc-core/src/docker/parse.rs `sizes_docker_prints_are_decimal_bytes`, `anything_else_is_not_a_size` (also `1.234GB`, `1e+03kB`; `N/A` is not given); crates/ddc-core/tests/docker.rs `what_ddc_cannot_read_is_unreadable_never_zero`
- [x] Test: with the probe failing (daemon down, no context, no `docker`), and with `system df -v` hanging past the timeout, exiting non-zero or printing garbage, the query reports the reason with the probe's detail and exits 0 with no breakdown; no other `docker` command is recorded — evidence: crates/ddc-core/tests/docker.rs `when_docker_does_not_answer_it_says_why_and_reads_nothing` (old and docker 29 wording, no context, no docker, probe timeout), `a_read_that_fails_or_hangs_says_so` (non-zero, df timeout, docker 29 wording without the socket path), `a_cancel_during_the_probe_runs_no_read`; crates/ddc-cli/tests/disk_docker_cli.rs `docker_not_answering_exits_0_with_the_reason`
- [x] Test: `ddc scan`, `ddc scan --background`, `ddc clean --dry-run` and the clean's re-check record no `docker` call (a fake runner that fails the test on any call) — evidence: crates/ddc-cli/tests/disk_docker_cli.rs `no_scan_and_no_dry_run_calls_docker` (a logging `docker` first on PATH records no call; the terminal dry run runs the re-check walk); the sheet's `clean --items --dry-run`: crates/ddc-core/tests/in_use.rs (no call); DDCKit ContractTests `dockerContentsCallers` (no scheduled path, scan or clean asks)
- [x] Test (schema): `ddc disk docker --json` validates against `docs/docker.schema.json`, and the committed schema equals the generated one — evidence: crates/ddc-cli/tests/disk_docker_cli.rs (`contract::holds("docker", …)`), crates/ddc-core/tests/docker.rs `committed_docker_schema_is_current`; DDCKit ContractTests `dockerContentsContracts` (version, statuses, field names)
- [x] Test (DDCKit): with a query result the Docker item counts its estimate in "frees about" and is not in "not counted"; with a failed query it stays in "not counted" and its row shows the reason; the sheet's total is a number in both cases; after a clean the next display runs the query again — evidence: DDCKit DockerContentsTests `sheet`, `sheetWithoutAnswer`, `once`, `query`, `stopAndLateAnswer`, `windowClosed`; CleanResultTests `dockerReclaimed`; PR #147
- [x] The executor still runs exactly `builder prune -f` and `image prune -f` after the readiness probe and nothing else (the tests of story "VM images: docker prune runs only while Docker Desktop answers" keep passing) — evidence: crates/ddc-core/tests/clean_safety.rs probe and prune tests unchanged and green (plus docker 29 wording in `the_probe_names_a_missing_context_or_says_what_docker_said`); `ready` maps the shared `readiness` one to one (crates/ddc-core/src/exec/native.rs)
- [x] `make check` and `make ddckit-test` pass — evidence: unit tier green on Linux (WSL, 1055/1055) and the native Windows leg (1091/1091); Gate green on #146 and on main fcf0a01; `make ddckit-test` 597/597 and a Debug app build; Gate green on #147
- [x] Owner check (runbook "Owner checks: manual acceptance across stories"): on a Mac with Docker Desktop running, the Review window's Docker item lists images, containers, volumes and build cache with counts equal to the TOTAL column of `docker system df`, sizes equal to its SIZE column, as docker rounds them, for containers, volumes and the build cache (images show the size unique to each, so they need not equal it), and after a clean the result shows docker's "Total reclaimed space", which is about the estimate — moved to runbook "Owner checks: manual acceptance across stories", section "Docker Desktop: what is inside, and what a clean frees"

## Final Summary

**What changed.** A read-only query, `ddc disk docker [--json]` (PR #146): the clean's own readiness probe, then one `docker --context desktop-linux system df -v`. It prints counts and sizes of images (unique to each), containers, volumes and build cache, and what the clean's two prunes would free, about, under `docs/docker.schema.json`; every docker answer exits 0 with a `status`. The parser reads docker's decimal sizes with three or four significant digits; `N/A`, and a volume's `-1B`, are "not given"; anything else is `unreadable`, never zero. The probe is shared with the executor, not copied, and knows docker 29's "failed to connect to the docker API", which also fixes the clean's skip detail. In the app (PR #147), the Review window's Docker Desktop item asks when shown, in a lane of its own, and lists what is inside, what a clean frees and what it leaves with the command to run, with Stop and Try Again; the sheet counts the estimate in "frees about" only after docker answered, once per runtime, and the item otherwise stays "not counted"; a clean that may have pruned, and a closed window, let go of the answer; the result shows docker's own reclaimed figures.

**Why.** A tester saw Docker Desktop as one 26 GB item and could not tell what a clean would remove.

**Tests.** Core and CLI tests for totals, estimate, parser, every failure, two calls only, no call from a scan, background scan or dry run, schema; DDCKit tests for decoding, the query's states, Stop and a late answer, the sheet's counting and the reclaimed line; contract tests pin the version, the statuses, the field names and the callers.

**Risks and follow-ups.** Sizes in the Docker row are decimal, as docker prints them; the sheet's header stays binary, so the two figures differ in units. Resource Saver was not measured: a probe that times out reads as "not running or not answering", with Try Again. On the containerd image store docker may give no shared size, so `UniqueSize` would be `N/A` and the estimate unknown; not checked. When the estimate is unknown the sheet's "not counted" line does not say why; the row does.

## Technical Notes

- `system df -v --format '{{json .}}'` prints one object `{"Images":[…],"Containers":[…],"Volumes":[…],"BuildCache":[…]}`; its values are strings (`"InUse":"true"`, `"Size":"1.2GB"`, `"Containers":"0"`). The non-`-v` summary lines carry a reclaimable figure that overstates what the prunes free; do not use it.
- Reuse the readiness probe; do not add a second one.
- An image kept only by its digest prints `<none>`/`<none>` in `system df -v` (its `Digest` field is empty there), and `docker image prune -f` does not remove it: the estimate cannot tell it from a dangling image. On Windows the probe's process name would be `com.docker.backend.exe` (not this story's concern).
- The query is the first `docker` call outside a clean; it never runs from the background path (safety ADR §8: the background path scans only).

## Dependencies

- Story "VM images: docker prune runs only while Docker Desktop answers" (done): the readiness probe and the prune commands.
