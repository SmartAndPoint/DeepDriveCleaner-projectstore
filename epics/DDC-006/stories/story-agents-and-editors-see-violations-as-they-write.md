---
type: story
id: "story-agents-and-editors-see-violations-as-they-write"
epic: "DDC-006"
title: "Agents and editors see violations as they write"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [agents, hooks, editor, tooling]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Agents and editors see violations as they write

| Field | Value |
|---|---|
| **Epic** | [DDC-006](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

A gate that fails at commit time leaves the agent to undo a large change. The cheapest moment to fix a 120-line function is the turn it was written. This story delivers the checker's findings where code is written:

1. **Claude Code.** A project PostToolUse hook after `Edit` or `Write` of a `.rs` file runs the prebuilt `ddc-shape --files <file>` (never `cargo run`, never clippy). The model sees any violation as feedback on its edit. PostToolUse cannot block, because the tool has already run, so the feedback is what makes the agent fix the code in the same turn.
2. **Editors.** rust-analyzer runs `scripts/ra-check.sh` through `rust-analyzer.check.overrideCommand` (one argv array). The script runs clippy with `--message-format=json --target-dir target/rust-analyzer`, then `ddc-shape --format cargo`, and prints both streams of JSON lines. Shape violations appear inline as diagnostics, with no separate language server. rust-analyzer gets its own target folder (`target/rust-analyzer`, passed by the script itself, since the docs do not say `cargo.targetDir` reaches an override command), so the editor and the gate never wait on one `target/` lock. The hook takes no lock at all.
3. **AGENTS.md.** One short paragraph: "Code shape is enforced by `make shape` and clippy. The limits and their reasons are in `docs/code-shape.md`. Fix violations; never add an exception without a reason." It points to the tool. It does not restate the rules.

## Decomposition

- [ ] `scripts/shape-hook.sh`:
  - sets `PATH=$HOME/.cargo/bin:$PATH`;
  - reads the hook's JSON input and filters `.rs` paths under `crates/` or `tools/`;
  - never builds: if `target/release/ddc-shape` is missing, or older than the newest file under `tools/ddc-shape/src`, it prints one line, `ddc-shape is not built or stale: run make shape`, and exits 0;
  - runs `ddc-shape --files`;
  - returns violations through the documented feedback channel (exit 2 with stderr, or JSON `decision: "block"` with a reason, whichever the current hooks documentation gives for PostToolUse).
- [ ] `.claude/settings.json`: a PostToolUse hook with matcher `Edit|Write|MultiEdit` that calls the script. Committed, so every session in the repository gets it (the epic's Open Question on committing it).
- [ ] Time: ≤ 1 s per edit on the reference machine, measured.
- [ ] rust-analyzer: `scripts/ra-check.sh` and the override setting, documented in `docs/code-shape.md` §Editor, with the note that saves get slower (clippy takes about 7–10 s warm here). Optionally a committed `.vscode/settings.json`, if the owner wants editor settings in the repository.
- [ ] The AGENTS.md paragraph, outside the projectstore block, plus `make hooks` from DDC-005 named next to it, so agent clones install the git hooks too.
- [ ] Codex and other agents: the same script, documented as a plain command (`make shape FILES=…`). `codex.json` stays local and uncommitted.

## Implementation Plan

## Acceptance Criteria

- [ ] In a Claude Code session in this repository, writing a function over the hard length limit returns feedback naming the function, its length and the limit, before the turn ends (evidence: transcript excerpt).
- [ ] A compliant edit produces no hook output and adds ≤ 1 s.
- [ ] Manual (owner): rust-analyzer in the owner's editor shows a `shape::` diagnostic inline on an over-long file.
- [ ] AGENTS.md has the pointer paragraph; `docs/code-shape.md` has the editor section.

## Final Summary

## Technical Notes

- Check the PostToolUse feedback contract against the current Claude Code hooks documentation when implementing. As of 2026-10-28, PostToolUse cannot block; exit 2 shows stderr to the model; JSON `decision: "block"` with a `reason` puts the reason next to the tool result.
- The hook never prompts, never needs the network, never runs `cargo`, and writes nothing.
- First-run cost for the owner: `make tools` now also builds `ddc-shape` (once, tens of seconds).

## Dependencies

- Story "ddc-shape: a syn-based checker for size, nesting, cohesion and boundaries".
- Story "Clippy at full strength: curated lints with reasons" (for the editor override).

## Attachments

-

---

*Last updated: 2026-10-28*
