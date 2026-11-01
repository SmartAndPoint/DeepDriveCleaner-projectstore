---
type: research
slug: "where-developer-disk-space-goes-windows"
title: "Where a developer's disk space goes (Windows)"
status: final
date: 2026-11-02
authors: ["Evgenii Konev"]
tags: [disk, windows, dev-artifacts, cleanup]
review_status: reviewed
reviewed_at: 2026-11-02
---

# Where a developer's disk space goes (Windows)

## Question

Where do the developer tools the catalog knows keep their caches on Windows, which of the shared catalog's Windows paths are right, what Windows adds, and what story "Windows scanner and rule catalog" (epic DDC-007) must handle before it flips Windows to supported. The macOS and Linux answers are [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] and [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]].

## TL;DR

With the owner's go-ahead (2026-11-02), a fixture set was built on the owner's lab machine: portable Node, uv, .NET 8, Go and a user install of Rust, and one small project per tool. Measured, read-only, on those fixtures:

1. **The shared catalog's Windows paths hold** for npm (`%LOCALAPPDATA%\npm-cache`), the pnpm store, the uv cache, pip, Cargo, rustup and Go (`%LOCALAPPDATA%\go-build`, `~\go\pkg\mod`). NuGet, pnpm's metadata cache and uv's managed Pythons have no rule on any OS.
2. **Hard links are the norm on Windows, not the exception.** A uv virtual environment shared 3,317 of its 3,364 files with the uv cache through hard links; a pnpm `node_modules` shared 1,897 of 1,923 files with the pnpm store. A scan that counts each hard-linked file as freed overstates what removing the venv or `node_modules` frees, by almost all of it. This decides step 6 of the scanner story: link counts are read, or, if that is over the load budget, the overstatement is stated.
3. **A cache can be larger than the project it serves.** One npm install of five packages left 214 MB in `npm-cache` beside an 86 MB `node_modules`.
4. **Toolchains dominate the fixtures:** a minimal Rust toolchain was 591 MB in `~\.rustup\toolchains`, the .NET 8 SDK 714 MB, the Go toolchain 236 MB in 15,657 files. They belong to the `toolchain` rules (newest kept), not to caches.
5. **WSL keeps each distribution as one disk image `ext4.vhdx`**, in one of two places: `%LOCALAPPDATA%\Packages\<publisher.distro>_<hash>\LocalState\` for Store-installed distributions, and `%LOCALAPPDATA%\wsl\{GUID}\` for the tar-based format of recent WSL releases. The image grows and does not shrink on its own. It is a VM item: reported, never deleted, and never compacted by ddc.
6. **A Windows profile holds reparse points that are not developer data:** the legacy junctions every profile carries (`Application Data`, `Local Settings`, `My Documents` and their localised names), and, where OneDrive's Files On-Demand is on, OneDrive's cloud placeholders. The walker must never follow the first kind and never open the second.

## Method

Measured on 2026-11-02, read-only, as the machine's ordinary user, over ssh:

- A fixture set built by `~\ddc-lab\setup.sh` (kept on the machine): portable Node 22, uv, .NET 8 SDK and Go in `~\ddc-lab\tools\`, Rust through `rustup-init --no-modify-path`, and these projects in `~\ddc-lab\projects\`:
  - `web-npm`: `npm install express typescript react react-dom vite eslint`;
  - `web-pnpm`: `pnpm add express lodash vite`;
  - `py-uv`: `uv venv --python 3.12`, `uv pip install requests numpy pandas`;
  - `py-pip`: a `venv` with `pip install requests flask`;
  - `dotnet-app`: `dotnet new console`, Newtonsoft.Json and Serilog, built;
  - `go-app`: a module importing cobra, built;
  - `rust-app`: `cargo new`, serde and serde_json, built.
  No `PATH` or registry change.
- Sizes and file counts of a fixed list of tool folders and the fixtures, with Windows PowerShell 5.1's `Get-ChildItem -Recurse -File -Attributes !ReparsePoint`, logical sizes; hard links counted by `LinkType`. That shell descends directory junctions, so the pnpm `node_modules` count may include a package reached twice. Disk images found only by their fixed file names (`ext4.vhdx`, `docker_data.vhdx`).
- Only the fixed list was measured; nothing else in the home was listed.

Limitations:
- Fixtures, not a lived-in developer home: the sizes show how each tool lays out its cache and what one install leaves, not how large caches grow over months. The macOS note's measured home is the reference for growth.
- Not measured: Gradle and Maven, JetBrains IDEs, VS Code and its forks, Visual Studio's caches, Docker Desktop, browsers. Their Windows paths stay as the shared catalog has them, from the tools' documentation, and step 13 of the scanner story re-checks them on a home that has them. Microsoft Edge, the browser every Windows install has, has no rule at all (`AppData\Local\Microsoft\Edge\User Data\Default\Cache` and `Code Cache`).
- Logical sizes (`Length`), not allocated sizes; NTFS compression and WOF were not in use on the fixtures.
- The fixtures were measured with NTFS semantics; ReFS (Dev Drive) and FAT/exFAT are not covered.
- The machine is private: this note describes what the tools did, not the machine.

## Findings

### 1. Cache locations

| Tool | Cache or store | Fixture size | Files | Shared catalog path |
|---|---|---|---|---|
| npm | `%LOCALAPPDATA%\npm-cache` | 214 MB | 1,775 | `AppData/Local/npm-cache`: right |
| pnpm | `%LOCALAPPDATA%\pnpm\store` (content), `%LOCALAPPDATA%\pnpm-cache` (metadata) | 39 MB + 14 MB | 1,915 + 107 | store right; the metadata cache has no rule on any OS |
| uv | `%LOCALAPPDATA%\uv\cache` | 83 MB | 3,388 | right |
| uv managed Pythons | `%APPDATA%\uv\python` | 60 MB | 3,359 | no rule on any OS; a toolchain, `review` |
| pip | `%LOCALAPPDATA%\pip\Cache` | 2 MB | 75 | right |
| NuGet | `~\.nuget\packages`, `%LOCALAPPDATA%\NuGet\v3-cache` | 17 MB + 7 MB | 47 + 21 | no rule on any OS |
| Cargo | `~\.cargo\registry` | 8 MB | 509 | home-relative: right |
| rustup | `~\.rustup\toolchains` | 591 MB | 150 | right (toolchain rule) |
| Go | `~\go\pkg\mod`, `%LOCALAPPDATA%\go-build` | 2 MB + 53 MB | 219 + 1,043 | right (`AppData/Local/go-build`, `go/pkg/mod`) |

The `uv` folder under `%LOCALAPPDATA%` held only `cache`; managed Pythons live under `%APPDATA%` (roaming), where a profile with roaming enabled copies them to the server at sign-out. The catalog should say so in the rule's description.

### 2. Hard links

| Fixture | Files | Hard-linked |
|---|---|---|
| `py-uv\.venv` (uv) | 3,364 | 3,317 |
| uv cache | 3,388 | 3,317 |
| `web-pnpm\node_modules` (pnpm) | 1,923 | 1,897 |
| pnpm store | 1,915 | 1,834 |
| `rust-app\target` | 196 | 28 |
| `py-pip\venv`, `web-npm\node_modules`, npm cache | — | 0 |

uv's default link mode on Windows is a hard link, and pnpm links packages from its store. Removing the venv or `node_modules` frees almost nothing while the cache holds the other link; removing the cache frees almost nothing while a project holds it. Cargo hard-links its uplifted artifacts (from `deps\` to `target\debug\`) inside `target` itself. Every number ddc shows for these trees is overstated unless the walker reads link counts (macOS reads them from `getattrlistbulk`, Linux from `statx`). `FileIdExtdDirectoryInfo` has no link count, so the scanner story's step 6 reads it for the files whose id repeats, measured against the load limits.

### 3. Toolchains

The .NET 8 SDK (714 MB, 5,140 files), the Go toolchain (236 MB, 15,657 files), a minimal Rust toolchain (591 MB) and uv's managed Python (60 MB) are each larger than every cache of the fixtures. Installed by their official installers they sit in `%ProgramFiles%\dotnet` and `%ProgramFiles%\Go` (outside the home: the Whole disk story's areas) or in the home (`~\.rustup`, `%APPDATA%\uv\python`, `%LOCALAPPDATA%\Programs\Python`). Only the home's are the scanner's.

### 4. Disk images

WSL keeps a distribution's disk as `ext4.vhdx` under `%LOCALAPPDATA%\Packages\<publisher.distro>_<hash>\LocalState\` (Store installs) or `%LOCALAPPDATA%\wsl\{GUID}\` (the tar-based format of recent releases). The image grows and never shrinks on its own; `wsl --manage <distro> --set-sparse true` or `Optimize-VHD` give space back, and both belong to the user. Docker Desktop's `%LOCALAPPDATA%\Docker\wsl` was not measured. Both are VM items: reported with their size, never planned.

### 5. Reparse points in a profile

- The legacy junctions every Windows profile carries (`Application Data`, `Cookies`, `Local Settings`, `My Documents`, `NetHood`, `PrintHood`, `Recent`, `SendTo`, `Start Menu`, `Templates`, localised on a localised Windows, and `AppData\Local\Application Data`, `History`, `Temporary Internet Files`) point back into the profile, several in a loop. Their ACL denies listing. A walker that followed them would count folders twice or loop; one that opened them would log access-denied noise. They are name surrogates: reported as links, never descended (the scanner story's step 6).
- With Files On-Demand on, OneDrive's files are cloud placeholders, reparse points that are not name surrogates; reading a placeholder's data downloads it. A work or school account's folder is named `OneDrive - <organisation>`, and the location can be moved, so the roots are found from the `OneDrive`, `OneDriveConsumer` and `OneDriveCommercial` variables, not by name. Dropbox and Box use the same Cloud Filter API on Windows. The scanner story keeps these roots not walked; walking them with placeholders recognised by their attributes is a later story.

## Comparison with macOS and Linux

- Same: the cache layout of npm, pip, uv, Cargo and Go, relative to their base folders.
- Different: the base folders (`%LOCALAPPDATA%` and `%APPDATA%` instead of `~/Library/Caches` or `~/.cache`); hard links as the default of uv and pnpm (macOS clones on APFS instead, Linux hard-links on ext4 as here); disk images for VMs (WSL), as Docker Desktop's raw image on macOS.

## Conclusion

For the scanner story:
1. Read link counts (step 6): without them, uv and pnpm trees show sizes that cleaning does not free.
2. Add shared rules, with a path per OS, for what no OS has yet: NuGet (`~/.nuget/packages` everywhere; the v3 cache in `%LOCALAPPDATA%\NuGet\v3-cache`, `~/.local/share/NuGet/v3-cache`, `~/.local/share/NuGet/http-cache`), pnpm's metadata cache (`%LOCALAPPDATA%\pnpm-cache`, `~/.cache/pnpm`, `~/Library/Caches/pnpm`), uv's managed Pythons (`%APPDATA%\uv\python`, `~/.local/share/uv/python`; toolchain, `review`), and Microsoft Edge's caches.
3. Both WSL disk locations (`%LOCALAPPDATA%\Packages\*\LocalState\ext4.vhdx`, `%LOCALAPPDATA%\wsl\*\ext4.vhdx`) and `%LOCALAPPDATA%\Docker\wsl` as report-only VM items.
4. The profile's legacy junctions are links, never descended; cloud roots (OneDrive by its variables, Dropbox, Box) stay not walked.
5. The tools not measured here keep their documented paths until a home with them is measured.

## Open questions

- [ ] How large the caches grow on a lived-in Windows developer home; a second measurement once one is available.
- [ ] Gradle, Maven, JetBrains, VS Code, Visual Studio and Docker Desktop paths on a home that has them.
- [ ] Confirm the tar-based WSL format's `%LOCALAPPDATA%\wsl\{GUID}` folder against Microsoft's documentation or release notes.
- [ ] Whether the `%TEMP%` rule plans the folder's contents only, given that running programs hold files there (the held-open guard skips those).

## References

- uv documentation: cache location and link mode on Windows; pnpm documentation: store location and hard links; npm, pip, NuGet, Go and rustup documentation for their default folders on Windows (read 2026-11-02).
- WSL documentation: [Build a custom Linux distribution for WSL](https://learn.microsoft.com/en-us/windows/wsl/build-custom-distro) (the tar-based format) and `wsl --manage --set-sparse`; the `{GUID}` folder of that format is not yet confirmed on an official page (open question).
- [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]] — the hard-link finding on ext4.
