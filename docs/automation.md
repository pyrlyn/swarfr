# Running swarfr automatically

## Running it automatically

The lossless passes are safe to run unattended: they never delete, they skip a profile dir with a
running build, and exit code `2` says a build was in the way. A `just` recipe after a build:

```just
swarfr:
    swarfr run ~/code || test $? -eq 2
```

Or as a service that looks when a build dir has gone cold rather than at a fixed hour:

```
swarfr daemon install
```

That writes a launchd agent (`~/Library/LaunchAgents/dev.swarfr.daemon.plist`) or a systemd
user unit (`~/.config/systemd/user/swarfr.service`) running `swarfr daemon run` at low CPU and
I/O priority, and starts it. The daemon runs the passes the config file names over its `roots`,
once per build: a build dir is visited when its last build is older than `min-age`, and not
again until it is built once more. It never keeps a build waiting for longer than its lock
budget (2 s), lossy passes run only if the config enables them, and `swarfr daemon status`
says what it did last and what is due when. `--print` shows the unit without installing it;
`swarfr daemon remove` stops and removes it. Windows has no service yet.
