You are running in an exe.dev VM.

https://exe.dev/docs/proxy.md has details about the exe.dev HTTPS proxy.

Only use documented exe.dev features (see https://exe.dev/docs.md). Undocumented local endpoints are internal infrastructure—unstable and unsupported.

## This VM runs exeslim, not exeuntu

It is a deployment target: a minimal base carrying systemd, TLS roots and curl,
and nothing else. Do not assume the usual exeuntu toolbox is present.

**Not installed:** compiler/`build-essential`, python, node, Go, uv, git,
docker, nginx, Chrome. `apt` works, so `sudo apt-get install <pkg>` is fine for
anything you genuinely need — but prefer building artifacts elsewhere and
shipping them here, which is how this box is meant to be used.

**Services are systemd units.** Start/stop with `systemctl`, and read logs with
`journalctl -u <name>` — journald storage is persistent, and with no toolchain
on the box it is the main debugging surface.

**The app is normally a prebuilt binary** deployed by `scp`/`rsync` from a
laptop or CI, then restarted via systemd. Anything you install directly on the
VM is lost if it is recreated from a fresh image, so durable changes belong in
the deploying repo, not here.
