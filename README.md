# exeslim

A minimal [exe.dev](https://exe.dev) base image for **deployment targets** — VMs
that run one service and are never developed on.

## Why

exe.dev's stock `exeuntu` image is deliberately a batteries-included agent
workstation. Its Dockerfile runs `unminimize`, reinstalls every package to
restore man pages, then installs `locales-all`, the `ubuntu-server` /
`ubuntu-standard` / `ubuntu-dev-tools` metas, `build-essential`, Chrome plus the
GTK stack, ffmpeg, imagemagick, mitmproxy, docker, the Go toolchain and uv. Its
own comment says: *"We aim for a usable non-minimal system."*

That is the right call for a VM where an agent might need anything. It is pure
overhead for a VM running a single static binary behind Caddy.

Measured on a real exe.dev VM, July 2026:

| Image | Disk used |
|---|---|
| `exeuntu`, after stripping caches, Go, uv, codex and the mise toolchain | 3,352 MB |
| stock `ubuntu:24.04` | 101 MB |
| **exeslim** (ubuntu:24.04 + systemd + ca-certificates + curl) | **~224 MB** |

Roughly **3.1 GB saved per VM**. exe.dev's Individual (Small) plan pools 100 GB
of disk measured as filesystem usage, so across a handful of service VMs this is
the difference between comfortable and cramped.

## What's in it

`systemd` + `systemd-sysv` + `dbus`, `ca-certificates`, `curl`, `sudo`,
`tzdata`, `locales`, and an `exedev` user at uid 1000 with passwordless sudo —
matching exeuntu so existing deploy scripts keep working.

Deliberately absent: compiler, python, docker, Go, node, any editor. If you need
those on the box, you want `exeuntu`, not this.

**Agent context** is carried over even though no agent ships here, since one may
be installed later and it costs ~1 KB. `AGENTS.md` lands at the XDG path Shelley
reads, with `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md` and `~/.pi/AGENTS.md`
symlinked to it — the same layout as exeuntu. It carries exeuntu's platform
guidance plus a note that this box has no toolchain, so an agent doesn't waste a
turn discovering there's no compiler.

**Locales:** `en_US.UTF-8` (default) and `en_GB.UTF-8` are generated; exeuntu's
`locales-all` is ~200 MB for the same job. Override the default per VM with
`new --env LANG=en_GB.UTF-8`, on the box with `sudo update-locale`, or per
session — macOS forwards `LANG` over SSH via `SendEnv`, so an interactive login
already inherits the client's locale.

`openssh-server` is also absent on purpose: exe.dev injects its own static
`sshd`, `sftp-server` and `sh` at `/exe.dev/bin`, so SSH works without it.
Verified against a stock `ubuntu:24.04` VM.

## Correlation with exeuntu

This image was diffed line-by-line against exeuntu's Dockerfile. Everything
below is platform wiring that exe.dev depends on, and is reproduced here:

| Platform requirement | Why it matters |
|---|---|
| `/etc/fstab` with `x-systemd.growfs` on `/dev/vda` | Without it `systemd-growfs@-.service` never runs, so `--disk=50GB` or `resize` boots with an **unexpanded root filesystem**. Verified `/dev/vda` is the real root device on live VMs. |
| `exe-setup.service` | Runs `/exe.dev/setup` on first boot. Without it `new --setup-script` is accepted and silently does nothing. |
| `CMD` file named exactly `init` | exe.dev's exetini decides a wrapper is an init *from the basename*, and execs rather than forks it. |
| `EXPOSE 8000` | Sets the default proxy port. With no `EXPOSE`, exe.dev defaults to `:80`. |
| `LABEL exe.dev/login-user=exedev` | Which user SSH lands as. |
| systemd mask/disable list | Units that hang or fight the platform in a container-as-VM: resolved, udev, the `-.mount` / `etc-*.mount` set, firstboot, plymouth, apt-daily. `ssh.service`/`ssh.socket` are masked because exe.dev supplies its own sshd. |
| `DefaultOOMPolicy=continue` + `SystemCallArchitectures=native` | Container-appropriate manager defaults. |
| `Storage=persistent` journald | `journalctl -u` is the only debugging surface on a box with no toolchain. |
| `/etc/tmpfiles.d/tmp.conf` | Stops systemd wiping `/tmp` at boot, which races boot-time users. |
| `rm /usr/sbin/policy-rc.d` | Otherwise apt-installed services silently fail to start. |
| exedev uid 1000 via `usermod -l` | Renames the stock `ubuntu` user so uid/gid/home/subuid line up with exeuntu. |
| linger + `XDG_RUNTIME_DIR` | Populates `/run/user/1000` so `systemd --user` works. |

Deliberately **not** carried over: Chrome/headless-shell, the Claude/Codex/pi
agent *binaries*, the pi extension and catalog, nginx + its index page,
xterm-ghostty terminfo, `EXPOSE 9999` (Shelley), `EXEUNTU=1`, and the whole
`unminimize` + man-pages + `locales-all` restoration. The agents' *context*
files are kept — see above.

`systemd-logind` is disabled but *not* masked, matching exeuntu — it is
involved in populating the XDG runtime dir sockets.

## How it boots

exe.dev runs the image's `Cmd` as PID 1. `/usr/local/bin/init` mirrors
exeuntu's own wrapper: create `/run/systemd`, mount cgroup2 if absent, set
`ip_unprivileged_port_start=0` so unprivileged services can bind low ports,
remount `/proc/sys` rw, then `exec /sbin/init`.

Without that wrapper you get exe.dev's fallback (`init-style: metadata`), which
runs the image `Cmd` under `exe-init` — a bare PID 1 with no service
supervision. Since every service here is a systemd unit, systemd is required.

## Usage

```sh
ssh exe.dev new --name=my-service --image=ghcr.io/ryanlewis/exeslim:latest
```

If the package is private, pass a token with `read:packages`:

```sh
ssh exe.dev new --name=my-service \
  --image=ghcr.io/ryanlewis/exeslim:latest \
  --registry-auth='"ryanlewis:ghp_yourtoken"'
```

> exe.dev's SSH command parser splits on spaces, so any argument containing
> spaces or a colon needs nested quoting: `--comment='"a b c"'`.

Want Shelley (and `new --prompt`) on these VMs? Uncomment
`LABEL exe.dev/install-shelley=true` in the Dockerfile.

## Patching

exe.dev VMs ship with the `apt-daily` timers masked, so nothing auto-patches.
The weekly CI rebuild is the patch mechanism: recreate or rebase a VM to pick up
a fresh base.
