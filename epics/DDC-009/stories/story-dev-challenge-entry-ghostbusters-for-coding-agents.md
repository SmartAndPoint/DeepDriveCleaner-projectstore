---
type: story
id: "story-dev-challenge-entry-ghostbusters-for-coding-agents"
epic: "DDC-009"
title: "DEV challenge entry: Ghostbusters for coding agents"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-01
external_refs: {challenge: "https://dev.to/challenges/hacktoberfest-week4-2026-10-26"}
tags: [community, agents, writing, demo]
code_refs: []
specs: []
started_at: "2026-10-31T15:10:00.000Z"
closed_at: "2026-11-01T00:00:00.000Z"
plan_updated_at: null
---

# DEV challenge entry: Ghostbusters for coding agents

| Field | Value |
|---|---|
| **Epic** | [DDC-009](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner enters DeepDriveCleaner in the DEV "Hacktoberfest Open-Source AI Challenge Week 4: Ghost in the Machine" (2026-10-31). Submissions close on 2026-10-31 at 23:59 PDT. The pitch is "Ghostbusters for coding agents". AI coding agents leave ghosts on the disk:
- transcripts of sessions nobody will reopen;
- abandoned worktrees with their own `node_modules` and `target`;
- downloaded runtimes and caches.

`ddc disk agents` finds them, tells a live session from a dead one, and removes nothing on its own. The direction: DeepDriveCleaner as the place where agents themselves ask about disk usage, through stable JSON contracts, on macOS, Linux and soon Windows.

Facts the post must keep straight:
- **Dates.** The repository starts on 2026-10-17. In the challenge week (2026-10-26 to 2026-10-31) came:
  - 224 of its 246 commits, and releases 0.3.0 to 0.12.1;
  - the agent footprint report (2026-10-28);
  - the platform seam and the Linux and Windows catalogs (2026-10-30 and 2026-10-31).
  
  The post says "most of it this week", not "all of it".
- **No model inside.** DeepDriveCleaner is not an AI model. Its AI side:
  - it is built for coding agents, including the open-source ones it recognises (opencode, Codex CLI);
  - it is built by agents, through this vault's process (stories, ADRs, critic and reviewer agents, every change through a PR).
  
  The post says so plainly. It does not claim an open-weight model.
- **The licence** is PolyForm Noncommercial 1.0.0: the source is public, but it is not an OSI open-source licence. The post calls it "source-available" unless the owner relicenses first.
- **Privacy.** The demo runs on a synthetic home folder, never the owner's. No private machine is named, per the runbook "Native runs: the Linux and Windows legs on the owner's machines".

## Decomposition

- [x] Read the challenge rules and the submission template (2026-10-31).
- [x] A demo home folder with synthetic agent data, and `ddc disk agents` output from it.
- [x] The post draft v1 and v2 (2026-10-31): v2 carries the owner's real numbers as totals only, with no project names or session ids, the `/private/tmp` finding, and links to ProjectStore.
- [x] Post v3 in two variants, "released" and "not released" (the default), ready early on the morning before the deadline, following the template:
  - What I Built;
  - Demo;
  - Code;
  - How I Built It;
  - Why Does Open Innovation Matter?;
  - The Spirit of It.

  Done: v3 to v5, then the published text; the "released" variant went out, because 0.14.0 was published before the post.
- [x] The owner reviews the post, decides the licence wording, adds screenshots or a short screen recording of the app, and publishes it from their DEV account before the deadline. — done: the licence stays PolyForm Noncommercial, described as source-available; the owner's screen recording went in as a cut GIF with the user-data names blurred.
- [x] Before publication, every number and date is re-checked against `git log`, the releases and the vault. A feature that is not released is worded as "next". — done by the critic before publication.
- [x] Tags: `devchallenge` and `hf26challenge` (both required by the template), plus two of `opensource`, `ai`, `rust`, `macos`. The cover image from story "Launch demo assets". — done: `devchallenge`, `hf26challenge`, `ai`, `rust` (the DEV API), and the cover is set.
- [x] "My Agent Session": a public link, the vault PR of this epic with the critic's review folded in. No transcript is uploaded, and no third-party session recorder is used.
- [x] "Prize Categories": removed (D2: overall only), unless the owner decides otherwise.
- [x] The "How I Built It" section names the open-source pieces: Codex CLI and ProjectStore.
- [x] The owner's checklist, with minute budgets: D1, D2, screenshots, storefront approvals, the variant to publish, publication, and the post URL recorded here. — the post: https://dev.to/evgeniikonev/ghostbusters-for-coding-agents-12-gib-of-ghosts-on-my-mac-and-15-gib-more-i-didnt-know-about-1ok7

## Implementation Plan

## Acceptance Criteria

- [x] The post is published on DEV before 2026-10-31 23:59 PDT, with the tags `devchallenge` and `hf26challenge`. Evidence: the post URL. — evidence: https://dev.to/evgeniikonev/ghostbusters-for-coding-agents-12-gib-of-ghosts-on-my-mac-and-15-gib-more-i-didnt-know-about-1ok7; the DEV API gives `published_at` 2026-11-01T06:56:08Z (2026-10-31 23:56 PDT) and the tags `devchallenge`, `hf26challenge`
- [x] Every number and date in the post matches `git log`, the releases and the vault on the day it is published. — evidence: the critic's pre-publication check; `gh pr list --state merged` gives 58 code and 36 vault pull requests merged before `published_at` (vault #1 was closed, not merged); 0.14.0 was published before the post

## Final Summary

**What changed.** The entry is published on DEV with the tags `devchallenge`, `hf26challenge`, `ai` and `rust` and a cover, before the deadline. The post frames coding agents as the ghosts and what they leave on disk as their slime, shows the owner's real totals (no project names), the 15 GiB temporary-folder finding, a terminal GIF on the synthetic demo home, a cut of the owner's screen recording of the Mac app, and the ProjectStore loop. "How I Built It" makes the method the story: velocity numbers, the Rust core and Swift app, the test counts, and the benchmark against nine other tools worded as "same league, not fastest". The licence is described as source-available.

**Why.** The goal was an audience for DeepDriveCleaner and for ProjectStore as a method, more than the prize.

**Checks.** Two critic passes on the text (facts against `git log`, the releases and the vault; theme fit; tone), a final check of the rendered preview, and the DEV API after publication.

**Follow-ups.** The README's "Challenge entry" line ships with the next patch; the storefronts, the +2/+7-day numbers and the cross-posts are the other stories of this epic.

## Technical Notes

- Only the owner publishes: the post goes out under their name.
- The session re-checks the post's numbers just before handing it over; the owner checks them again at publication.

## Dependencies

- Story "Launch demo assets" (the demo block and the cover).
- Story "Repository storefronts for the launch" (where the links land).
- DDC-003 story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees" (only for the "released" variant).

## Attachments

-

---

*Last updated: 2026-10-31*
