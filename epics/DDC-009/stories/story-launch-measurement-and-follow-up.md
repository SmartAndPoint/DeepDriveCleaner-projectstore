---
type: story
id: "story-launch-measurement-and-follow-up"
epic: "DDC-009"
title: "Launch measurement and follow-up"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [launch, community, metrics, writing]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Launch measurement and follow-up

| Field | Value |
|---|---|
| **Epic** | [DDC-009](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

A launch without numbers cannot inform the next one. This story records where the three repositories stand before the post, follows up with readers, and prepares the second push: a post about ProjectStore as a method.

- **Baseline (2026-10-31):** DeepDriveCleaner ★4, the vault ★0, ProjectStore ★12. Before publishing, record:
  - stars and forks (`gh repo view --json stargazerCount,forkCount`);
  - 14-day views and clones (`gh api repos/<repo>/traffic/views` and `…/clones`), which need push access, which the owner has.
- **+2 and +7 days:**
  - the same numbers;
  - DEV's views and reactions for the post;
  - the referrers (`traffic/popular/referrers`).
- **Comments.** The owner answers comments on the post within a day. "Here is what ddc found on my machine" reports become issues or rule stories when they name a new place where agents keep data.
- **The second post, the week after:** about ProjectStore, roughly "One developer and a team of agents: ten releases in a week, and why nothing broke". It covers:
  - the vault;
  - the critic and the reviewer;
  - PR-only `main`;
  - the private-scan hook;
  - real examples from this project, such as the 15 GB finding that became a story before any code.
- **Cross-posts while the launch is fresh**, because stars arrive in the first 72 hours:
  - +1 to +3 days: Show HN and r/ClaudeAI;
  - with the ProjectStore post, the week after: r/rust and Lobsters.
  
  Each follows its venue's self-promotion rules, which the owner checks first.

## Decomposition

- [ ] The baseline numbers, recorded in this story before publication.
- [ ] The +2 and +7-day numbers.
- [ ] Draft the ProjectStore post. It goes through the critic like any other vault text before the owner publishes it.
- [ ] A cross-post plan with dates, approved by the owner.

## Implementation Plan

## Acceptance Criteria

- [ ] The baseline, +2 and +7-day numbers are recorded here with dates and the commands that produced them.
- [ ] The ProjectStore post draft exists and is critic-approved.

## Final Summary

## Technical Notes

- The traffic API keeps 14 days. The +7-day reading covers the whole launch.
- Every post and cross-post is published by the owner, under their name.

## Dependencies

- Story "DEV challenge entry: Ghostbusters for coding agents".

## Attachments

-

---

*Last updated: 2026-10-31*
