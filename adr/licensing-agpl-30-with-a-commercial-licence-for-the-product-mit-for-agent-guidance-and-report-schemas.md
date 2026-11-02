---
type: adr
id: "licensing-agpl-30-with-a-commercial-licence-for-the-product-mit-for-agent-guidance-and-report-schemas"
title: "Licensing: AGPL-3.0 with a commercial licence for the product, MIT for agent guidance and report schemas"
status: accepted
date: 2026-11-01
authors: ["Evgenii Konev"]
tags: [license, distribution, agents, contract]
external_refs: {}
supersedes: null
superseded_by: null
review_status: reviewed
reviewed_at: 2026-11-01
---

# Licensing: AGPL-3.0 with a commercial licence for the product, MIT for agent guidance and report schemas

| Field | Value |
|---|---|
| **Status** | accepted |
| **Date** | 2026-11-01 |
| **Authors** | Evgenii Konev |

---

## Context

Until the first release that carries this ADR, the whole code repository is under PolyForm Noncommercial 1.0.0 (`LICENSE.md`, `license` in `Cargo.toml`), with a contributor clause that lets SmartAndPoint relicense contributions under any terms (README, "Contributing"). The owner kept that licence on 2026-10-28 after weighing GPL-3.0 with dual licensing (research "Distribution through agents", the Homebrew section) and for the DEV launch (epic DDC-009, decision D1).

On 2026-11-01 the owner reopened the question and chose a softer licence: open source in the OSI sense, with a commercial licence for those who do not want its obligations. The owner's goal stays the same: anyone may read, use and change the code, and a company that wants to put it into a closed, paid product talks to SmartAndPoint first.

Two parts of the repository need a more permissive licence than the product, whatever the product's licence:

- **The agent guidance** (`agents/`: the Claude Code skill, `agents/AGENTS.snippet.md`, `agents/README.md`). It exists to be copied into other people's repositories and `~/.claude/skills/` (ADR "Agent footprint", §6). Under a copyleft licence, a company that copies it into its own repository would have to ask what that does to the repository; it contains no product logic, only how to run `ddc` and read its JSON.
- **The report schemas** (`docs/*.schema.json`). They are the published contract of the CLI's JSON (Rust core ADR). A tool that reads `ddc`'s output needs to vendor them or generate code from them; a schema describes an interface and carries no implementation.

There is no published way to ask for a commercial licence: the README names no contact.

**Who holds the copyright** (checked 2026-11-01): `git log origin/main` shows one author, the owner, on all 272 commits, and no `Co-authored-by` trailer; the only third-party files in the repository are the website's Heroicons (MIT) and the app's Sparkle, pulled by Swift Package Manager (MIT). So relicensing needs no outside consent.

Copyright protects code, not ideas or algorithms. Anyone may reimplement what `ddc` does from scratch; no licence changes that, and the project holds no patent.

## Decision

1. **The product is licensed under the GNU Affero General Public License, version 3 only** (SPDX `AGPL-3.0-only`), from 0.16.4, the first release that carries this change: the Rust crates, the CLI, the menu bar app, scripts, tests, benchmarks, and every file not named in 3. `LICENSE.md` holds the copyright line `Copyright 2026 SmartAndPoint` and the verbatim AGPL text, nothing else, so GitHub detects it. `license` in `Cargo.toml` and in `fuzz/Cargo.toml` (a separate workspace) becomes `AGPL-3.0-only`; the app's `NSHumanReadableCopyright` (the About panel) names the AGPL. No per-file SPDX headers: a file's licence follows its path.
2. **A commercial licence is available from SmartAndPoint** for anyone who wants to use, embed or distribute DeepDriveCleaner without the AGPL's obligations (for example in a closed-source product). The contact is **legal@smartandpoint.com**, named in the README's License section. Its terms are set case by case; this ADR does not fix them.
3. **MIT for two paths**, `Copyright 2026 SmartAndPoint`:
   - `agents/**` (the agent skill, the `AGENTS.md` snippet, their README);
   - `docs/*.schema.json` (every report schema, present and future).
   `LICENSE-MIT.md` at the repository root holds the copyright line and the verbatim MIT text. The list of MIT paths is one fenced block of globs in the README's License section, the single source the structure test reads. The skill's frontmatter carries `license: MIT (Copyright 2026 SmartAndPoint; text in LICENSE-MIT.md of the DeepDriveCleaner repository)`, and `AGENTS.snippet.md` carries the same line inside its markers, so a copy of either file alone keeps its notice. The schemas are generated from the Rust types' doc comments (`DDC_UPDATE_SCHEMA=1`); they are program output that SmartAndPoint licenses under MIT, and a contributor's doc comment reaches them through the contributor clause.
4. **Earlier releases keep their licence.** Every release before the first AGPL release was published under PolyForm Noncommercial 1.0.0, and that grant stays; SmartAndPoint does not additionally offer them under the AGPL. The first AGPL release's notes name the boundary.
5. **Contributions** keep the existing clause, which is what makes dual licensing possible: by contributing, a contributor lets SmartAndPoint use, modify and relicense the contribution under any terms, including commercial ones. A contribution to an AGPL path is offered under AGPL-3.0-only, one to an MIT path under MIT, each with that grant on top.
6. **Dependencies** stay permissive (`deny.toml`: MIT, Apache-2.0, Unicode-3.0). AGPL-3.0 can include them; the commercial licence needs every dependency to allow closed redistribution, which a copyleft dependency would not, so the allow-list does not widen to copyleft.
7. **The name and the icon.** The copyright in the icon files and the scripts that draw them is licensed like the rest of the product (a fork needs them to build). The trademark rights in the name and the icon are reserved, as the AGPL's §7(e) allows: `TRADEMARKS.md` cites it, and says a fork uses a different name and icon. `DISCLAIMER.md` is an additional disclaimer the AGPL's §7(a) allows, and points to its §15 and §16.
8. **The README's License section explains the boundary** in a short "Can I…?" list, true to the AGPL:
   - use it, personally or in a company: yes;
   - change it and share it: yes, under the AGPL, with the changed source for those you give it to, or who use a changed `ddc` over a network;
   - ship the unchanged `ddc` beside your own product and call it as a separate program: yes, keep its licence and point to its source; your product stays yours (an aggregate, AGPL §5);
   - link `ddc-core`, change `ddc`, or build your product on its code, and keep that closed: needs the commercial licence;
   - the skill and the schemas: MIT, anywhere;
   - the name and the icon: not without permission (TRADEMARKS.md).
9. **Release artifacts** ship `LICENSE.md` as today (`scripts/build-dist.sh` asserts the tarball's exact listing and `scripts/build-app.sh` puts it in the app; no MIT file ships, so both stay as they are). Each release's notes end with "Source: the tag vX.Y.Z; dependencies as pinned in `Cargo.lock` and `Package.resolved`", the clear directions next to the object code that the AGPL's §6(d) asks for. The Homebrew cask DSL has no licence stanza, so the cask template does not change.

## Rationale

1. The owner chose an OSI open-source licence over a noncommercial one (2026-11-01). AGPL-3.0 is the strongest common copyleft: whoever distributes a changed `ddc`, or runs a changed `ddc` for others over a network, publishes the change. A closed product cannot be built on its code without the commercial licence, which is the route the owner wants companies to take.
2. AGPL-3.0 is DFSG-compatible, so the licence no longer blocks a `homebrew/core` formula; the other conditions (built from source, notability, not a macOS app as main output) still apply, and a later story decides.
3. The skill and the schemas are how other tools and agents talk to `ddc`; copyleft on them would deter integrations and protect nothing.
4. `-only`, not `-or-later`: the terms stay the ones the owner chose; moving to a future version is a relicensing decision, which the owner can take alone while the contributor clause holds.
5. Naming a contact turns "copyleft" into "or ask us", which is the point of dual licensing.

## Alternatives Considered

### Alternative A: Keep PolyForm Noncommercial 1.0.0

**Pros**:
- Forbids every commercial use without a licence, including selling a product whose source is published.

**Cons**:
- Not open source; forbids a company's internal use; closes `homebrew/core`.

**Rejected because**: the owner chose a softer, open-source licence on 2026-11-01.

### Alternative B: Functional Source License 1.1, converting to MIT after two years

**Pros**:
- Allows any use except a competing product; each version becomes MIT after two years.

**Cons**:
- Not OSI open source until the conversion; every version becomes fully permissive after two years.

**Rejected because**: the owner chose AGPL-3.0 over it on 2026-11-01.

### Alternative C: MIT or Apache-2.0 for everything

**Pros**:
- The widest adoption.

**Cons**:
- Anyone may embed the product in a closed, paid product without asking.

**Rejected because**: it gives up the commercial route.

## Consequences

**Positive**:
- DeepDriveCleaner is open source in the OSI sense; the DEV post's and the website's "source-available" wording can become "open source".
- A company that wants a closed product has a route: the commercial licence.
- The skill and the schemas can be copied anywhere.

**Negative / trade-offs**:
- A company may sell a product built on `ddc` without asking, if it publishes that product's whole source under the AGPL; and it may ship the unchanged `ddc` beside a closed product and call it, as an aggregate. The commercial route covers closed code built on `ddc`, not arm's-length use. The owner accepts this (2026-11-01).
- Two licences in one repository: a file's licence follows its path, so a moved file can change licence. A test keeps the paths honest.
- PolyForm-era releases stay PolyForm; two licences exist side by side in the release history.

**What changes in code / process**:
- `LICENSE.md` (copyright line and verbatim AGPL), new `LICENSE-MIT.md`, `AGPL-3.0-only` in `Cargo.toml` and `fuzz/Cargo.toml`, `NSHumanReadableCopyright` in `app/DeepDriveCleaner/Info.plist`, the notice in the skill and the snippet, a line in `agents/README.md`, the README's License section (decisions 2, 3, 8) and its Disclaimer paragraph, `DISCLAIMER.md` (§7(a), §15, §16), `TRADEMARKS.md` (§7(e); "the licences cover the code"), the comments in `deny.toml` (its `[licenses.private] ignore = true` is what keeps the workspace's own AGPL out of the dependency allow-list check), the release-notes ending of decision 9.
- A structure test: (a) every tracked file under `agents/` and every `docs/*.schema.json` matches the README's MIT globs, (b) no other tracked file matches, (c) `Cargo.toml` and `fuzz/Cargo.toml` say `AGPL-3.0-only`, (d) `Info.plist` names the AGPL, (e) the README names `legal@smartandpoint.com` and links both licence files, (f) no tracked file outside `docs/releases/`, `website/` and the vault mentions `PolyForm`.
- The research note "Distribution through agents" and the runbook "Website publication and social preview operations" get a dated note that the licence changed; epic DDC-009's decision D1 is answered. Closed epics and stories that name PolyForm stay as history.
- Outside this repository, the owner's: the DEV post's licence paragraph; the website's licence wording and its generated `llms.txt` (another session's files under `website/`); GitHub's licence detection follows `LICENSE.md` on its own.

## Open Questions

- The copyright line names SmartAndPoint, while every commit is authored by the owner personally. If SmartAndPoint is the legal entity that holds the code's copyright (by assignment), the line stands and the entity signs commercial licences; if not, the line names the owner. The owner settles it.
- The vault repository has no licence file; it is "all rights reserved" by default. A separate decision.

## References

- Research "Distribution through agents", section on Homebrew and the licence (2026-10-28).
- Epic DDC-009 "Ghostbusters launch: the DEV challenge and a first audience", decision D1.
- ADR "Agent footprint: worktrees, harness data and bounded consent for agents", §6.
- ADR "Rust core: library + CLI with JSON output".
- GNU AGPL 3.0: https://www.gnu.org/licenses/agpl-3.0.html
