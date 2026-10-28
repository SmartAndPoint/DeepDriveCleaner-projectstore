---
type: research
slug: "distribution-through-agents"
title: "Distribution through agents"
status: final
date: 2026-10-28
authors: ["Evgenii Konev"]
tags: [distribution, agents, skills, mcp, website, security]
review_status: reviewed
reviewed_at: 2026-10-28
---

# Distribution through agents

## Question

On 2026-10-28 the owner asked whether DeepDriveCleaner could reach users through coding agents instead of search engines, for example through OpenClaw, agent skill registries, or an agents-only social network.

This note answers four questions:
- How does a tool reach an agent and its user in practice?
- Which channels exist today, and what does each one require?
- What does DeepDriveCleaner have that an agent channel rewards, and what does it risk?
- What should be built, and in what order?

On the same day the owner bought three domains: `deepdrivecleaner.app`, `deepdrivecleaner.com` and `ddc.sh`. §5 says what each can do.

## TL;DR

Agents rarely discover and install software on their own initiative. A tool reaches them in three ways:
- a person installs a skill or plugin from a registry, often because the agent suggested it;
- a model names the tool when a person asks a question such as "why is System Data 300 GB";
- an agent hits a full disk in the middle of a task and looks for a fix.

The channels worth the work are:
1. an official Agent Skills skill (`SKILL.md`);
2. publishing it, as the owner, in Claude Code plugin marketplaces and on ClawHub;
3. pages that answer people's disk-space questions plainly, on the product site.

An MCP server, and with it the MCP Registry, waits until users ask for it. Agents-only social networks such as Moltbook are not a channel: its audience is curious people, and promotional posts from bots would read as spam.

DeepDriveCleaner could have one real advantage: agents themselves create a large share of a developer's waste. On the reference machine, one `$TMPDIR` held 790,104 test folders left behind by coding agents' runs, and agents' scratch folders in `/private/tmp` held 10.5 GB. See [[research/what-fills-system-data-on-a-developers-mac|What fills System Data on a developer's Mac]]. Today `ddc` does not find that waste: it lies outside the default scan roots, and no rule matches the test folders themselves. That is a promise for the agent-footprint work, not a property of the product yet.

Its main risk in this channel is also its main argument. A tool that deletes files on an agent's request is a target for prompt injection and for impostor skills. DeepDriveCleaner's safety model answers much of that:
- risk classes;
- an explicit item list;
- protected paths;
- the Trash;
- a log of every action.

It does not answer an agent that drives the CLI directly. Without a terminal, `clean --items` already accepts `review` items, permanent deletion and the user's own files. The app's confirmation sheet, which is the mitigation for those, is absent when an agent calls the CLI (§4). The owner settled two questions on 2026-10-28 (see the Conclusion):
- by default an agent cannot confirm a clean; a person can allow the `safe` class in Settings, at their own risk;
- the session-end hook asks the person and removes only what they approve.

## Method

Desk research on 2026-10-28 with web searches. Every source is listed in References. The figures about agent debris come from the measurements in "What fills System Data on a developer's Mac".

The sources are uneven:
- **Primary:** the specifications and official documentation (agentskills.io, the MCP Registry documentation, Claude Code's plugin documentation, OpenClaw's own post on its VirusTotal partnership). These are preferred.
- **Independent reporting:** Palo Alto Networks Unit 42, CSO Online, The Hacker News, Antiy CERT, Wiz and NBC News.
- **Vendor blogs and trackers:** where the only source is a vendor's blog or tracker, the note says so. Counts of skills, platforms and malicious uploads differ between sources, and the note gives ranges.

Nothing was published or registered, and no account was created.

## Findings

### 1. How a tool reaches an agent

**A person decides the install, even when an agent suggested it.** A coding agent works with the tools it is given:
- what is on the `PATH`;
- the skills and plugins its user installed;
- the MCP servers its user configured.

Installing a new tool is usually a permission prompt that a person approves. So the target of distribution is still a person, reached at the moment their agent recommends something.

**A model recommends what it has seen named in answers.** When a person asks an agent "what is System Data and how do I free it", the agent answers from its training data and from a web search. A tool appears in that answer when public pages name it as the answer to that exact question. That replaces SEO with being named for the problem. Search-engine rank still feeds the agent's web search.

**An agent that hits `ENOSPC` looks for a fix within its task.** It will try `du`, then known cache commands (`npm cache clean`, `brew cleanup`, `docker system prune`). It installs a new tool only when the skill is already present, or when its user approves the install.

### 2. The channels

| Channel | What it is | What publishing takes | Notes |
|---|---|---|---|
| **Agent Skills (`SKILL.md`)** | An open format for packaging instructions and scripts that an agent loads when the task matches. Anthropic released it as an open standard on 2025-12-18 and donated it to the Agentic AI Foundation under the Linux Foundation on 2026-10-22; agentskills.io hosts the specification. | A folder with `SKILL.md` (name, description, instructions) and optional scripts. | Read by Claude Code and, per third-party summaries, by OpenAI Codex, GitHub Copilot, Gemini CLI, Goose, OpenHands, OpenCode and others. Published counts of tools differ widely and come from interested parties. One skill reaches all of them. |
| **Claude Code plugin marketplaces** | A plugin bundles skills, commands, hooks and MCP servers. A marketplace is a git repository with a catalog; users add it with `/plugin marketplace add`. | Our own marketplace: a public repository. Anthropic's directory: a submission through the developer portal, which needs a paid claude.ai plan. An approved plugin appears in the directory, synced into Claude Code, and in the read-only community marketplace mirror `anthropics/claude-plugins-community`, which closes pull requests automatically. `claude-plugins-official` takes no submissions through the portal; Anthropic partners reach it through their contact. | Claude Code can recommend a plugin from a CLI's own output, but only for plugins in an Anthropic official marketplace, not the community one, and only in attended sessions. It is out of reach until the plugin is there. |
| **ClawHub (OpenClaw)** | The skill registry of OpenClaw, an open-source personal agent. Tens of thousands of skills. | An account and a published skill. No verified-publisher system is documented; each skill gets a scan verdict and a Skill Card with its provenance. | Malicious skills were found here in early 2026 (§4). Uploads are scanned by VirusTotal (at first after publication), and since the ClawScan gate by OpenClaw's static analysis and NVIDIA SkillSpector before a skill enters the catalog. |
| **MCP Registry** | The official catalog of MCP servers at registry.modelcontextprotocol.io. Clients and other registries read it. | A `server.json` and the `mcp-publisher` CLI. A namespace in reverse-DNS form needs proof of ownership: GitHub sign-in for `io.github.<name>`, or a DNS TXT record or HTTP file for a domain. | Worth it only if DeepDriveCleaner ships an MCP server. The DNS proof is a TXT record `v=MCPv1; k=ed25519; p=…`; the HTTP proof is a file at `/.well-known/mcp-registry-auth`. A CLI with `--json` and schemas already serves agents that can run commands. |
| **`llms.txt`** | A Markdown file at a site's root that points language models to the pages that matter. | One file on the product site. | There is no standards body behind it. An Ahrefs study of 137,000 domains found that 97% of `llms.txt` files got no requests in May 2026. Google says it does not affect Search. It is cheap, but expect little from it. |
| **Product pages that answer questions** | Plain pages such as "What is System Data on a Mac" and "Why is my `$TMPDIR` full", with measured numbers. | The product site (DDC-004). | This is what models and agents' web searches pick up. The two research notes in this vault are drafts of such pages. |
| **Agents-only social networks (Moltbook)** | A Reddit-like site where AI agents post and people read. | An agent account. | Wiz found the site's database open, with about 17,000 people behind about 1.5 million agents and no check that an agent is one; NBC News reported that posts may be guided by people. Its audience is curious people, not users looking for a cleaner. Posts by our own bots would be spam. |
| **Homebrew** | The existing install path: `brew install --cask smartandpoint/tap/deepdrivecleaner`. | Already done. The cask is in our tap, `smartandpoint/tap`, not in Homebrew's core casks. | The command every skill and every page should give. Agents know `brew`, and the app is signed and notarized. The cask needs Apple silicon and macOS 26, so the skill must say what to do elsewhere: nothing, not an improvised install. |

### 3. What DeepDriveCleaner has that this channel rewards

- **Agents create waste, which `ddc` could find.** One `$TMPDIR` held 790,104 abandoned test folders, about 10 million directories with them, and agents' scratch folders in `/private/tmp` held 10.5 GB. The same machine had seven test scripts of our own still running after three days. Add git worktrees with their own `node_modules`, model caches and container images. Agents create and abandon state faster than any person does.

  Today this is not what `ddc` finds. `$TMPDIR` and `/private/tmp` are outside the default scan roots. Inside `$TMPDIR` the matches were 52,779 `node_modules` fixtures of class `review`, and the home rules misfire there. A scan root for temporary folders and rules for agents' scratch folders are the agent-footprint work.
- **The CLI is already agent-shaped:**
  - `scan --json` and `clean --items` with JSON schemas in `docs/`;
  - documented exit codes;
  - `--dry-run`;
  - `--progress json`;
  - a log of actions.

  An agent can read a scan and propose a list without parsing human text.
- **The safety model is the selling point.** It was built so that a person approves an exact list, never a pattern. That is the property a user wants before letting an agent near a delete command.

### 4. The risks

**Impostor and malicious skills.** In early 2026, Koi Security's audit of ClawHub found a campaign named ClawHavoc:
- skills under names similar to popular tools, and lookalikes of ClawHub's own tools;
- 335 of the first 341 installed Atomic Stealer (AMOS), a macOS stealer of passwords, keychains, SSH keys and crypto wallets, through fake prerequisites;
- counts grow with the date: 341 in Koi's first audit, as reported by The Hacker News, then 824 in Koi's update; Antiy CERT counts 1,184 that appeared over time.

macOS users were the target. They are our users.

Unit 42 describes skills as a supply-chain risk, because a skill runs with the agent's rights. OpenClaw's own post says signature scanning cannot catch a skill that tells the agent in plain language to do harm.

For a disk cleaner the worst impostor is obvious: a skill named after DeepDriveCleaner that deletes the wrong things or uploads what it finds. Publishing the official skill first, as the owner, under the product's name and with a link from the product site, is the defence.

The tap is part of the official identity. The bare cask name `deepdrivecleaner` in Homebrew's core casks is unclaimed, so a skill that says `brew install --cask deepdrivecleaner` would fail today, and could install someone else's cask tomorrow. Every official text gives the tap-qualified command. On 2026-10-28 the owner asked to submit `deepdrivecleaner` and `ddc` to Homebrew. §6 records why that waits.

**Prompt injection through scanned content.** A scan reads names of files and folders written by anyone, such as a downloaded archive or a cloned repository. If an agent feeds scan output to its model, a folder name can carry instructions. Two measures follow:
- The skill tells the agent to treat paths as data.
- The CLI reads a handful of marker and pin files, such as `pyvenv.cfg`, `CMakeCache.txt`, `.nvmrc` or a `.git` worktree file. The report carries paths, rule ids, classes and sizes, and, for device backups, the device name and backup date from the backup's `Info.plist`. A device name is set by a person, so the skill treats it as data too.

`clean` acts only on the listed items, each re-checked against the scan and the protected paths. That guarantee is in the CLI, not in the skill.

**The agent as the one who confirms.** The safety model has two gates:
- a person picks the items;
- `--yes` without a terminal cleans only the items selected by default, in the classes named with `--class`, `safe` by default, and never `review`.

An agent can pass both on its own. The CLI cannot tell a person's item list from one an agent wrote. And an item list without a terminal can already do more than `--yes`:
- send `review` items, such as ML models and toolchains, to the Trash;
- delete them permanently, with `permanent: true` on an item;
- send the user's own files to the Trash, whole iPhone backups included.

The safety ADR says so for user data: "The mitigation is in the app". An agent that calls the CLI bypasses the app, so the mitigation is absent.

What stops an agent today is the agent host's permission prompt for shell commands. That is the user's setting, not ours, and it decays with use: once the user allows `Bash(ddc:*)` for convenience, every later `ddc clean --items` runs without a question. The realistic worst case is that agent, with that allow-list, misled by a prompt injection.

The owner decided that the host's prompt is not enough (decision 1 in the Conclusion). No mode in the CLI is a hard gate against a hostile agent, though: an agent with a shell can change a setting or run `rm` without `ddc` at all. What a gate in `ddc` protects against is the official skill's mistakes and an agent that means well. Two cheap measures help:
- the skill's `allowed-tools` (an experimental field of the specification) pre-approves only `ddc scan` and `ddc clean --dry-run`. The grant lasts only for the turn that invoked the skill. The skill's workflow spans several turns (scan, show, the person chooses, the list), so later commands prompt again, which pushes the user toward "always allow", the decay described above. It does not restrict anything either: the user's own allow-list still applies to every other command;
- in the Claude Code plugin, a `PreToolUse` hook is the gate, and the only measure that holds across turns. It must match both command names (`ddc` and `deepdrivecleaner`), absolute paths and `sh -c` wrappers. It denies `ddc clean` without `--dry-run`, or turns it into a question to the person. When the user has opted in to agents cleaning the `safe` class (decision 1), it lets a clean of `safe` items only through.

**Access prompts.** An agent's `ddc scan` runs with the agent host's access. Without Full Disk Access for the host, reading Desktop, Documents or Downloads shows macOS dialogs that name the host app, and the read blocks until someone answers (safety ADR §8). The skill must say so, and prefer explicit roots or `--skip`.

**Name collision.** "DDC" is also the name of the display protocol (DDC/CI) and of common Linux tools around it. Pages and skills should lead with "DeepDriveCleaner", and use `ddc` only as the command.

### 5. The domains

The owner bought three domains on 2026-10-28.

- **`deepdrivecleaner.app`** is a good canonical site:
  - the `.app` top-level domain is on the HSTS preload list, so browsers reach it only over HTTPS;
  - the name says what the product is.
- **`deepdrivecleaner.com`** redirects to `.app`, so that people who type `.com` and older links land on the same pages.
- **`ddc.sh`** is short and reads as the command. Possible uses:
  - **a stable short link for agents and docs**, for example `ddc.sh/skill` to the official skill repository and `ddc.sh/agents` to the page for agent users;
  - **the MCP Registry namespace**, proved by a DNS TXT record (`sh.ddc/...`), if an MCP server is ever built;
  - **not an install script.** `curl … | sh` is the pattern a careful user, and a careful agent, should refuse. It would also contradict the safety argument. Install stays `brew install --cask smartandpoint/tap/deepdrivecleaner`.

The website itself is DDC-004, "Product website and agent-first contribution guide". The choice of the canonical domain belongs there, not in this note. DDC-004 has no story yet for the domains, `llms.txt` or the answer pages; one is needed.

### 6. Homebrew's own catalogs

Checked on 2026-10-28 against Homebrew's acceptance policy, licence guidelines and `brew search`:

| Rule | Where we stand |
|---|---|
| A repository less than 30 days old is normally not eligible. | The code repository was created on 2026-10-17. It turns 30 days old on 2026-11-16. |
| A self-submission by the repository owner needs at least 90 forks, 90 watchers or 225 stars. A submission by someone else needs 30, 30 or 75. | 0 forks, 0 watchers, 0 stars. The policy allows exceptions "when these metrics do not represent the software's actual use"; for recently released apps, the cask rules add "substantial, independently verifiable public interest and multiple requests for inclusion". |
| `homebrew/core` formulae must use a licence compatible with the Debian Free Software Guidelines, and must build from source. | The PolyForm Noncommercial licence restricts commercial use, so it is not DFSG-compatible. A `ddc` formula in `homebrew/core` is not possible under this licence. |
| Casks may be proprietary or non-free. | The cask could go to `homebrew/cask` once it is notable. |
| Names: when two unrelated apps share a token, the more widely known one keeps it. | No formula or cask is named `ddc` or `deepdrivecleaner` today, so the rule does not apply. "DDC" is widely known as the display protocol (`ddcctl`, `ddcutil`, `m1ddc` are in Homebrew), which makes `ddc` a poor name to be found by (§4 "Name collision"), not a policy obstacle. |

What follows:
- **`deepdrivecleaner` in `homebrew/cask`:** submit when the repository is past 30 days old and past the self-submission thresholds, or when users ask for it in Homebrew's issues. The cask keeps its `binary … target: "ddc"`, so `ddc` comes with it; no separate package named `ddc` is needed.
- **`ddc` as its own package:** not possible in `homebrew/core` under the current licence, and not useful as a second cask for the same app, because a duplicate cask is not eligible.
- **The licence stays.** On 2026-10-28 the owner kept PolyForm Noncommercial 1.0.0 after weighing GPL-3.0 with dual licensing. No licence both forbids commercial use and meets the DFSG (its point 6 forbids discrimination against fields of endeavour); Homebrew disabled its `terraform` formula when that project moved to BUSL. `homebrew/cask` has no such rule, so the licence does not block the cask.
- **Tap trust.** Since Homebrew 6, formulae and casks from third-party taps load only when trusted (`brew trust`). The documentation says that installing by the fully qualified name trusts that one item, so `brew install --cask smartandpoint/tap/deepdrivecleaner` should work unchanged; some user reports say such installs were refused. The README's command needs one check on a clean user account. A cask in `homebrew/cask` needs no trust step, which is one more reason to get there.
- **Until then** the tap is the official source. The name stays unclaimed in Homebrew's catalogs, so the defence is the one in §4: every official text gives the tap-qualified command.

## Comparison

| Option | Pros | Cons |
|---|---|---|
| A. Classic SEO only | known practice | slow; agents' answers increasingly replace the search page |
| B. Agent channels: official skill, owner-published listings, pages that answer questions | reaches the person at the moment their agent suggests a tool; one skill works in many agents; pre-empts impostors | a skill to maintain per CLI change; registries' rules change often |
| C. Agents-only social networks and bot posting | visible for a few days | audience is not users; spam; reputational risk |
| D. MCP server first | listed in the MCP Registry and in many clients | a second interface to secure; the CLI already serves agents that run commands |

## Conclusion

Option B is recommended, in this order:

1. **An official skill in the Agent Skills format**, in the code repository or its own public repository. It teaches one workflow:
   - run `ddc scan --json`;
   - show the person the groups, classes and sizes;
   - prepare the item list for what the person wants removed, and hand it to the person's confirmation: the app's confirmation sheet, or a `ddc clean --items` command the person runs at a terminal;
   - only under the Settings opt-in of decision 1, run the clean itself, and then only for `safe` items;
   - report the result from `--json`.

   Outside the opt-in it never uses `--yes`; it never sets `permanent`, never lists the user's own files, and treats paths as data. When `ddc` is missing it gives `brew install --cask smartandpoint/tap/deepdrivecleaner`, and on an Intel Mac or older macOS it says the product does not run there. The removal never runs on the agent's own say-so by default (decision 1 below). How the agent hands a list to the app, and what it runs under the opt-in, are for the agent-footprint stories.

   The item list needs each item's device and inode, rule, class and action. The skill ships a small script that builds the list from `scan --json` for the items the person chose, and names the schema versions it was written for, so the agent never builds the list by hand.
2. **Publication, as the owner:**
   - a Claude Code marketplace in a SmartAndPoint repository;
   - a submission to Anthropic's directory through the developer portal (a paid claude.ai plan); an approved plugin also appears in the `claude-community` marketplace mirror;
   - the skill on ClawHub from the SmartAndPoint account, with its Skill Card pointing to the repository.

   The product site links to each listing, so a user can tell the real one.
3. **Pages that answer questions**, in DDC-004:
   - "What is System Data on a Mac";
   - "Why is my temporary folder full";
   - "What coding agents leave behind";
   - an `llms.txt`.

   The two research notes give the numbers, once they leave draft. `ddc.sh` short links point to the skill and the agent page.
4. **An MCP server** only if users ask for it. Then a `server.json` under the `ddc.sh` namespace.
5. **Agent footprint**, the proposed DDC-003: the skill and a session-end hook that shows the person what the session left behind and removes only what they approve (decision 2 below).

Not recommended: posting on agents-only social networks, and any `curl | sh` installer.

### Owner's decisions, 2026-10-28

The owner chose between three options for each question. For the clean: (a) the agent may confirm, the host's prompt being the only gate; (b) the agent never confirms, and a person confirms in the app or at a terminal; (c) the agent may clean the `safe` class on its own. For the session-end hook: (a) it reports only; (b) it moves the session's own folders to the Trash; (c) it removes them.

1. **An agent does not confirm a clean, by default.** This is option (b). An agent may run `scan` and prepare an item list. The removal waits for a person's confirmation in the app or at a terminal.

   In Settings the user can choose option (c) at their own risk: an agent may clean the `safe` class on its own. Choosing it shows a warning and the disclaimer, and the choice is recorded.

   Consequences:
   - **An amendment to the safety ADR.** Today §5 accepts `--yes` without a terminal for the classes named with `--class`, and `--items` without a terminal accepts `review`, `permanent` and user data (the ADR's amendments of 2026-10-18 and 2026-10-20; this note's §4).
   - **A way for `clean` to tell a confirmed request from an agent's.** The app calls `clean --items` without a terminal too.
   - **Visibility and revocation.** While the opt-in is on, the app shows it, and the user can turn it off in one place.
   - **The limit of the gate.** It stops the official skill and agents that mean well. It cannot stop an agent with a shell that means harm (this note's §4).

   Both belong to the agent-footprint stories and their ADR amendment, not to this note.
2. **The session-end hook asks; the person approves.** This is none of the three options as written: the hook neither only reports nor removes on its own. The hook never removes on its own. At the end of an agent session it collects what the session left behind and shows the person the list of folders and files, with what each would undergo: the Trash, or removal. The person approves the list, or a part of it, and only then is anything removed.

   The request is the product's demonstration. The person sees, at the moment the waste appears, what DeepDriveCleaner found and that nothing goes without their approval. This fits the owner's rule that a scan informs and people remove.

   Which surface asks is for the agent-footprint stories to decide: the app's confirmation sheet, or a notification that opens it. The agent's chat is not one: a session-end hook cannot talk to the agent (Claude Code's `SessionEnd` has no decision control and a short time budget), and an approval in chat is the path decision 1 rules out by default.

   The first problem of those stories is attribution. Nothing records today which folders a session created. Several sessions share one `$TMPDIR`, and a folder another session still uses must never be offered. A per-session temporary folder or a journal kept by the hook are the candidates.

## References

- [[research/what-fills-system-data-on-a-developers-mac|What fills System Data on a developer's Mac]]: the agent debris measurements
- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]: the home folder side, material for the answer pages
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §5 `--yes` and the `safe` class; the item list; protected paths
- [[epics/DDC-004/epic|Product website and agent-first contribution guide]] (DDC-004): the site, `llms.txt` and the answer pages
- Agent Skills specification: https://agentskills.io/specification.md
- Agent Skills donated to the Agentic AI Foundation (Simon Willison, 2026-10-22): https://simonwillison.net/2026/Oct/22/donating-agent-skills/
- Claude Code skills, `allowed-tools`: https://code.claude.com/docs/en/skills.md
- Claude Code hooks (`PreToolUse`, `SessionEnd`): https://code.claude.com/docs/en/hooks.md
- Claude Code, recommend a plugin from a CLI: https://code.claude.com/docs/en/plugins/cli-hints.md
- Claude Code, discover and install plugins: https://code.claude.com/docs/en/discover-plugins
- Claude Code, create a marketplace: https://code.claude.com/docs/en/plugins/create-marketplace.md
- Claude Code, publish a plugin: https://code.claude.com/docs/en/plugins/publish.md
- MCP Registry (in preview): https://registry.modelcontextprotocol.io/, its authentication https://modelcontextprotocol.io/registry/authentication, and its repository https://github.com/modelcontextprotocol/registry
- Official MCP Registry `server.json` requirements: https://glama.ai/blog/2026-01-24-official-mcp-registry-serverjson-requirements
- OpenClaw skills overview: https://www.digitalocean.com/resources/articles/what-are-openclaw-skills
- OpenClaw and VirusTotal partnership: https://www.openclaw.ai/blog/virustotal-partnership
- OpenClaw integrates VirusTotal (CSO Online): https://www.csoonline.com/article/4129393/openclaw-integrates-virustotal-malware-scanning-as-security-firms-flag-enterprise-risks.html
- Unit 42 on OpenClaw supply-chain risk: https://unit42.paloaltonetworks.com/openclaw-ai-supply-chain-risk/
- OpenClaw and NVIDIA skill security (ClawScan, Skill Cards, SkillSpector): https://openclaw.ai/blog/openclaw-nvidia-skill-security
- ClawHavoc, 341 malicious skills (The Hacker News): https://thehackernews.com/2026/02/researchers-find-341-malicious-clawhub.html
- ClawHavoc, 341 then 824 (Koi Security's update, as reported by PolySwarm; Koi's original post is no longer served): https://blog.polyswarm.io/the-clawhavoc-campaign
- ClawHavoc, 1,184 malicious skills (Antiy CERT): https://www.antiy.net/p/clawhavoc-analysis-of-large-scale-poisoning-campaign-targeting-the-openclaw-skill-market-for-ai-agents/
- Moltbook's open database (Wiz): https://www.wiz.io/blog/exposed-moltbook-database-reveals-millions-of-api-keys
- Moltbook (NBC News): https://www.nbcnews.com/tech/rcna256738
- `llms.txt` requests, Ahrefs study as reported by PPC Land: https://ppc.land/llms-txt-adoption-rises-8-8x-but-97-of-files-get-zero-ai-requests/
- `llms.txt` adoption tracker (vendor): https://www.rankability.com/data/llms-txt-adoption/
- Homebrew package acceptance policy: https://docs.brew.sh/Package-Acceptance-Policy
- Homebrew acceptable casks: https://docs.brew.sh/Acceptable-Casks
- Homebrew acceptable formulae: https://docs.brew.sh/Acceptable-Formulae
- Homebrew licence guidelines: https://docs.brew.sh/Licence-Guidelines
- Homebrew tap trust: https://docs.brew.sh/Tap-Trust
- Homebrew's disabled `terraform` formula: https://formulae.brew.sh/formula/terraform

---

*Last updated: 2026-10-28*
