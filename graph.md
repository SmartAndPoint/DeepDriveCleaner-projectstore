---

projectstore: derived
generated_at: 2026-10-18T06:36:56.400Z

---

# Vault link graph

Nodes (layout artifacts, keyed by vault-relative path) and typed edges,
derived from body links and frontmatter relations (source of truth).
Regenerate via `/projectstore:graph`; grep a path to see an artifact's
full typed neighborhood, both directions, in one call.

## Nodes

| Path | Title | Type | Status |
|------|-------|------|--------|
| adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md | Cleanup safety model: risk classes, plan and confirmation | adr | accepted |
| adr/rust-core-library-and-cli-with-json-output.md | Rust core: library + CLI with JSON output | adr | accepted |
| epics/DDC-001/epic.md | Disk space cleaner CLI for developers | epic | in-progress |
| epics/DDC-001/stories/story-ddc-scan-report.md | ddc scan report | story | done |
| epics/DDC-001/stories/story-detection-rule-catalog.md | Detection rule catalog | story | done |
| epics/DDC-001/stories/story-duplicates-and-apfs-private-size.md | Duplicates and APFS private size | story | planned |
| epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md | Homebrew tap and the v0.1.0 release | story | in-progress |
| epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md | Progress output while ddc clean runs | story | planned |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | Safe cleanup: ddc clean | story | done |
| epics/DDC-001/stories/story-scheduled-background-scan.md | Scheduled background scan | story | done |
| epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md | Workspace skeleton and parallel filesystem scanner | story | done |
| research/where-developer-disk-space-goes-macos.md | Where a developer's disk space goes (macOS) | research | final |

## Edges

| From | Kind | To |
|------|------|----|
| adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md | wikilink | research/where-developer-disk-space-goes-macos.md |
| adr/rust-core-library-and-cli-with-json-output.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| adr/rust-core-library-and-cli-with-json-output.md | wikilink | research/where-developer-disk-space-goes-macos.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-ddc-scan-report.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-detection-rule-catalog.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-duplicates-and-apfs-private-size.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-scheduled-background-scan.md |
| epics/DDC-001/epic.md | epic-contains | epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-ddc-scan-report.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-detection-rule-catalog.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-duplicates-and-apfs-private-size.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-scheduled-background-scan.md |
| epics/DDC-001/epic.md | mdlink | epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md |
| epics/DDC-001/epic.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/epic.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| epics/DDC-001/epic.md | wikilink | research/where-developer-disk-space-goes-macos.md |
| epics/DDC-001/stories/story-ddc-scan-report.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-ddc-scan-report.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/stories/story-ddc-scan-report.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| epics/DDC-001/stories/story-ddc-scan-report.md | wikilink | epics/DDC-001/stories/story-detection-rule-catalog.md |
| epics/DDC-001/stories/story-detection-rule-catalog.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-detection-rule-catalog.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/stories/story-detection-rule-catalog.md | wikilink | epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md |
| epics/DDC-001/stories/story-detection-rule-catalog.md | wikilink | epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md |
| epics/DDC-001/stories/story-detection-rule-catalog.md | wikilink | research/where-developer-disk-space-goes-macos.md |
| epics/DDC-001/stories/story-duplicates-and-apfs-private-size.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-duplicates-and-apfs-private-size.md | wikilink | epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md |
| epics/DDC-001/stories/story-duplicates-and-apfs-private-size.md | wikilink | research/where-developer-disk-space-goes-macos.md |
| epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md | wikilink | epics/DDC-001/stories/story-scheduled-background-scan.md |
| epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md | wikilink | epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | wikilink | epics/DDC-001/stories/story-ddc-scan-report.md |
| epics/DDC-001/stories/story-scheduled-background-scan.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-scheduled-background-scan.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/stories/story-scheduled-background-scan.md | wikilink | epics/DDC-001/stories/story-ddc-scan-report.md |
| epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
