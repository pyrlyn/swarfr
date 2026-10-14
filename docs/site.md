---
title: swarfr
tagline: Sweep the swarf out of Cargo target/ directories without deleting what you still build with and without slowing builds.
repo: https://github.com/pyrlyn/swarfr
install: "cargo install --locked --git https://github.com/pyrlyn/swarfr swarfr"
version: "0.1.1"
accent: "#D9A441"
order: 6
---

<!-- Website copy for the pyrlyn project site, copied to pyrlyn/landing as
content/projects/swarfr.md; front matter follows CONTENT_CONTRACT.md in that repository.
Sources (checked 2026-10-14): README.md, docs/usage.md and docs/passes.md; version from the newest GitHub release (v0.1.1, a prerelease). The accent is a placeholder: the repository has no brand colour yet. -->

## Overview

Much of a Cargo `target/` directory is swarf: duplicated and highly compressible build output,
and whole targets of worktrees git no longer knows about. `swarfr` finds every build dir under a
root and shrinks it without deleting anything you still build with: it works under cargo's own
build lock, keeps mtimes so nothing is rebuilt, and skips a profile with a running build.

swarfr is early. Lossless `compress` and `dedupe` need a filesystem that can share blocks or
compress (APFS, btrfs, XFS with reflinks, bcachefs); on Windows it builds and reports only.

## Features

- **compress.** Transparent filesystem compression of stable build artifacts; hardlink groups stay groups.
- **dedupe.** Identical files across and inside targets become copy-on-write clones of one copy.
- **seed.** A new worktree's target starts as a clone of a sibling's, so third-party crates are not rebuilt.
- **Opt-in lossy passes.** `orphans`, `evict`, `incremental` and `doc` delete rebuildable data only when named with `--lossy`.
- **advise.** Read-only report of what makes targets bigger than they need to be, file and key included.
- **Beyond Cargo.** The cargo home, Go caches, SwiftPM, .NET and CMake build dirs go through the same passes.
- **Runs unattended.** `swarfr daemon install` runs the lossless passes when build dirs go cold.

## Install

Build from source with cargo:

```bash
cargo install --locked --git https://github.com/pyrlyn/swarfr swarfr
```

## Usage examples

See every build dir under a root, read-only:

```bash
swarfr status ~/code
swarfr advise ~/code
```

Plan, then apply, the lossless passes:

```bash
swarfr run --dry-run ~/code
swarfr run ~/code
```

Remove targets of worktrees git forgot, after checking the plan:

```bash
swarfr run --dry-run --lossy orphans ~/code
```

## Links

- Repository: <https://github.com/pyrlyn/swarfr>
- Documentation: <https://github.com/pyrlyn/swarfr/tree/main/docs>
- Releases: <https://github.com/pyrlyn/swarfr/releases>
- License: your choice of GNU GPLv3, a royalty-free license, or a commercial license (see <https://github.com/pyrlyn/swarfr#license>)
