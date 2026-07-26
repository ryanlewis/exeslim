# exeslim

A minimal [exe.dev](https://exe.dev) base image for **deployment targets** — VMs
that run one service and are never developed on.

```sh
ssh exe.dev new --name=my-service --image=ghcr.io/ryanlewis/exeslim:2026-07-26.7.1
```

## Why

exe.dev's stock `exeuntu` image is deliberately a batteries-included agent
workstation. Its Dockerfile runs `unminimize`, reinstalls every package to
restore man pages, then installs `locales-all`, the `ubuntu-server` /
`ubuntu-standard` / `ubuntu-dev-tools` metas, `build-essential`, Chrome plus the
GTK stack, ffmpeg, imagemagick, mitmproxy, docker, the Go toolchain and uv. Its
own comment says: *"We aim for a usable non-minimal system."*

That is the right call for a VM where an agent might need anything. It is pure
overhead for a VM running a single static binary behind Caddy.

Measured as filesystem usage on real exe.dev VMs, July 2026:

| Image | Disk used |
|---|---|
| `exeuntu`, *after* stripping caches, Go, uv, codex and the mise toolchain | 3,352 MB |
| stock `ubuntu:24.04` | 101 MB |
| **exeslim** | **175 MB** |

**~3.2 GB saved per VM.** exe.dev's Individual (Small) plan pools 100 GB of disk
measured as filesystem usage, so across a handful of service VMs this is the
difference between comfortable and cramped.

> Measuring on a live VM? Anything that has run `apt-get update` carries ~60 MB
> of package lists in `/var/lib/apt/lists`, which inflates the reading. A VM
> booted from this image and left alone sits at 175 MB.

## What's in it

`systemd` + `systemd-sysv` + `dbus` + `dbus-user-session`, `ca-certificates`,
`curl`, `iproute2`, `sudo`, `tzdata`, `locales`, and an `exedev` user at uid 1000
with passwordless sudo — matching exeuntu so existing deploy scripts keep
working.

Deliberately absent: compiler, python, node, Go, uv, git, docker, nginx, any
editor. If you need those on the box, you want `exeuntu`, not this. `apt` works,
so `sudo apt-get install` is available when something is genuinely required.

**`iproute2`** earns its ~1 MB. When a unit is `active` but the HTTPS proxy
returns nothing, `ss -tlnp` distinguishes a service bound to `0.0.0.0:8000`
(reachable by the proxy) from one bound to `127.0.0.1:8000` (not) in a single
command. On a box with no toolchain that is the difference between seeing and
guessing.

**`openssh-server` is absent on purpose.** exe.dev injects its own static `sshd`,
`sftp-server` and `sh` at `/exe.dev/bin`, so SSH works without it — verified
against a stock `ubuntu:24.04` VM.

**Agent context is carried over** even though no agent ships here, since one may
be installed later and it costs ~1 KB. `AGENTS.md` lands at the XDG path Shelley
reads, with `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md` and `~/.pi/AGENTS.md`
symlinked to it — the same layout as exeuntu. It carries exeuntu's platform
guidance plus a note that this box has no toolchain, so an agent doesn't waste a
turn discovering there's no compiler.

**Locales:** `en_US.UTF-8` (default) and `en_GB.UTF-8` are generated; exeuntu's
`locales-all` is ~200 MB for the same job. Override per VM with
`new --env LANG=en_GB.UTF-8`, on the box with `sudo update-locale`, or per
session — macOS forwards `LANG` over SSH via `SendEnv`, so an interactive login
already inherits the client's locale. Any other locale will emit setlocale
warnings; add it to `locale-gen` in the Dockerfile.

## Verified on a real VM

Not just built — booted, with `--disk=15GB`, an `--env` override and a setup
script that installed a package and registered a systemd unit:

| Check | Result |
|---|---|
| Root filesystem grown to requested size | `systemd-growfs-root` → `active / success`, `/dev/root 15G` |
| Failed units after boot | **0** |
| `exe-setup.service` (`--setup-script`) | `success`, ran as `exedev`, self-cleaned `/exe.dev/setup` |
| Proxy port from `EXPOSE` | defaulted to `8000` |
| HTTPS proxy end to end | served the app over the public URL |
| `--env LANG=en_GB.UTF-8` | applied |
| Masks held through boot | `ssh.service`, `systemd-resolved`, `apt-daily.timer` all `masked` |
| Agent context | `~/.claude/CLAUDE.md` → `~/.config/shelley/AGENTS.md`, readable |
| `ss` | `/bin/ss`, listing listeners with owning processes |

`apt-get update`, `apt-get install` and creating a systemd unit from scratch all
work on the running VM, which also confirms the `policy-rc.d` removal.

## Correlation with exeuntu

This image was diffed line-by-line against exeuntu's Dockerfile. Everything
below is platform wiring that exe.dev depends on, and is reproduced here:

| Platform requirement | Why it matters |
|---|---|
| `/etc/fstab` with `x-systemd.growfs` on `/dev/vda` | Without it `systemd-growfs@-.service` never runs, so `--disk=50GB` or `resize` boots with an **unexpanded root filesystem**. `/dev/vda` confirmed as the real root device on live VMs. |
| `exe-setup.service` | Runs `/exe.dev/setup` on first boot. Without it `new --setup-script` is accepted and silently does nothing. |
| `CMD` file named exactly `init` | exe.dev's exetini decides a wrapper is an init *from the basename*, and execs rather than forks it. |
| `EXPOSE 8000` | Sets the default proxy port. With no `EXPOSE`, exe.dev defaults to `:80`. |
| `LABEL exe.dev/login-user=exedev` | Which user SSH lands as. |
| systemd mask/disable list | Units that hang or fight the platform in a container-as-VM: resolved, udev, the `-.mount` / `etc-*.mount` set, firstboot, apt-daily. `ssh.service`/`ssh.socket` are masked because exe.dev supplies its own sshd. |
| `DefaultOOMPolicy=continue` + `SystemCallArchitectures=native` | Container-appropriate manager defaults. |
| `Storage=persistent` journald | `journalctl -u` is the main debugging surface on a box with no toolchain. |
| `/etc/tmpfiles.d/tmp.conf` | Stops systemd wiping `/tmp` at boot, which races boot-time users. |
| `rm /usr/sbin/policy-rc.d` | Otherwise apt-installed services silently fail to start. |
| exedev uid 1000 via `usermod -l` | Renames the stock `ubuntu` user so uid/gid/home/subuid line up with exeuntu. |
| linger + `XDG_RUNTIME_DIR` | Populates `/run/user/1000` so `systemd --user` works. |

Deliberately **not** carried over: Chrome/headless-shell, the Claude/Codex/pi
agent *binaries*, the pi extension and catalog, nginx + its index page,
xterm-ghostty terminfo, `EXPOSE 9999` (Shelley), `EXEUNTU=1`, and the whole
`unminimize` + man-pages + `locales-all` restoration. The agents' *context*
files are kept — see above.

`systemd-logind` is disabled but *not* masked, matching exeuntu — it is involved
in populating the XDG runtime dir sockets.

## How it boots

exe.dev runs the image's `Cmd` as PID 1. `/usr/local/bin/init` mirrors exeuntu's
own wrapper: create `/run/systemd`, mount cgroup2 if absent, remount `/proc/sys`
rw, set `ip_unprivileged_port_start=0` so unprivileged services can bind low
ports, then `exec /sbin/init`.

One deliberate divergence: exeuntu does that remount *after* the sysctl writes.
That works only because `/proc/sys` happens to be writable on the current
runtime — verified, `ip_unprivileged_port_start` is `0` on live VMs and services
running as `exedev` do bind `:80`. Ordering the remount first means that if
`/proc/sys` ever arrives read-only, the writes still land instead of being
silently skipped by their `-w` guards.

Without that wrapper you get exe.dev's fallback (`/exe.dev/etc/init-style` reads
`metadata`), which runs the image `Cmd` under `exe-init` — a bare PID 1 with no
service supervision at all. Since every service here is a systemd unit, systemd
is required.

## Usage

### Don't use `:latest`

**exe.dev caches mutable tags.** A VM created shortly after a push can be served
the *previous* `:latest`. This was observed, not theorised: a VM built minutes
after a push came up without a newly added package, while the same build's
immutable tag had it — and both tags resolved to an identical digest at the
registry, so the stale copy came from exe.dev's side.

Each build publishes four tags. Only one of them is immutable:

| Tag | Moves? |
|---|---|
| `:latest` | yes, every build |
| `:2026-07-26` | yes — two builds on the same day overwrite it |
| `:<sha>` | yes — the scheduled rebuild reuses the same commit |
| **`:2026-07-26.7.1`** (date + run number + attempt) | **no — pin this** |

Tags shown are examples; the current list is on the
[package page](https://github.com/ryanlewis/exeslim/pkgs/container/exeslim).

```sh
ssh exe.dev new --name=my-service --image=ghcr.io/ryanlewis/exeslim:2026-07-26.7.1
```

Or pin the digest, which is immutable by construction:

```sh
ssh exe.dev new --name=my-service \
  --image=ghcr.io/ryanlewis/exeslim@sha256:<digest>
```

### Notes

The package is public, so no `--registry-auth` is needed. If you fork this and
keep the package private, pass a token with `read:packages`:

```sh
ssh exe.dev new --name=my-service \
  --image=ghcr.io/ryanlewis/exeslim:2026-07-26.7.1 \
  --registry-auth='"ryanlewis:ghp_yourtoken"'
```

> exe.dev's SSH command parser splits arguments on spaces, so anything
> containing a space needs nested quoting: `--comment='"a b c"'`. Without it the
> text is parsed as positional arguments and `new` fails.

Want Shelley (and `new --prompt`) on these VMs? Uncomment
`LABEL exe.dev/install-shelley=true` in the Dockerfile.

## Patching

exe.dev VMs ship with the `apt-daily` timers masked, so nothing auto-patches.
The weekly CI rebuild is the patch mechanism: recreate or rebase a VM to pick up
a fresh build.

The build runs `apt-get dist-upgrade` before installing anything. That matters
more here than it looks: this image installs only a handful of packages by name,
so without a dist-upgrade every *other* package in the base layer would sit at
whatever Canonical last shipped, however often the job reran.

Renovate keeps the GitHub Actions and the base image digest current
(`renovate.json`). Ubuntu major/minor bumps are deliberately disabled so the
image can't drift off LTS onto an interim release — Renovate's ubuntu
versioning already treats only LTS as stable, so this is belt-and-braces. That
bump is a
two-yearly human decision.

Dependency PRs build without publishing — the workflow runs on `pull_request`
but only pushes tags from `main`. A digest bump or a major Actions bump
therefore gets validated before it can reach `:latest`.

## Licence

The build recipe in this repository — `Dockerfile`, `init`, `exe-setup.service`,
`tmpfiles-tmp.conf`, `AGENTS.md` and the workflow — is MIT licensed, so fork and
adapt it freely. See [LICENSE](LICENSE).

The **image** that recipe produces is a different matter: it is Ubuntu plus a
handful of Debian packages, each under its own licence (largely GPL and
permissive). MIT covers the recipe, not the distribution contents, and cannot
relicense them. Redistributing an Ubuntu-derived image is ordinary practice, but
if you ship it as part of a product, the packages' terms are the ones that
apply.
