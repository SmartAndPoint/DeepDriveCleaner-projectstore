---

projectstore: derived
generated_at: 2026-10-18T09:50:05.787Z

---

# Code map

Epic ↔ code mapping, derived from frontmatter `code_refs` (source of truth).
Regenerate via `/projectstore:codemap`; edit refs via `/projectstore:codemap set`.

| Epic | Title | Status | code_refs |
|------|-------|--------|-----------|
| [[epics/DDC-001/epic\|DDC-001]] | Disk space cleaner CLI for developers | in-progress | `Cargo.toml`, `Makefile`, `rust-toolchain.toml`, `clippy.toml`, `rustfmt.toml`, `crates/ddc-core/`, `crates/ddc-cli/`, `docs/scan.schema.json`, `bench/` |
| [[epics/DDC-002/epic\|DDC-002]] | Menu bar app | planned | — |

## Story-level refs (files each story touched)

| Epic | Story | code_refs |
|------|-------|-----------|
| DDC-001 | ddc scan report | `crates/ddc-core/src/model.rs`, `crates/ddc-core/src/report/`, `crates/ddc-core/src/store.rs`, `crates/ddc-core/tests/report.rs`, `crates/ddc-cli/src/cli.rs`, `crates/ddc-cli/src/cmd/scan.rs`, `crates/ddc-cli/src/render.rs`, `crates/ddc-cli/src/progress.rs`, `crates/ddc-cli/src/size.rs`, `crates/ddc-cli/tests/scan_cli.rs`, `docs/scan.schema.json` |
| DDC-001 | Detection rule catalog | `crates/ddc-core/src/rules/`, `crates/ddc-core/src/class.rs`, `crates/ddc-core/tests/rules.rs`, `crates/ddc-core/tests/common/mod.rs`, `bench/make_baseline.py`, `bench/check_baseline.py` |
| DDC-001 | Safe cleanup: ddc clean | `crates/ddc-core/src/plan.rs`, `crates/ddc-core/src/exec/`, `crates/ddc-core/src/sys/procs.rs`, `crates/ddc-core/src/sys/fsops.rs`, `crates/ddc-core/src/rules/mod.rs`, `crates/ddc-core/src/rules/catalog.rs`, `crates/ddc-core/tests/clean_safety.rs`, `crates/ddc-cli/src/cmd/clean.rs`, `crates/ddc-cli/src/prompt.rs`, `crates/ddc-cli/tests/clean_cli.rs`, `bench/make_baseline.py` |
| DDC-001 | Scheduled background scan | `crates/ddc-cli/src/launchd.rs`, `crates/ddc-cli/src/notify.rs`, `crates/ddc-cli/src/cmd/background.rs`, `crates/ddc-cli/src/cmd/schedule.rs`, `crates/ddc-cli/src/cmd/scan.rs`, `crates/ddc-cli/src/cmd/mod.rs`, `crates/ddc-cli/src/cli.rs`, `crates/ddc-cli/src/main.rs`, `crates/ddc-cli/tests/background_cli.rs`, `crates/ddc-cli/tests/structure.rs`, `crates/ddc-core/src/report/mod.rs`, `crates/ddc-core/src/rules/visitor.rs` |
| DDC-001 | Workspace skeleton and parallel filesystem scanner | `Cargo.toml`, `rust-toolchain.toml`, `clippy.toml`, `rustfmt.toml`, `Makefile`, `crates/ddc-core/src/lib.rs`, `crates/ddc-core/src/env.rs`, `crates/ddc-core/src/sys/`, `crates/ddc-core/src/scan/`, `crates/ddc-core/tests/scanner.rs`, `crates/ddc-cli/src/main.rs`, `crates/ddc-cli/src/cmd/du.rs`, `crates/ddc-cli/src/cmd/diag.rs`, `crates/ddc-cli/tests/du_cli.rs`, `crates/ddc-cli/tests/diag_cli.rs`, `bench/` |
