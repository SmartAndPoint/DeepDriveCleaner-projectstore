---
type: epic
id: "DDC-009"
title: "Ghostbusters launch: the DEV challenge and a first audience"
status: in-progress
priority: p1
created: 2026-10-31
updated: 2026-10-31
external_refs: {challenge: "https://dev.to/challenges/hacktoberfest-week4-2026-10-26"}
tags: [launch, community, writing, demo, agents]
code_refs: []
review_status: pending
reviewed_at: null
---

# DDC-009: Ghostbusters launch: the DEV challenge and a first audience

| Field | Value |
|---|---|
| **Status** | in-progress |
| **Priority** | p1 |
| **Created** | 2026-10-31 |
| **Updated** | 2026-10-31 |

---

## Goal

DeepDriveCleaner enters the DEV "Hacktoberfest Open Source AI Challenge: Ghost in the Machine" as "Ghostbusters for coding agents". The owner's aim is not the prize (2026-10-31). It is an audience:
- people who try DeepDriveCleaner and star it;
- developers who discover ProjectStore, the method the project is built with.

The challenge is the first push. A second post, about ProjectStore, follows the week after.

Success criteria:

- The entry is published before 2026-10-31 23:59 PDT, with the required tags `devchallenge` and `hf26challenge`.
- What the post shows is true in the released version a reader installs that day. A feature that is not released is described as next, not as done.
- A visitor who follows a link from the post finds, within ten seconds of landing on each repository, what it is, how to install it, and how it relates to the other two:
  - DeepDriveCleaner;
  - its vault;
  - ProjectStore.
- Stars and traffic are measured before publication and after 2 and 7 days. The numbers go into this epic's Final notes, and the second post plans from them.

## Context

- On 2026-10-31 the owner ran `ddc disk agents` on the reference Mac:
  - about 12 GiB of coding-agent data under the home folder: Codex 5.5, Claude Code 3.5, Cursor 2.8 and opencode 0.07 GiB;
  - about 15 GB more in Claude Code's per-session temporary folders under `/private/tmp`, which `ddc` does not see.
  
  The owner set the product priority from this: "working with the disk: show how much each agent session ate in its own working files and how to clean it, and a skill for agents". The two product stories in DDC-003 below carry it.
- Baseline on 2026-10-31: DeepDriveCleaner ★4, its vault ★0, ProjectStore ★12.
- How the challenge is judged, from its page:
  - writing quality, weighted most;
  - relevance to the theme, with open-source AI or open-weight models central;
  - creativity;
  - technical execution.
  
  DeepDriveCleaner has no model inside, which is the entry's weak point. The post says so plainly. Its AI side is that agents built it, and that it serves agents, including the open-source ones it recognises (opencode, Codex CLI).
- The template asks which open-source AI the project uses: "open-weight models, agent harnesses, frameworks, local inference". The honest answer:
  - two harnesses built the project: Claude Code and the open-source Codex CLI (Apache-2.0);
  - both worked from one open-source framework, ProjectStore (MIT);
  - no open-weight model runs inside the tool.
  
  This also serves the owner's real goal, because it puts ProjectStore in the "How I Built It" answer.
- The DEV FAQ: commits made after the deadline must be noted in the project's README. Both repositories bump a version on every PR, so each README carries a "challenge entry" line naming the submitted commit.
- The licence is PolyForm Noncommercial 1.0.0, source-available rather than OSI open source. The post words it that way unless the owner relicenses (decision D1).
- The repository starts on 2026-10-17. Most of the work fell in the challenge week (224 of 246 commits by 2026-10-31). The post claims "most of it this week", never "all of it".

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-dev-challenge-entry-ghostbusters-for-coding-agents](./stories/story-dev-challenge-entry-ghostbusters-for-coding-agents.md) | in-progress | The post: draft, real numbers, demo, honesty notes, owner's review, publication before the deadline |
| [story-repository-storefronts-for-the-launch](./stories/story-repository-storefronts-for-the-launch.md) | planned | Descriptions, topics and the first screen of the READMEs of DeepDriveCleaner, its vault and ProjectStore, linked to each other; an "For coding agents" entry point |
| [story-launch-demo-assets](./stories/story-launch-demo-assets.md) | planned | A synthetic demo home with agent ghosts, a recorded terminal demo, screenshots of the app's Coding agents section, and a cover image |
| [story-launch-measurement-and-follow-up](./stories/story-launch-measurement-and-follow-up.md) | planned | Baseline and +2/+7-day numbers, answering comments, the ProjectStore post and cross-posts the week after |

The product work the entry depends on lives in DDC-003 "Agent footprint":
- "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees";
- "Agent skill v0: ask ddc before guessing about the disk".

## Schedule (deadline 2026-10-31 23:59 PDT)

| When | What | Who |
|---|---|---|
| The evening before | This plan, critic-approved, merged | session |
| Overnight, P0 (about 2 h) | Post v3 fixes. The demo block regenerated with a live and a stale session. The cover image. Storefront PR drafts. The README "challenge entry" note. The owner's checklist | session |
| Overnight, P1 (about 5 h) | The per-session report MVP (Claude Code only), through the gate. Then a release, run the way earlier releases were (`make tag`, then `make dist` from an unsandboxed shell). It also ships the Linux CLI that `main` already has | session |
| Overnight, P2 | Agent skill v0 files and their structure test | session |
| The morning before the deadline, early | Post v3 ready in two variants: "released" and "not released" | session |
| The morning, owner's window | D1 and D2; screenshots; approve the storefront PRs; publish the matching variant. If a release lands later, edit the live post to the released variant (DEV allows edits before the deadline) | owner |
| 23:59 PDT | Deadline | — |

The "not released" variant is the default. It shows what 0.12.1 does, describes the per-session report and the Linux CLI as "on `main`, in the next release", and links the stories. The session re-checks every number in the post just before handing it over. The owner checks again at publication.

## Expected Results

- [ ] The entry is published on time, with every claim true for the version a reader installs.
- [ ] The three repositories' first screens are updated and link to each other.
- [ ] Measurement at +2 and +7 days is recorded.
- [ ] The ProjectStore post is drafted.

## Dependencies

- DDC-003 stories "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees" and "Agent skill v0: ask ddc before guessing about the disk".

## Open Questions

- [ ] D1, the owner: keep PolyForm Noncommercial and say "source-available", or relicense before publishing.
- [ ] D2, the owner: which prize categories to list. The partner categories need a partner technology; none is used. The proposal is the overall category only.

## Related

- [[epics/DDC-003/epic|DDC-003 · Agent footprint]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]

---

*Last updated: 2026-10-31*
