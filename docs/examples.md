# swarfr — examples

Copy-paste recipes for the common jobs. Every command below was run against swarfr 0.1.1; the
sample output comes from a tiny throwaway project, so your sizes will be larger. The full option
list is in [usage.md](usage.md), the config keys in [configuration.md](configuration.md).

## See what you have, change nothing

```bash
swarfr status ~/code
```

```
family /Users/me/code/app/.git
  checkout /Users/me/code/app
    cargo: 1 build dirs, 0.00 GiB
         0.00 GiB  built 0d ago      target
1 targets, 0.00 GiB on disk (0.00 GiB logical)
orphaned worktrees: 0.00 GiB; not compressed yet: 0.00 GiB; dedupe candidates (upper bound): 0.00 GiB
```

The last line is the headline: how much is orphaned, how much a lossless run could still
compress, and an upper bound for what deduplication could share.

For scripts, `--json` prints the same inventory as one document with one entry per build dir:

```bash
swarfr status --json ~/code | jq '.targets[] | {checkout, position, guard}'
```

## Find out why targets are big

```bash
swarfr advise ~/code
```

```
/Users/me/code/app/Cargo.toml
  profile.dev.debug: full debuginfo is the largest single thing in a dev target; `debug = "line-tables-only"` keeps backtraces with file and line
  profile.release.strip: `strip = "debuginfo"` for a profile nobody debugs
```

`advise` names the key and the file it belongs in. It never edits either.

## Preview, then run the lossless passes

```bash
swarfr run --dry-run ~/code     # prints the plan, touches nothing
swarfr run ~/code               # compress + dedupe; deletes nothing
cargo build                     # in any project: nothing to recompile
```

```
/Users/me/code/app/.git
  compress: planned 0 (0 bytes), applied 0 (0 bytes), skipped 0
  dedupe: planned 0 (0 bytes), applied 0 (0 bytes), skipped 0
files hashed: 0
```

## Delete only what you name

A deleting pass runs only with `--lossy`, and `--dry-run` prints each removal and its reason first:

```bash
# worktrees git no longer knows about
swarfr run --dry-run --lossy orphans ~/code

# profile dirs not built for 30 days
swarfr run --dry-run --lossy evict --evict-idle-days 30 ~/code

# keep the most recently built, until everything fits in 50 GiB
swarfr run --lossy evict --evict-max-total-gib 50 ~/code
```

## Start a worktree warm

```bash
cd ~/code/app
swarfr worktree add ../app-fix -b fix
```

Everything after `worktree add` is `git worktree add`'s own arguments. The new worktree is then
seeded from the most recently built checkout of the same repository; if there is none, the
worktree is still added and swarfr says there was nothing to seed from:

```
added worktree /Users/me/code/app-fix
  nothing to seed from: no other checkout of this repository has a target there
```

Seed an existing empty checkout by hand with `swarfr seed --from ../app ../app-fix`. A checkout
that already has a target dir is refused.

## Run it unattended

Put the roots in `~/.config/swarfr/config.toml`:

```toml
roots = ["~/code"]
lossy = ["orphans"]

[evict]
idle-days = 30
```

Then check the config with one pass of the daemon, or install it as a service:

```bash
swarfr daemon run --once        # look once, run if anything is due, exit
swarfr daemon install --print   # show the launchd/systemd unit; writes nothing
swarfr daemon install           # write it and keep running in the background
```

To try a different config without touching the default one, pass `--config FILE` to `run` or
`daemon run` (`status` and `advise` take roots on the command line instead):

```bash
swarfr run --config ./swarfr.toml --dry-run
```
