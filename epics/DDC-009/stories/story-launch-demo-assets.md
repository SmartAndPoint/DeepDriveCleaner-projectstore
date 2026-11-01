---
type: story
id: "story-launch-demo-assets"
epic: "DDC-009"
title: "Launch demo assets"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-01
external_refs: {}
tags: [launch, demo, screenshots]
code_refs: ["bench/demo/agent-ghosts.sh", "bench/test_agent_ghosts.py"]
specs: []
started_at: "2026-11-01T00:00:00.000Z"
closed_at: "2026-11-01T00:00:00.000Z"
plan_updated_at: null
---

# Launch demo assets

| Field | Value |
|---|---|
| **Epic** | [DDC-009](../epic.md) |
| **Status** | done |
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

- [x] The demo-home script, idempotent, writing only inside the folder it is given. It refuses a path inside the real home. — done: `bench/demo/agent-ghosts.sh`, PR #58 and #59
- [x] ~~The recording script: the commands and pauses, so the owner can record in one take.~~ Dropped: the terminal GIF was rendered from real output without a recorder, and the owner recorded the app in one take without a script.
- [x] Cover image (SVG source plus PNG). — done: agent ghosts and their slime; the app icon was not used
- [x] The owner's screenshots, checked for private names. — done: the owner's recording was cut to leave out a private project name, and the user-data names are blurred

## Progress, 2026-11-01

- `bench/demo/agent-ghosts.sh` is in the code repository (PR #58). It refuses a folder inside the real home, and since 0.14.1 (PR #59) it refuses before creating anything.
- The terminal GIF is real output of the released `ddc` 0.12.1 on the demo home: four harnesses and one live session, from its record (0.12.1 lists record-backed sessions; the per-session rows came with 0.14.0); the cover (2000×840, the 1000×420 ratio), the animated ProjectStore loop, and a cut of the owner's screen recording with the user-data names blurred are in the post.

## Implementation Plan

## Acceptance Criteria

- [x] The demo-home script builds the tree twice in a row with the same result, and refuses `$HOME` and any path inside it (test or recorded run). — evidence: `bench/test_agent_ghosts.py` (a folder inside the home, relative, through a symlink, climbing with `..`, and `stop`: each exits 1 and creates nothing), PR #59; built twice in a row with the 0.14.1 script, the sorted listing of every file and its size (the live record, named by a pid, left out) hashed the same both times (`find home tmp -type f ! -path 'home/.claude/sessions/*' -exec stat -f '%N %z' {} + | sort | shasum -a 256`: 35 files)
- [x] `ddc disk agents` on the demo home shows all four harnesses and at least one live and one dead session (recorded output). — evidence: on the demo home, 0.12.1 (the GIF in the post) listed the four harnesses with one live session, and the 0.14.0 build added the per-session rows: one live, four not live
- [x] The cover image is 1000×420 and readable at 50 % size. — evidence: the post's cover, rendered at 2000×840 from an SVG; its labels checked by eye in the rendered PNG
- [x] Manual (owner), recorded in the owner-checks runbook: the screenshots show no project or session name the owner does not want public. — moved to the runbook "Owner checks", section "Launch demo assets (DDC-009)"

## Final Summary

**What changed.** The launch has its demo assets: `bench/demo/agent-ghosts.sh` builds a synthetic home with four harnesses and one live session and refuses any folder inside the real home before creating it (`bench/test_agent_ghosts.py`); a terminal GIF of real `ddc` output on that home; a cover of agent ghosts and their slime; a cut of the owner's screen recording of the app; and the ProjectStore loop, rendered from the site's own animation and published on YouTube in 4K.

**Checks.** The script's refusal tests run in `make bench-test`; the frames were checked for private names before publication, and the owner's own check is in the owner-checks runbook.

## Technical Notes

- `ddc disk agents` is never pointed at the owner's real home folder by a session. The owner's own run of 2026-10-31 supplies the real totals.
- Liveness reads a Claude Code record `sessions/<pid>.json` and checks only pid, uid and start time; there is no process-name check. So the script writes two records: one naming its own `sleep` child (live), and one naming a pid that has exited (stale). The demo then shows both a live and a stale session, which is the post's "live is not dead" claim.

## Dependencies

- DDC-003 story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees", for step 2 of the recording.

## Attachments

-

---

*Last updated: 2026-10-31*
