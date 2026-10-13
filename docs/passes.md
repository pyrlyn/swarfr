# Commands and passes

What `status`, `run`, `seed` and `advise` do, every pass and its flags, and the ecosystems beyond Cargo. The step-by-step guide is [usage.md](usage.md).

## Usage

Works today:

```
swarfr status ~/code            # read-only: every target under the root
swarfr status --cargo-home ~/code  # and what the registry sources weigh
swarfr status --json ~/code
swarfr run --dry-run ~/code     # plan only
swarfr run ~/code
swarfr run [--dry-run] [--pass <PASS>]... [--lossy <PASS>]... [--index <FILE>]
               [--config <FILE>] [--json] [<ROOT>...]
               [--min-age <SECS>] [--min-size <BYTES>]
swarfr advise ~/code            # read-only: what makes these targets bigger
swarfr seed [--from <DIR>] [--dry-run] [--index <FILE>] [<DIR>]
swarfr worktree add [--dry-run] [--index <FILE>] <GIT ARGS>...
swarfr run --cargo-home ~/code     # and the registry sources in ~/.cargo
swarfr run --store "$(go env GOCACHE)"  # a content-addressed store, compress only
swarfr run --go                         # GOCACHE, and the unpacked modules in GOMODCACHE
swarfr run --dry-run --lossy orphans ~/code
swarfr run --dry-run --lossy evict --evict-idle-days 30 ~/code
swarfr run --lossy evict --evict-max-total-gib 50 ~/code
```

`status` lists build dirs grouped by family (a repository and its worktrees), then by checkout,
with a subtotal per ecosystem and its five largest build dirs (`--all` lists every one), each by
its place in the checkout: size on disk as `du` counts it, days since the last build, `ORPHANED` for a worktree git no longer knows, `PROJECT GONE` for a target whose `Cargo.toml`
is gone from a live checkout, and
totals — bytes not compressed yet and an upper bound of what dedupe could share.

`run` applies two lossless passes. **compress**: files of 8 KB and more get transparent APFS
compression (LZFSE); hardlink groups stay groups. **dedupe**: files with equal content become
copy-on-write clones of one copy, compressed if that copy is. Targets are compared inside a
family, which is where most duplicates are. What the compression backend refused, and why, is
printed at the end. A `<ROOT>` is searched for targets; a target dir itself works too.

It takes cargo's own lock, skips profile dirs with a running build, leaves alone files younger
than one hour or too small to win a block (8 KB for compress, 4 KB for dedupe), works on private
copies and swaps them in with `rename`, keeps mtimes so nothing is rebuilt, and remembers content
hashes in `~/.cache/swarfr/hashes-v1.bin` so the next run reads only new files.
Only dirs carrying cargo's own `CACHEDIR.TAG` count as targets. `--lossy` enables a
pass that deletes rebuildable data; lossless passes need no flag. How the engine keeps a target
safe is described in `DESIGN.md`, "Engine" and "Safety invariants".

`--pass <PASS>` runs only the passes you name (`orphans`, `evict`, `incremental`, `doc`,
`compress`, `dedupe`), which
is how the benchmarks tell them apart. `--min-age` and `--min-size` move the two floors below;
they exist for measurements, and the defaults are what `docs/bench.md` justifies.

**orphans** deletes, so it is off unless you name it: `--lossy orphans` removes the whole
`target/` of a checkout that is a git worktree the repository no longer registers (its `.git`
file points at a missing worktree record). Nothing outside `target/` is touched — such a
checkout can hold work git can no longer report. A target moved out of its checkout
(`CARGO_TARGET_DIR`) goes when that checkout is gone altogether: its owner is read from the
dep-info cargo leaves in it. No threshold, and every removal is printed
with its reason on a dry run too. A target whose `Cargo.toml` is gone from a checkout that is
still there — a deleted crate, or one only another branch has — goes too, but only with
`--orphans-project-idle-days N` and only once it has not been built for N days: a branch switch
looks exactly like a deletion. Without the flag it is only reported.

**seed** copies instead of deleting. In a fresh worktree, `swarfr seed` clones the target of
a sibling checkout of the same repository — the one built most recently, at the same place
inside it — into yours. On APFS every file is a `clonefile`, so the new target shares its blocks
with the old one and costs no disk space until something rewrites it. `incremental/`, the lock
files and leftover temp files stay behind, a profile dir with a running build is reported and
not copied, and a checkout that already has a target is refused rather than merged into.

How much of the first build it saves depends on what moved: units whose absolute path changed
(the workspace members, path dependencies) are compiled again, everything else is reused. The
test suite measures it against an empty target rather than assuming it.

**incremental** deletes, so it is off unless you name it: `--lossy incremental
--incremental-idle-days <N>` drops `target/<profile>/incremental/` in profile dirs with no build
for N days. Cargo writes that cache for workspace members only, and it is not part of a
fingerprint: right after the pass nothing is stale at all, and the cost is one non-incremental
rebuild the next time you edit a crate in that workspace.

**doc** deletes, so it is off unless you name it: `--lossy doc` removes `<target>/doc`, the
rustdoc output `cargo doc` writes again from scratch and no build reads — `cargo clean --doc` by
another name. It goes only while this tool holds the target's build locks and nothing in the
target is being built, and its size is reported like every other removal.

**evict** deletes, so it is off unless you name it: `--lossy evict` plus `--evict-idle-days <N>`
(profile dirs such as `target/debug` with no build for N days), `--evict-max-total-gib <N>`
(then the least recently built, until everything under the roots fits), or both. Only whole
profile dirs go, only under cargo's lock, never one with a running build or one built since the
run started looking. Every removal is printed with its reason; `--dry-run` prints the same list
and removes nothing. Cargo rebuilds what was removed on the next build of that profile.

Add `--evict-whole-target` and a target whose every profile dir is being evicted goes whole, so
`doc/`, `package/`, `tmp/` and `CACHEDIR.TAG` leave with it instead of surviving as an empty
shell. One profile with a build running, or one built since the run started looking, keeps the
target dir and the free profiles are evicted on their own. Only `target/` is ever removed; the
sources next to it are not.

`--across-families` compares every target under the roots with every other instead of one
repository at a time. Unrelated projects do share bytes — the same crate, the same version, the
same features — and the hash index already holds what it takes to find them. The price is a
wider lock: the run holds every target's build locks for its whole length, so it is off by
default and belongs in a scheduled run rather than between two builds. Measured
(`docs/bench.md`): about half of a freshly built target was already on disk in an unrelated
project — 172.6 MiB of 351.8 MiB — and nothing went stale.

`status` and `advise` also report what a toolchain upgrade left behind: cargo records the
compiler that built each unit in its own fingerprints, so units whose rustc is no longer the one
in use are countable without parsing a single hashed file name. The bytes are an estimate — the
profile's size in the share of those units — because a fingerprint does not name the files it
produced. Nothing removes them: `cargo clean`, or a rebuild, is the only cure today.

`--cargo-home [DIR]` adds the cargo home to the run as a group of its own: the extracted
registry sources (`registry/src`) and git checkouts (`git/checkouts`) are compressed like any
other stable artifact. Without a value the flag resolves `CARGO_HOME`, else `~/.cargo`. Only
compress runs there — the `.crate` archives in `registry/cache` and cargo's `registry/index` are
left alone — and the whole group is taken under cargo's own `<home>/.package-cache` lock, so a
running `cargo fetch` stops the pass instead of racing it. Nothing is deleted and no file's
content or mtime changes, which is what decides whether cargo unpacks a crate again; it does not.
Measured on a clone of a real home (`docs/bench.md`): 1.52 GiB of registry sources down to
469 MiB, 69% off, and the build afterwards reports nothing stale and unpacks nothing again.
The run needs no target dirs of its own, so `swarfr run --cargo-home` alone is a valid run,
and `swarfr status --cargo-home` reports the same dirs without touching them (it is opt-in
because measuring them costs a second walk).

`--store DIR` compresses a content-addressed store: `GOCACHE`, `~/.cabal/store`, Zig's global
`o/`, dune's shared cache — dirs whose files are named by their content and never get other
bytes. There is no lock to take, so the tool touches only entries older than an hour, runs
`compress` and nothing else, and never a lossy pass; the store's own tool evicts from it. It
refuses ccache and sccache dirs, which compress their own entries, and any dir inside a cargo
target or a cargo home. A `GOCACHE` after `go build std` went from 216 MiB to 64 MiB, and the
build afterwards compiled nothing (`docs/bench.md`). Repeatable; `stores = [...]` in the config
does the same on every run.

`--go` asks `go env` for both of Go's caches: `GOCACHE` becomes a store, and `GOMODCACHE` — the
unpacked module sources, Go's counterpart of `registry/src` — a group of its own. `go` keeps
every module dir read-only so that nobody edits a dependency by accident. The tool makes a dir
writable only while it swaps compressed copies into it, and puts its mode back straight after;
no file's bytes, mode or mtime change, so `go mod verify` stays green. A module cache of 146
MiB went down to 105 MiB (`docs/bench.md`).

SwiftPM packages are found too: a `.build` dir holding `workspace-state.json`. `swift build`
locks it with a file in the temp dir named after its path, and the tool takes the same lock, so a
running build makes the package busy exactly as a cargo build does. Only the build outputs are
worked on (`.build/out`, or `.build/<triple>` from the older build system); dependency checkouts
and the mmapped compilation cache are left alone, and dedupe uses clones only. On
swift-argument-parser built in debug and release, `.build` went from 357 MiB to 157 MiB and the
next `swift build` compiled nothing (`docs/bench.md`). Xcode's DerivedData is not handled yet.

.NET projects are found by `obj/project.assets.json`: their `obj/` and `bin/` go through the
passes too. MSBuild takes no lock, so they get the tier for build systems without one: files
younger than a day are left alone, a running `dotnet` in or around the project makes it busy,
and lossy passes run only where that check could answer. Equal files are shared by clones only,
never by hardlinks, even with `--link-artifacts`: MSBuild's `Copy` writes through a hardlink into
every other path. `UseArtifactsOutput` (`artifacts/`) is not found yet.

CMake build dirs are found by `CMakeCache.txt`, wherever they are, and go through the same
no-lock tier: a running `cmake`, `make`, `ninja` or `ctest` in or around one makes it busy, and
equal files are shared by clones only. The owner is the source dir the cache names, so a build
dir whose `CMakeLists.txt` is gone shows up as such. An in-source build (`cmake .`) is never
touched: there the build dir is the source tree. On fmt built in debug with its tests, the build
dir went from 158 MiB to 52 MiB and the next `cmake --build` built nothing (`docs/bench.md`).

**advise** changes nothing: it reads the manifests and cargo configs of the projects it finds
and names what makes their targets bigger than they need to be — full debuginfo where
`line-tables-only` would do, dependency debuginfo nobody steps into, a missing `strip` in
release, `[unstable]` keys a stable toolchain ignores without a word, and
`cache.auto-clean-frequency`, which cargo has cleaned the cargo home by since 1.88. Every finding
names the file and the key it is about, including keys the file does not have yet, because
cargo's own defaults are part of the problem. Below them come the things only the inventory
shows: what `incremental/` weighs here, families whose targets could share a `[build] build-dir`,
checkouts that `seed` would fill, and orphaned worktrees. `--json` prints the same as data.

`--json` prints the same report as one JSON document instead of the table. Exit codes: `0`
everything the run planned was done, `1` the run failed (bad flags, bad config, I/O), `2` a
profile dir was skipped because a build held its lock — what a scheduled run needs to tell
"nothing to do" from "come back later".
