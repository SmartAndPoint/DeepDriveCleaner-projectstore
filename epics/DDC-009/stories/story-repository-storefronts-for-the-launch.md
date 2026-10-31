---
type: story
id: "story-repository-storefronts-for-the-launch"
epic: "DDC-009"
title: "Repository storefronts for the launch"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [launch, readme, github]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Repository storefronts for the launch

| Field | Value |
|---|---|
| **Epic** | [DDC-009](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The post sends readers to three repositories. Each one's first screen must answer three questions within ten seconds: what this is, how to try it, and where the other two fit in.

- **DeepDriveCleaner.**
  - GitHub description: today "Find and safely clean developer disk junk on macOS…". It becomes a developer disk cleaner that also finds what coding agents leave behind. The platforms it names are those of the latest release: "for macOS, with a Linux CLI next and Windows after" until a release ships Linux, then "for macOS and Linux, Windows next".
  - Topics: `disk-cleanup`, `developer-tools`, `rust`, `macos`, `linux`, `cli`, `coding-agents`, `claude-code`, `codex`, `cursor`, `opencode`.
  - The README's first screen: one image (from story "Launch demo assets"), the two install lines, a "Find what your coding agents left behind" line with the `ddc disk agents` command, a "For coding agents" link to the skill, and "Built in the open with ProjectStore" linking both the vault and ProjectStore. The rest of the README stays as it is.
- **The vault** (DeepDriveCleaner-projectstore).
  - Its README opens with what a visitor is looking at: the real backlog, decisions and runbooks behind DeepDriveCleaner, kept with ProjectStore.
  - It gives three entry points: the board (`kanban.md`), one ADR worth reading (the cleanup safety model), and one closed story with its evidence.
  - It links ProjectStore.
- **ProjectStore.** A "See it on a real project" link to the DeepDriveCleaner vault. ProjectStore is the owner's separate repository: the change is proposed as a PR there, or handed to the owner if that repository's own rules say otherwise.

These are public, outward-facing changes. They are drafted as PRs and merged only after the owner approves them (schedule in the epic).

## Decomposition

- [ ] A "Challenge entry" line in both READMEs (code and vault): "Submitted to the DEV Hacktoberfest Open-Source AI Challenge Week 4 at commit `<sha>`; commits after 2026-10-31 23:59 PDT are later work". It is merged before publication, and its commit is updated to the submitted one. This is the DEV FAQ's rule.
- [ ] Draft the DeepDriveCleaner README's first screen as a PR. The README is on `.config/rust-free-paths`, so the PR skips the Rust tiers.
- [ ] Draft the vault README as a vault PR.
- [ ] Draft the ProjectStore link, per that repository's contribution rules.
- [ ] Prepare the description and topics change as one `gh repo edit` command, run after the owner's approval.
- [ ] The owner approves, then everything merges or runs before the post is published.

## Implementation Plan

## Acceptance Criteria

- [ ] The three PRs are merged and the `gh repo edit` has run, before publication. Evidence: the PR numbers and `gh repo view --json description,repositoryTopics`.
- [ ] Both READMEs carry the "Challenge entry" line naming the submitted commit, merged before publication.
- [ ] Every link between the three repositories resolves (checked with `gh api` or `curl -sI`, status 200).
- [ ] Nothing in the changes describes the owner's machines or contains a private string: the private-scan hook passes.

## Final Summary

## Technical Notes

- Names and handles of people are not added to any README.

## Dependencies

- Story "Launch demo assets", for the image.
- DDC-003 story "Agent skill v0: ask ddc before guessing about the disk", for the "For coding agents" link. Without it the link is left out.

## Attachments

-

---

*Last updated: 2026-10-31*
