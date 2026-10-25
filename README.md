> [!NOTE]
> This project is under active development. We need "testing volunteers": try it and report what breaks.

# swarfr

[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=listepo_dunnage&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=listepo_dunnage) [![Coverage](https://sonarcloud.io/api/project_badges/measure?project=listepo_dunnage&metric=coverage)](https://sonarcloud.io/component_measures?id=listepo_dunnage&metric=coverage) [![Tests](https://img.shields.io/sonar/tests/listepo_dunnage?server=https%3A%2F%2Fsonarcloud.io&compact_message)](https://sonarcloud.io/component_measures?id=listepo_dunnage&metric=tests)

Swarf: the chips and shavings a lathe or a mill throws off — metal cut away, waste that is not
the part. Much of a Cargo `target/` directory is swarf, and `swarfr` sweeps it out — without
deleting what you still build with and without slowing builds.

Status: early. Lossless `compress` and `dedupe` need a filesystem that can share blocks or
compress; Windows builds and reports, and the opt-in lossy passes will try to remove there,
but those two plan no work until `T21` in `plan.md`. A step-by-step guide is in `docs/usage.md`;
whether the same passes fit C++, .NET, Go and other build systems is studied in
`docs/ecosystems.md`, and the architecture that would carry them, monorepos included, in
`docs/architecture.md`.

## Why

On the machine this was designed for: 44 target dirs, ~158 GB. 114 GB of that sat in 30 worktree
checkouts git no longer knew about; nothing flags those. Inside the live targets, age-based
cleaners find ~50 MB of 14 GB, and `cargo-cache` only cleans `~/.cargo`. The live bytes are
duplicated and highly compressible:

- `deps/` compresses to ~20–30% of its size (object files to ~5%);
- 36% of bytes across sibling targets are identical content;
- a new worktree rebuilds every third-party crate from scratch.

Measured on a real 587-crate workspace (`docs/bench.md`): two freshly built targets go from
3.63 GiB to about 0.93 GiB of blocks on disk — 74% — and cargo afterwards reports not one unit
out of date. Incremental builds stay where they were, within the noise.

## How

One planner, several approaches that reinforce each other (details in `DESIGN.md`):

- **compress** — transparent APFS compression of stable artifacts;
- **dedupe** — identical files across and inside targets become copy-on-write clones;
- **seed** — a new worktree's target starts as a zero-byte clone of a sibling's, so third-party
  crates are not rebuilt;
- **orphans / evict** — opt-in removal of targets whose worktree is gone, idle targets, and
  least-recently-built targets above a global size cap.

Everything runs under cargo's own build lock, preserves mtimes so nothing is rebuilt, and treats
hardlink groups as one unit.

## Install

Releases carry binaries for Apple Silicon macOS, x86_64 Linux and x86_64 Windows. Every release is
a dev-build prerelease, so the installer link names the version (v0.1.1 is the newest):

```bash
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/pyrlyn/swarfr/releases/download/v0.1.1/swarfr-installer.sh | sh
```

Or `the package manager install pyrlyn/swarfr`, or build from a checkout — see [docs/usage.md](docs/usage.md#install).

## Usage

```bash
swarfr status ~/code            # read-only: every build dir under the root, by family
swarfr advise ~/code            # read-only: what makes these targets bigger
swarfr run --dry-run ~/code     # plan the lossless passes (compress, dedupe)
swarfr run ~/code               # apply them
swarfr seed                     # start a fresh worktree's target as a clone of a sibling's
swarfr run --dry-run --lossy orphans ~/code   # opt-in: targets of worktrees git forgot
swarfr daemon install           # run the lossless passes when build dirs go cold
```

Lossy passes (`orphans`, `evict`, `incremental`, `doc`) run only when named with `--lossy`.
Exit code `2` means a build held a lock and the run should be retried later.

## Documentation

| | |
| --- | --- |
| [docs/usage.md](docs/usage.md) | User guide: install, the first five minutes, what is safe and what deletes |
| [docs/examples.md](docs/examples.md) | Copy-paste recipes: inventory, dry runs, lossy passes, warm worktrees, the daemon |
| [docs/passes.md](docs/passes.md) | Every command, pass and flag; Cargo home, Go, SwiftPM, .NET and CMake |
| [docs/configuration.md](docs/configuration.md) | `config.toml` keys and per-repository rules |
| [docs/platforms.md](docs/platforms.md) | What works on which filesystem; hardlinks vs clones |
| [docs/automation.md](docs/automation.md) | `just` recipes and the `swarfr daemon` service |
| [docs/bench.md](docs/bench.md) | The measurements behind the defaults |
| [DESIGN.md](DESIGN.md) | Engine and safety invariants |
| [docs/architecture.md](docs/architecture.md), [docs/ecosystems.md](docs/ecosystems.md) | Beyond Cargo: architecture and ecosystem study |

## License

You can use this project under **any** of the following licenses, at your choice:

1. [GNU GPLv3](LICENSE): free for open source applications on any platform, including embedded systems.
2. [Royalty-free License](LICENSE-ROYALTY-FREE.md): free for proprietary desktop, mobile, and web applications, as long as you disclose that your application uses this project. Embedded systems are not covered.
3. [Commercial license](PRICING.md): for proprietary applications, including embedded systems, without the attribution requirement.
