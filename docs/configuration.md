# Configuration

## Configuration

`$XDG_CONFIG_HOME/swarfr/config.toml`, or `~/.config/swarfr/config.toml`. Every key is
optional and every flag wins over the file; `--config <FILE>` reads another file instead, and a
file named there must exist. An unknown key stops the run rather than being ignored.

```toml
roots = ["~/code"]          # what `run` and `status` search when the command line names none
lossy = ["orphans"]         # lossy passes to enable, as `--lossy` would; thresholds still apply
min-age = 3600              # seconds; both lossless passes
across-families = true      # compare targets of different repositories too
min-size = 8192             # bytes; both lossless passes

[evict]
idle-days = 30
max-total-gib = 50
whole-target = true         # take the target dir itself once all of its profiles are evicted

[orphans]
project-idle-days = 7       # with `orphans`: also a target whose Cargo.toml is gone, idle this long

[incremental]
idle-days = 7

[index]
idle-days = 30              # forget hashes no run has looked up for this long; the default

[family."/Users/me/code/monorepo/.git"]
skip = true                 # never touch this repository and its worktrees

[family."/Users/me/code/big/.git"]
skip-paths = ["vendor", "third_party/llvm"]  # these places, in every checkout
ecosystems = ["cargo", "cmake"]              # only these build systems here
```

A family is a repository and its worktrees, keyed by the git common dir `status` prints (a target
without a repository is its own family). Per family there is only what to leave alone: `skip`
for all of it, `skip-paths` for build dirs whose place in their checkout starts with one of the
paths (whole components), and `ecosystems` for the build systems it is worked on by. A skipped
build dir is left out before any pass chooses: the `evict` cap does not count it either. The
`evict` cap and the idle rules are decided over everything under the roots at once, so they stay
global.
