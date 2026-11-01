---
type: story
id: "story-cli-invocation-cases-a-table-of-ddc-calls-and-the-suggested-commands-they-print"
epic: "DDC-005"
title: "CLI invocation cases: a table of ddc calls and the suggested commands they print"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [cli, tests, gate]
code_refs: []
specs: []
started_at: "2026-11-01T12:08:42.744Z"
closed_at: "2026-11-01T12:52:12.821Z"
plan_updated_at: "2026-11-01T12:08:42.744Z"
---

# CLI invocation cases: a table of ddc calls and the suggested commands they print

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner ran `ddc scan .` in `~/Projects`. The scan was right, but its summary said "run `ddc clean`", and `ddc clean` with no path reviews the home (or the folders `settings.json` names), not `~/Projects`. Following the advice would have planned a clean of a different scope. 0.14.2 fixed that one line with a unit test of `render.rs`. Nothing would have caught it: the CLI tests check each flag, mostly through the JSON report, and almost never the text a person reads or the commands it tells them to run. The same pattern is still elsewhere: a background scan's notification says "Run `ddc clean` to review." whatever paths the scan was given.

This story adds one table of `ddc` calls, `crates/ddc-cli/tests/cases.json`, run against a synthetic home. Each case names a call (its arguments and the folder it runs in) and what it must do: exit code, the scanned roots, what it finds, text that must and must not appear (substrings or regexes, since sizes and durations vary). Three rules hold for every case, so a new case is one entry of JSON and no Rust.

1. **Follow-through.** Every suggestion in the output is a backtick span that starts with `ddc ` (the renderer prints suggestions only that way; placeholders such as `<path>` or `ack|snooze` mark a span as never run). The runner splits it with POSIX quoting, expands `~` to the case's home and runs the built binary directly, never through a shell. Each suggestion passes the safety allowlist (rule 3) or is refused by name. A suggested `clean` runs with `--dry-run`. Its scope is read from `<data-dir>/last-scan.json`, which the dry run writes before the plan, and compared with the case's roots as canonical, deduplicated sets. A suggestion that is neither run nor listed in the file's `not_run` patterns with a reason fails the case.
2. **Coverage.** Every subcommand and every argument of the CLI, hidden ones included, appears in at least one case or in the file's `uncovered` list with a reason. `--help` hides `du`, `diag`, `notify`, `scan --background` and the `--system-root` flags, so the rule walks clap's command tree instead: a `#[cfg(test)]` test in `src/cli.rs` reads `Cli::command()` and `include_str!("../tests/cases.json")`.
3. **Safety allowlist.** The runner runs only these, and refuses anything else by name, in cases and in follow-through alike:
   - `scan`, `rules`, `du`, `diag`, and a bare `--help` or `--version`. `scan --background` runs only together with `--no-notify` or `--notify json`, because without them it can show a real notification;
   - `clean` only with `--dry-run`, never with `--yes`, `--items` or `--permanent`;
   - `keep` and `notify` only with `--data-dir` inside the temporary root;
   - every `disk` command only with `--system-root` inside the temporary root, and `disk free` never, because its dry run runs each tool's own dry run (brew, conda);
   - `schedule status`, which with a temporary home stops at "Not installed" and reaches no launchctl.
   `schedule install` and `schedule uninstall` are in `uncovered`. `uninstall` boots out the real `gui/<uid>` agent whatever `HOME` is. `disk free` and all its flags are in `uncovered` too. Every path argument must lie inside the temporary root, not only inside its home, so a folder beside the home can exercise the outside-home suggestion. `HOME`, `USERPROFILE`, `APPDATA` and `LOCALAPPDATA` point inside it, and so does `DDC_DATA_DIR`, the data folder's fallback when the flag is absent. `--data-dir` is passed too, where the command has the flag; `du`, `rules`, `schedule` and `diag` do not have it. Coverage leaves out clap's built-in `help` and `version` arguments.

The owner asked for this on every build and above all before every commit (2026-11-01). This changes part of the owner's 2026-10-31 tiering, under which pre-commit only checks (fmt and clippy) and builds nothing. The pre-commit hook now also runs the cases when a staged path can change the CLI's behaviour. Pre-push, CI's Gate and the native legs run them with the rest of nextest.

"Every build" here means the hook tiers and CI's Gate. The release smoke in `release.yml` stays as it is.

## Platforms

A case states its expectations for macOS. A platform overlay (`expect.linux`, `expect.windows`) changes only what differs there. Capabilities differ in two ways:

- **Exit.** `Schedule`, `WholeDisk`, `CodingAgents` and `Homebrew` refuse with exit 69 where they are missing (`gate.rs`). The runner derives that from `ddc_core::platform::CAPABILITIES`: a case whose command needs one of them expects 69 and the refusal there, with no overlay needed. This deliberately asserts the refusal rather than skipping the case, as `unported_cli.rs` already does. So no case needs a line in `.config/native-skips`.
- **Output.** The other capabilities change what a successful run prints, not its exit. Examples: no Full Disk Access hint off macOS, a different rule catalog, no `private_bytes`, and no notifier on Linux. Overlays carry those differences. The fixture uses rules that every catalog lists, found by their paths on this platform. The plan chooses where it reads them from: `ddc_loadgen::fixtures::catalog`, ddc-cli's dev-dependency that the seam ADR's §9 amendment names for per-platform fixtures, or the `rule_path` helpers moved there from `ddc-core/tests/common`.

Windows is not supported yet (`SUPPORTED = false`). Every case but `rules`, `--help` and `--version` therefore expects the unsupported exit there, and that is written down, not skipped.

## Decomposition

- [ ] The case format: a shared fixture (folders and files relative to the temporary root, with the home inside it), the cases with platform overlays, `not_run` and `uncovered`. Unknown keys are an error (`deny_unknown_fields`).
- [ ] The runner `crates/ddc-cli/tests/cases_cli.rs`. It runs the cases in a few `#[test]`s by group, so no single test nears nextest's 120 s limit under gate load. Each case gets a fresh temporary root. Each `#[test]` gathers every failing case before it fails, naming each case with its call, stdout and stderr.
- [ ] The allowlist (rule 3), with a test that it refuses each forbidden command and flag.
- [ ] Follow-through (rule 1): extraction, POSIX splitting (unit-tested on the quoting vectors of `render.rs`'s tests), and the scope comparison. The comparator also gets a unit test on canned output: plain `ddc clean` after a scan of own roots fails, `ddc clean ~/Projects` passes.
- [ ] Coverage (rule 2) in `src/cli.rs`, with a unit test of the rule on a made-up clap `Command`.
- [ ] The background run writes the would-be notification text to its log whenever the decision is to notify, whether or not a notifier exists. That text names the scanned paths, the way the summary does. The Linux leg and macOS both check that line in the case. The app's own notification text (`NotificationText.swift`) is out of scope.
- [ ] Seed cases, at least:
  - the owner's call: `scan .` in `~/Projects` suggests `ddc clean ~/Projects`;
  - `scan .` in a project subfolder;
  - `scan` with no path;
  - two roots, and nested roots (`scan a a/b`; the walker drops the nested one);
  - a root with a space and one with a quote;
  - a root outside the home but inside the temporary root;
  - the settings' roots given by path;
  - `--skip`, and `--exclude` including one that drops a settings exclude;
  - a missing path;
  - `scan --json`, checked against its schema through `tests/contract`;
  - `scan --progress json`;
  - `scan --all`;
  - `clean --dry-run .`;
  - `du .`;
  - `rules`;
  - `disk` with `--system-root`;
  - `scan --background --no-notify --threshold 1K` with paths and `notify_low_disk: false` in the fixture's settings, whose logged notification text is checked;
  - `keep remove`, with the dev and inode of the fixture's `.ddc-keep` folder, which the runner reads itself (the case format has a placeholder for them);
  - `schedule status`, whose suggestion `ddc schedule install` is a `not_run` entry;
  - `diag`, checked by exit code only.
- [ ] `make cases`, which is `$(CARGO) nextest run -p ddc-cli --test cases_cli $(FEATURES) --locked`, so pre-push's `nextest-unit` reuses its build. Then a separate query `scripts/gate-scope.sh staged-cases` answering `yes` or `no`, tested in `scripts/tests/test_gate_scope.py`. `staged` keeps its one-word `rust|skip` contract. Then the pre-commit hook, which fails closed (anything but `no` runs `make cases`), `AGENTS.md` "The gate" and the hook's header comment.

## Implementation Plan

From the planner's placement plan (2026-11-01), against code at v0.14.4.

1. `regex` as a workspace dev-dependency of ddc-cli (already in Cargo.lock through jsonschema).
2. Background text: `of_settings_roots` moves from `cmd/scan.rs` to `cmd/mod.rs` as `own_roots`; `render::clean_command` becomes `pub`; `notify::message(d, clean)` prints the given command; `cmd/background.rs` logs `notification: <text>` to `<data>/logs/background.log` whenever the decision is to notify, notifier or not, with a unit test beside its siblings.
3. `crates/ddc-cli/tests/cases.json`: `fixture` (home, dirs, files, settings), `data_dir_commands`, `cases` (name, group, cwd, args, settings, expect with `linux`/`windows` overlays), `not_run`, `uncovered`; placeholders `{root}`, `{home}`, `{data}`, `{dev:REL}`, `{ino:REL}`, `{rule:ID}` (from `ddc_loadgen::fixtures::catalog`).
4. `crates/ddc-cli/tests/cases_cli.rs`, one file with inline modules (format, place, split, allow, follow, run). One `#[test]` per group, failures collected. Refusals derived from `SUPPORTED`/`CAPABILITIES` with the core's own messages. Unit tests: the parse, the splitter on `render.rs`'s vectors, each allowlist refusal, the comparator on canned text.
5. Coverage in `src/cli.rs`'s `#[cfg(test)]`: walk `Cli::command()` with hidden args, a call covers what clap matches from the command line; the rule tested on a made-up `Command`; stale `uncovered` names and `data_dir_commands` checked against clap.
6. The revert check by hand (`render.rs` printing plain `ddc clean`), recorded at close.
7. `make cases`; `scripts/gate-scope.sh staged-cases` (`yes|no`, fails to `yes`) with tests in `test_gate_scope.py`; the pre-commit hook runs `make cases` after lint unless the answer is `no`; `AGENTS.md` "The gate".
8. Linux leg on an aarch64 machine and the Windows leg natively; CI for x86_64 Linux.
9. Timing in four states, version bump, release notes, reviewer.

`diag` and a bare `--help`/`--version` join rule 3's allowlist: `diag` only reads the I/O policy, limits, statfs and the process table. `clean --json` and `clean --local-snapshots` need `--items`, which the allowlist refuses, so they are in `uncovered`. A follow-up worth its own story: the suggested `ddc clean` after `scan --skip X` drops the `--skip`; the comparator checks roots only, as this story says.

## Acceptance Criteria

- [x] `crates/ddc-cli/tests/cases.json` and `crates/ddc-cli/tests/cases_cli.rs` exist, with the seed cases. They pass on macOS and on the Gate's Linux and Windows legs. On Linux and Windows, a case whose command needs a refusing capability passes by its exit 69, and an output difference passes by its overlay. No case is skipped and no native-skips line is added. — evidence: 31 cases; `cargo nextest run -p ddc-cli --test cases_cli` 10/10 on macOS; the Linux leg on an aarch64 machine, 743 tests passed with the listed skips; PR #64's `check-x86_64-unknown-linux-gnu` and `check-x86_64-pc-windows-msvc` green; refusals asserted in `cases_cli.rs` `refusal()`. No case skips. The one line added to `.config/native-skips/windows.txt` is for the new background.rs unit test, which skips on Windows like its siblings, not for a case. No case needs an overlay yet; the merge is tested by `an_overlay_replaces_only_the_keys_it_names_on_its_platform`.
- [x] Follow-through: every suggestion in every case's output is run under the allowlist or matched by a `not_run` pattern, or the case fails. The comparator's unit test shows plain `ddc clean` failing after a scan of own roots. Once, by hand, reverting `clean_command` makes the owner's case fail; the Final Summary records that run. — evidence: `cases_cli.rs` `follow::follow`; test `plain_ddc_clean_after_a_scan_of_own_roots_fails_the_comparison`; the revert run is in the Final Summary.
- [x] The allowlist refuses `scan --background` without `--no-notify` or `--notify json`, `clean` without `--dry-run`, `--yes`, `--items`, `--permanent`, `schedule install|uninstall`, `disk free`, a `disk` command without `--system-root`, and any path outside the temporary root. A test shows each refusal. — evidence: test `the_allowlist_refuses_each_forbidden_call`, each row checked with the reason it must give.
- [x] Coverage: the test in `src/cli.rs` fails when a subcommand or an argument of `Cli::command()`, hidden ones included, is in no case and not in `uncovered`. A unit test on a made-up `Command` shows it failing. — evidence: `cli::tests::every_command_and_argument_is_in_a_case_or_uncovered`, `the_rule_names_what_no_call_covers`, plus `every_uncovered_name_is_a_real_command_or_argument` and `data_dir_commands_are_the_commands_with_data_dir`.
- [x] The background scan's logged notification text names the scanned paths, and a case checks it on macOS and Linux. — evidence: `cmd/background.rs` logs `notification: …` whenever the decision is to notify; unit test `the_notification_is_logged_with_the_clean_of_the_scanned_paths`; cases "a background scan of a folder logs the notification with the clean of that folder" and "a background scan of the home logs plain ddc clean", green on macOS and on the aarch64 Linux run.
- [x] `make cases` runs the cases alone. The pre-commit hook runs it after `make lint` unless `scripts/gate-scope.sh staged-cases` answers `no`. It answers `yes` when a staged path is under `crates/` or is `Cargo.toml` or `Cargo.lock`, and its tests are in `test_gate_scope.py`. `staged` still answers only `rust` or `skip`. Like lint, it checks the working tree. The hook comment and `AGENTS.md` say so. The Final Summary records the time it adds in four states: cold, warm with no change, after a CLI change, and after a core change. — evidence: Makefile `cases`; `.githooks/pre-commit`; class `StagedCases` in `scripts/tests/test_gate_scope.py`, 49 tests OK, `test_staged_keeps_its_one_word`; AGENTS.md "The gate"; the hook run by hand ran 10/10 cases; timings in the Final Summary.
- [x] `make gate-fast`, `make gate` and the native legs run the cases with the rest of nextest, outside the `load` group. — evidence: `make gate-fast` 641 passed with `cases_cli` in it (pre-push of PR #64); the `load` filter of `.config/nextest.toml` does not name it; the native legs run `nextest --workspace` (743 on Linux).
- [x] Version bump and release notes in the same PR (`make bump`). — evidence: 0.14.6, `docs/releases/0.14.6.md`, PR #64.

## Final Summary

**What changed.**
- `crates/ddc-cli/tests/cases.json` holds 31 calls of `ddc`, with a shared fixture (home, folders beside it, settings), per-platform overlays, `not_run` and `uncovered`.
- `cases_cli.rs` runs each call on a fresh canonical temporary root, then runs the advice its output and logs give. That covers the background log as well as stdout and stderr.
- The allowlist reads arguments the way clap does and fails closed: a word it cannot classify is a path, and must lie inside the root.
- `src/cli.rs` checks coverage by walking clap's tree, hidden arguments included.
- The background notification now prints `render::clean_command` of the scan's own roots, decided by `own_roots`, which moved from `cmd/scan.rs` to `cmd/mod.rs`. It is logged whenever the decision is to notify, so Linux checks it too.
- `make cases`, `scripts/gate-scope.sh staged-cases` and the pre-commit hook run the table before every commit that touches `crates/` or `Cargo.*`.

**Why.** The owner's `ddc scan .` in `~/Projects` suggested a plain `ddc clean`, which reviews the home. Nothing exercised the printed advice end to end (the owner, 2026-11-01).

**Tests executed.**
- The revert run: `render.rs` `report()` changed to `clean_command(&[], home)`, then `cargo nextest run -p ddc-cli --test cases_cli -E 'test(roots_cases) | test(scan_cases)'`. Both failed, 5 cases each. Each failed by its expected line (`stdout lacks "`ddc clean ~/Projects`"`) and by follow-through (`` `ddc clean` covers {…/home}, the case scanned {…/home/Projects} ``). The line was then restored.
- `make cases` timing on the developer machine:
  - cold (a fresh target folder): 28 s;
  - warm with no change: 2 s;
  - after a CLI change: 3 s;
  - after a core change: 9 s.
- `make gate-fast`: 641 passed.
- The Linux leg on an aarch64 machine: 743 passed, native-skips as listed.
- PR #64's Gate:
  - the Windows leg first failed only on the unlisted skip of the new background unit test, then passed;
  - the cases themselves passed there, by the refusal.

**Risks and follow-ups.**
- `core.hooksPath` is shared by every worktree of the clone and points at the main checkout's `.githooks`. A worktree runs the hook of whatever commit that checkout is on, and gets the cases only after that checkout pulls this change.
- The suggested `ddc clean` after `scan --skip X` drops the `--skip`. The comparator checks roots only, as this story says, so the cases do not catch it. That is worth a story of its own.
- The text output of `disk` and `disk agents` is never followed through, because both cases use `--json`. Neither prints a `ddc …` suggestion today.
- The Windows machine was unreachable over ssh, so Windows was checked by CI only.

## Technical Notes

- The runner executes text parsed from program output. It is safe because of the allowlist, not because of the expectation that a dangerous command would never be suggested.
- `clean --dry-run` takes the exclusive lock and rewrites `last-scan.json`. A case checks its own output first, then runs the follow-throughs in order.
- On macOS a temporary folder under `/var/folders` resolves through `/private`, so roots are compared canonicalised on both sides.
- On Windows, `tilde()` prints `\` and `quoted()` puts any path with `\` in single quotes. That is a wrong suggestion for cmd and PowerShell, and the cases will show it when Windows is supported. Until then, Windows cases expect the unsupported exit.
- The goldens record whole outputs for fixed load profiles. The cases check chosen text for chosen calls and run the advice. They do not replace each other.
- The cases test binary is not built by `make unported-check`, which names only its own test binaries.

## Dependencies

- Story "Mandatory gate before every commit" (DDC-005): the hooks this extends.
- Story "Coverage floor and property tests for the safety invariants" (DDC-005): `tests/contract` checks the JSON output against its schema. Cases with `--json` reuse it.

## Attachments

- Code PR #60 (0.14.2): the summary's suggested clean names the scanned paths. That fix is the first regression these cases hold.

---

*Last updated: 2026-11-01*
