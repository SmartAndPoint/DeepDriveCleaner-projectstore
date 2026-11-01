---
type: story
id: "story-docker-desktop-what-is-inside-its-disk-image-and-what-a-clean-frees"
epic: "DDC-002"
title: "Docker Desktop: what is inside its disk image, and what a clean frees"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [rules, docker, vm, review-window]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Docker Desktop: what is inside its disk image, and what a clean frees

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
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

`system df` walks every image, container and volume and can take long with large volumes: the query has its own timeout (set at the plan gate from a measurement on the owner's Docker Desktop, with Resource Saver on and off), and a Stop or a closed window cancels it.

## Decomposition

- [ ] Plan gate: measure `docker version` and `system df -v` wall time on the owner's Docker Desktop, VM running and in Resource Saver; record whether `version` wakes the VM; set the timeout
- [ ] Core: a `docker` module beside `brew/` and `conda.rs` behind `CommandRunner`: the probe, one `system df -v`, a tested parser of docker's size strings, the totals and the estimate
- [ ] CLI: `ddc disk docker [--json]`, a schema file `docs/docker.schema.json` with its own `schema_version`, the README
- [ ] DDCKit: decode, a model that runs the query for the shown item, the row text, the sheet's counted/not-counted change
- [ ] Menu bar ADR §6: the dated one-line amendment

## Implementation Plan

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (core, fake `CommandRunner` with a recorded `system df -v` output from Docker Desktop and a ready probe): the four totals match the records; the estimate is the build-cache sum over unshared, unused, non-internal, non-frontend records plus the `UniqueSize` sum over dangling images with no container; exactly two `docker` calls are recorded, the probe and `system df -v`
- [ ] Test: the parser reads docker's size strings (`0B`, `512kB`, `1.23MB`, `12.3GB`, `1.1TB`) as decimal bytes; anything else is a parse failure, not zero
- [ ] Test: with the probe failing (daemon down, no context, no `docker`), and with `system df -v` hanging past the timeout, exiting non-zero or printing garbage, the query reports the reason with the probe's detail and exits 0 with no breakdown; no other `docker` command is recorded
- [ ] Test: `ddc scan`, `ddc scan --background`, `ddc clean --dry-run` and the clean's re-check record no `docker` call (a fake runner that fails the test on any call)
- [ ] Test (schema): `ddc disk docker --json` validates against `docs/docker.schema.json`, and the committed schema equals the generated one
- [ ] Test (DDCKit): with a query result the Docker item counts its estimate in "frees about" and is not in "not counted"; with a failed query it stays in "not counted" and its row shows the reason; the sheet's total is a number in both cases; after a clean the next display runs the query again
- [ ] The executor still runs exactly `builder prune -f` and `image prune -f` after the readiness probe and nothing else (the tests of story "VM images: docker prune runs only while Docker Desktop answers" keep passing)
- [ ] `make check` and `make ddckit-test` pass
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): on a Mac with Docker Desktop running, the Review window's Docker item lists images, containers, volumes and build cache with counts and sizes that match `docker system df`, and a clean frees about the estimate

## Final Summary

## Technical Notes

- `system df -v --format '{{json .}}'` prints one object `{"Images":[…],"Containers":[…],"Volumes":[…],"BuildCache":[…]}`; its values are strings (`"InUse":"true"`, `"Size":"1.2GB"`, `"Containers":"0"`). The non-`-v` summary lines carry a reclaimable figure that overstates what the prunes free; do not use it.
- Reuse the readiness probe; do not add a second one. On Windows the probe's process name would be `com.docker.backend.exe` (not this story's concern).
- The query is the first `docker` call outside a clean; it never runs from the background path (safety ADR §8: the background path scans only).

## Dependencies

- Story "VM images: docker prune runs only while Docker Desktop answers" (done): the readiness probe and the prune commands.
