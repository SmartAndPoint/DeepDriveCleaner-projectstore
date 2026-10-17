# DeepDriveCleaner-projectstore

The project's memory: decisions, specs, epics and stories as plain markdown, kept by projectstore.

This is the [ProjectStore](https://projectstore.dev) vault of [DeepDriveCleaner](https://github.com/SmartAndPoint/DeepDriveCleaner), a macOS tool that finds and safely cleans what development leaves on the disk. It is public on purpose, as a worked example of building software through ProjectStore. A research note came first, then two ADRs and an epic with stories. A fresh-context critic reviewed every artifact before code was written, a planner placed the work, and a reviewer checked each story's acceptance criteria against the code before the story was closed with evidence.

Where to start reading:

1. [Where a developer's disk space goes (macOS)](research/where-developer-disk-space-goes-macos.md): the research note behind the project.
2. [Rust core: library + CLI with JSON output](adr/rust-core-library-and-cli-with-json-output.md) and [Cleanup safety model: risk classes, plan and confirmation](adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md): the two decisions.
3. [DDC-001: Disk space cleaner CLI for developers](epics/DDC-001/epic.md): the epic and its stories, each closed with the tests and measurements that prove it.
4. [kanban.md](kanban.md) for the board and [code-map.md](code-map.md) for where each story lives in the code.

To work on the project, clone this repository next to the code repository and follow the Contributing section of the [code README](https://github.com/SmartAndPoint/DeepDriveCleaner#contributing).

## Folders

| Folder | Purpose |
|--------|---------|
| [ADRs](adr/README.md) | Architecture Decision Records — significant technical decisions with context, considered options, and consequences. |
| [Specs](specs/README.md) | Normative "how" per subsystem — ADR references, numbered behavioral contracts, acceptance additive to the covered stories. |
| [Epics](epics/README.md) | Large bodies of work broken into stories. Each epic gets its own subfolder with a `stories/` directory. |
| [Research](research/README.md) | Research notes — investigations, spikes, comparisons, and findings. |
| [Concepts](concepts/README.md) | Concept notes — definitions, mental models, and glossary entries. |
| [Meetings](meetings/README.md) | Meeting notes — date-prefixed records of discussions, decisions, and action items. |
| [Ops](ops/README.md) | Operational runbooks — step-by-step how-tos with verification and rollback. |
| [Diagrams](diagrams/README.md) | Architecture and flow diagrams — system context, sequence, and data-flow visuals. |

---

*`kanban.md`, `graph.md` and `code-map.md` are derived views: `reconcile` regenerates them from the files in these folders.*
