# Platforms

## Platforms

The tool builds and runs on macOS, Linux and Windows. What it *does* depends on what the
filesystem can do, and it never pretends:

| | macOS | Linux | Windows |
| --- | --- | --- | --- |
| `status`, `advise`, `seed` | yes | yes | yes |
| `dedupe` (block sharing) | yes, APFS | yes on btrfs, XFS (`reflink=1`), bcachefs | not yet — ReFS is `T21` |
| `dedupe` where blocks cannot be shared | — | ext4, and any other: hardlinks, see below | not yet |
| `compress` | yes, APFS/LZFSE | yes on btrfs | not yet — NTFS is `T21` |

The question is the filesystem, not the operating system, so the tool asks yours instead of
guessing from its name: it writes a small temp file, tries to clone it and tries to set the
compression flag, and removes both. `status` tells you the answer:

```
  1.4 GiB  built 2d ago  /home/you/project/target
           this filesystem neither shares blocks nor compresses: compress finds nothing here, dedupe only links cargo home sources
```

A pass that cannot win anything there plans nothing and says so — it does not copy files around
for no gain.

### Sharing without copy-on-write

Where blocks cannot be shared, the only way to store one file once is one inode under both
names: a hardlink. That is safe for some files and dangerous for others, so it is split by what
the file is rather than by what the filesystem allows.

- **The cargo home's unpacked sources** (`registry/src`, `git/checkouts`) are shared with no flag
  at all. Cargo extracts a crate into a fresh directory and writes `.cargo-ok` last; it never
  rewrites an extracted file in place. This is also where the bytes are — 1.52 GiB of them on
  the machine in `docs/bench.md`.
- **Build artifacts** are shared only with `--link-artifacts`, because rustc opens its outputs
  with truncate: a later build that rewrites one linked artifact rewrites **every other name
  pointing at the same inode**, in every target that was sharing it. Nothing enables that for
  you, and the run says so while it does it. On a filesystem that clones, the flag changes
  nothing — a clone is better and needs no permission.

The engine refuses to link two files whose permissions differ, because one inode can only hold
one mode, and the shared inode keeps the later of the two modification times, so no name
suddenly reads older than what it was built from.

`seed` works everywhere: where the filesystem shares blocks the copy is free, where it does not
it costs the disk but still saves the build, and it says which of the two happened.

Version-gated features (unit-level pruning, shared build-dir automation) are in `roadmap.md`.
