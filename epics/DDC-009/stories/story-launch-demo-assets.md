---
type: story
id: "story-launch-demo-assets"
epic: "DDC-009"
title: "Launch demo assets"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [launch, demo, screenshots]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Launch demo assets

| Field | Value |
|---|---|
| **Epic** | [DDC-009](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The challenge requires a demo. Readers decide in seconds from one image. The assets show real behaviour without exposing anyone's real data.

- **A synthetic demo home** (`bench/demo/agent-ghosts.sh` in the code repository, or a script in the session's scratch space if the owner prefers it out of the repository). It builds:
  - a home folder with Claude Code, Codex, Cursor and opencode data;
  - two sessions, one of them live;
  - a session temporary folder;
  - a worktree.
  
  `HOME=<demo> ddc disk agents` then shows every ghost kind, with sizes large enough to read (hundreds of MiB, sparse where the scanner counts allocated blocks; the script says which).
- **A terminal recording**, 30 to 45 seconds:
  1. `ddc disk agents` on the demo home;
  2. the per-session view (if released);
  3. `ddc disk agents --json | jq` piped into a question an agent would ask.
  
  Made with a recorder the owner already has, or as a sequence of screenshots. No tool is installed for it without the owner's approval.
- **Screenshots of the Mac app's "Coding agents" section**, taken by the owner on the reference Mac. The owner checks them for project names they do not want public; blurring is fine.
- **A cover image** for DEV (1000×420): the app icon, "Ghostbusters for coding agents", and the totals line "12 GB + 15 GB of agent ghosts on one Mac".

## Decomposition

- [ ] The demo-home script, idempotent, writing only inside the folder it is given. It refuses a path inside the real home.
- [ ] The recording script: the commands and pauses, so the owner can record in one take.
- [ ] Cover image (SVG source plus PNG), made from the app icon in `scripts/icons/`.
- [ ] The owner's screenshots, checked for private names.

## Implementation Plan

## Acceptance Criteria

- [ ] The demo-home script builds the tree twice in a row with the same result, and refuses `$HOME` and any path inside it (test or recorded run).
- [ ] `ddc disk agents` on the demo home shows all four harnesses and at least one live and one dead session (recorded output).
- [ ] The cover image is 1000×420 and readable at 50 % size.
- [ ] Manual (owner), recorded in the owner-checks runbook: the screenshots show no project or session name the owner does not want public.

## Final Summary

## Technical Notes

- `ddc disk agents` is never pointed at the owner's real home folder by a session. The owner's own run of 2026-10-31 supplies the real totals.
- Liveness reads a Claude Code record `sessions/<pid>.json` and checks only pid, uid and start time; there is no process-name check. So the script writes two records: one naming its own `sleep` child (live), and one naming a pid that has exited (stale). The demo then shows both a live and a stale session, which is the post's "live is not dead" claim.

## Dependencies

- DDC-003 story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees", for step 2 of the recording.

## Attachments

-

---

*Last updated: 2026-10-31*
