---
type: story
id: "story-humantime-to-jiff-rfc-3339-times-without-an-unmaintained-crate"
epic: "DDC-005"
title: "humantime to jiff: RFC 3339 times without an unmaintained crate"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-18
updated: 2026-11-18
external_refs: {}
tags: ["deps", "cargo-deny", "time"]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# humantime to jiff: RFC 3339 times without an unmaintained crate

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On 2026-11-18 cargo deny failed main's full Gate on RUSTSEC-2026-0543 (chrono unmaintained) and RUSTSEC-2026-0542 (humantime; the advisory says it wraps chrono, though humantime 2.4.0 in `Cargo.lock` has no dependency, so removing it takes no path to chrono away). Neither is a vulnerability, and the owner chose to let both through `deny.toml` with reasons (PR #174) so 0.38.0 could go out. humantime formats and parses the RFC 3339 times of every report and of the action journal (about 40 calls in `ddc-core` and `ddc-cli`); chrono is used once for the Linux trash's deletion date and comes in through the `trash` crate too.

This story moves humantime's calls to jiff (the replacement RUSTSEC-2026-0543 recommends for chrono), with byte-for-byte the same output, and takes RUSTSEC-2026-0542 out of `deny.toml`. chrono stays ignored while `trash` depends on it; whether the Linux trash's own date moves to jiff is the plan's question.

## Decomposition

Rough until the plan gate.

- [ ] Every humantime call moved to jiff behind one small module of `ddc-core`, with the same text out and the same leniency in
- [ ] The Linux trash's `DeletionDate` written without chrono, if jiff gives the same local-time format
- [ ] `deny.toml`: RUSTSEC-2026-0542 removed; RUSTSEC-2026-0543 kept with its reason while `trash` needs chrono
- [ ] `cargo deny check` on main, the machete and msrv steps of the full Gate

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test: the times every report and the journal write are the same strings as before, from fixed instants, and the parsers accept what they accepted and reject what they rejected (the strict one takes only `...Z`; jiff takes offsets and lowercase, so a check is needed)
- [ ] `deny.toml` ignores no advisory for humantime and bans the crate (`[bans] deny = [{ name = "humantime" }]`), so the Gate refuses it coming back
- [ ] main's full Gate is green, deny included

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The advisories: https://rustsec.org/advisories/RUSTSEC-2026-0542, https://rustsec.org/advisories/RUSTSEC-2026-0543.
- jiff's MSRV must fit the workspace's (`make msrv`); its licence (MIT OR Unlicense) passes the allow list; `default-features = false, features = ["std"]` while the trash date stays on chrono.
- Byte-for-byte output: format through jiff's strftime, since its `Display` prints fractional seconds. `disk/home.rs` and `footprint/mod.rs` treat a time that does not parse as unreadable: keep that.

## Dependencies

- PR #174 (the ignore), its reasons in `deny.toml`

## Attachments

-

---

*Last updated: 2026-11-18*
