---

projectstore: derived
generated_at: 2026-10-22T04:46:27.591Z

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
| adr/menu-bar-app-swift-shell-over-the-ddc-cli.md | Menu bar app: Swift shell over the ddc CLI | adr | proposed |
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
| epics/DDC-002/epic.md | Menu bar app | epic | in-progress |
| epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md | App skeleton and build pipeline | story | in-progress |
| epics/DDC-002/stories/story-automatic-cleaning-of-download-caches-after-a-scheduled-scan.md | Automatic cleaning of download caches after a scheduled scan | story | planned |
| epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md | Background scan and notifications in the app | story | planned |
| epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md | CLI: clean an explicit item list | story | done |
| epics/DDC-002/stories/story-cli-progress-events.md | CLI: progress events | story | planned |
| epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md | CLI: shared settings, types and the notify decision | story | done |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | Distribution: the 0.2.0 cask with the app | story | in-progress |
| epics/DDC-002/stories/story-faster-clean-re-check-only-what-the-selection-needs.md | Faster clean: re-check only what the selection needs | story | planned |
| epics/DDC-002/stories/story-hig-pass-keyboard-colour-empty-states-and-menus.md | HIG pass: keyboard, colour, empty states and menus | story | in-progress |
| epics/DDC-002/stories/story-menu-bar-popover.md | Menu bar popover | story | in-progress |
| epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md | Release 0.2.1: the Review window in the Dock, and skip what a running process holds open | story | in-progress |
| epics/DDC-002/stories/story-review-and-clean-window.md | Review and clean window | story | in-progress |
| epics/DDC-002/stories/story-review-window-clean-only-the-selection-in-this-group.md | Review window: clean only the selection in this group | story | in-progress |
| epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md | Review window: sorted sidebar, Select all and folder groups | story | in-progress |
| epics/DDC-002/stories/story-settings-folders-types-notifications.md | Settings: folders, types, notifications | story | planned |
| epics/DDC-002/stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md | Signing team: parameterise TEAM_ID, then switch to SmartAndPoint | story | planned |
| epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash.md | User data: pick your own files for the Trash | story | in-progress |
| research/where-developer-disk-space-goes-macos.md | Where a developer's disk space goes (macOS) | research | final |

## Edges

| From | Kind | To |
|------|------|----|
| adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md | wikilink | research/where-developer-disk-space-goes-macos.md |
| adr/menu-bar-app-swift-shell-over-the-ddc-cli.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| adr/menu-bar-app-swift-shell-over-the-ddc-cli.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| adr/menu-bar-app-swift-shell-over-the-ddc-cli.md | wikilink | epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md |
| adr/menu-bar-app-swift-shell-over-the-ddc-cli.md | wikilink | epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md |
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
| epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md | wikilink | epics/DDC-002/stories/story-cli-progress-events.md |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/stories/story-safe-cleanup-ddc-clean.md | wikilink | epics/DDC-001/stories/story-ddc-scan-report.md |
| epics/DDC-001/stories/story-scheduled-background-scan.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-scheduled-background-scan.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-001/stories/story-scheduled-background-scan.md | wikilink | epics/DDC-001/stories/story-ddc-scan-report.md |
| epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md | mdlink | epics/DDC-001/epic.md |
| epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner.md | wikilink | adr/rust-core-library-and-cli-with-json-output.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-automatic-cleaning-of-download-caches-after-a-scheduled-scan.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-cli-progress-events.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-faster-clean-re-check-only-what-the-selection-needs.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-hig-pass-keyboard-colour-empty-states-and-menus.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-menu-bar-popover.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-review-and-clean-window.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-review-window-clean-only-the-selection-in-this-group.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-settings-folders-types-notifications.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md |
| epics/DDC-002/epic.md | epic-contains | epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-automatic-cleaning-of-download-caches-after-a-scheduled-scan.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-cli-progress-events.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-faster-clean-re-check-only-what-the-selection-needs.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-hig-pass-keyboard-colour-empty-states-and-menus.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-menu-bar-popover.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-review-and-clean-window.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-review-window-clean-only-the-selection-in-this-group.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-settings-folders-types-notifications.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md |
| epics/DDC-002/epic.md | mdlink | epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash.md |
| epics/DDC-002/epic.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| epics/DDC-002/epic.md | wikilink | epics/DDC-001/epic.md |
| epics/DDC-002/epic.md | wikilink | epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md |
| epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md | wikilink | epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release.md |
| epics/DDC-002/stories/story-automatic-cleaning-of-download-caches-after-a-scheduled-scan.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md | wikilink | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md | wikilink | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md | wikilink | epics/DDC-002/stories/story-menu-bar-popover.md |
| epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md | wikilink | epics/DDC-002/stories/story-settings-folders-types-notifications.md |
| epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md | wikilink | adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md |
| epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| epics/DDC-002/stories/story-cli-progress-events.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-cli-progress-events.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| epics/DDC-002/stories/story-cli-progress-events.md | wikilink | epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs.md |
| epics/DDC-002/stories/story-cli-progress-events.md | wikilink | epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md |
| epics/DDC-002/stories/story-cli-progress-events.md | wikilink | epics/DDC-002/stories/story-review-and-clean-window.md |
| epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md | wikilink | epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-menu-bar-popover.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-review-and-clean-window.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-settings-folders-types-notifications.md |
| epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md | wikilink | epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash.md |
| epics/DDC-002/stories/story-faster-clean-re-check-only-what-the-selection-needs.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-hig-pass-keyboard-colour-empty-states-and-menus.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-menu-bar-popover.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-menu-bar-popover.md | wikilink | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/stories/story-menu-bar-popover.md | wikilink | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md | wikilink | epics/DDC-002/stories/story-cli-progress-events.md |
| epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md | wikilink | epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md |
| epics/DDC-002/stories/story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open.md | wikilink | epics/DDC-002/stories/story-review-and-clean-window.md |
| epics/DDC-002/stories/story-review-and-clean-window.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-review-and-clean-window.md | wikilink | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/stories/story-review-and-clean-window.md | wikilink | epics/DDC-002/stories/story-cli-clean-an-explicit-item-list.md |
| epics/DDC-002/stories/story-review-and-clean-window.md | wikilink | epics/DDC-002/stories/story-cli-progress-events.md |
| epics/DDC-002/stories/story-review-and-clean-window.md | wikilink | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/stories/story-review-and-clean-window.md | wikilink | epics/DDC-002/stories/story-menu-bar-popover.md |
| epics/DDC-002/stories/story-review-window-clean-only-the-selection-in-this-group.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-settings-folders-types-notifications.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-settings-folders-types-notifications.md | wikilink | epics/DDC-002/stories/story-app-skeleton-and-build-pipeline.md |
| epics/DDC-002/stories/story-settings-folders-types-notifications.md | wikilink | epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app.md |
| epics/DDC-002/stories/story-settings-folders-types-notifications.md | wikilink | epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision.md |
| epics/DDC-002/stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md | mdlink | epics/DDC-002/epic.md |
| epics/DDC-002/stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md | wikilink | adr/menu-bar-app-swift-shell-over-the-ddc-cli.md |
| epics/DDC-002/stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md | wikilink | epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app.md |
| epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash.md | mdlink | epics/DDC-002/epic.md |
