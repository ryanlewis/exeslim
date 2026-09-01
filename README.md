# exeslim

A minimal [exe.dev](https://exe.dev) base image for **deployment targets** — VMs
that run one service and are never developed on.

```sh
ssh exe.dev new --name=my-service --image=ghcr.io/ryanlewis/exeslim:<build-id>
```

Take `<build-id>` from the
[package page](https://github.com/ryanlewis/exeslim/pkgs/container/exeslim) — it
looks like `2026-07-26.20.1`. `:latest` works too, but exe.dev caches it for up
to an hour — [picking a tag](#picking-a-tag).

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
| `exe-setup.service` (`--setup-script`) | `success`, ran as `exedev`, deleted `/exe.dev/setup`. exe.dev writes the file back on every boot, so the unit runs again each time; see [Setup scripts run on every boot](#setup-scripts-run-on-every-boot) |
| Proxy port from `EXPOSE` | defaulted to `8000` |
| HTTPS proxy end to end | served the app over the public URL |
| `--env LANG=en_GB.UTF-8` | applied |
| Masks held through boot | `ssh.service`, `systemd-resolved`, `apt-daily.timer` all `masked` |
| Agent context | `~/.claude/CLAUDE.md` → `~/.config/shelley/AGENTS.md`, readable |
| `ss` | `/bin/ss`, listing listeners with owning processes |

`apt-get update`, `apt-get install` and creating a systemd unit from scratch all
work on the running VM, which also confirms the `policy-rc.d` removal.

### Setup scripts run on every boot

`exe-setup.service` deletes `/exe.dev/setup` after a successful run, but exe.dev
overwrites the file on every boot with the script captured at `new` time. The
unit's `ConditionPathExists` then passes again and the script runs again. This
is platform behaviour: the stock exeuntu image ships the same unit and behaves
the same way. `ssh exe.dev doc customization` says the script runs "at first
boot, once"; on the VMs tested it does not.

So a `--setup-script` must be idempotent. A script that is not fails on the
second boot and every boot after, leaves `exe-setup.service` in `failed`, and
`systemctl --failed` is never clean again. Seen on a real VM: a script with a
bare `useradd --system` succeeded on the first boot and exited 9
(`user already exists`) on the second.

Guard anything that cannot run twice:

```sh
id -u caddy >/dev/null 2>&1 || sudo useradd --system --user-group caddy
```

The guard has to be in the script when the VM is created. `--setup-script`
exists only on `new`, there is no command to view, edit or clear the script
stored against an existing VM, and editing `/exe.dev/setup` on the box does
not help because the stored copy overwrites it at the next boot. On a live VM
that is already failing, the remaining options are to disable the unit, run
`systemctl reset-failed` after every reboot, or rebuild the VM.

## Correlation with exeuntu

This image was diffed line-by-line against exeuntu's Dockerfile. Everything
below is platform wiring that exe.dev depends on, and is reproduced here:

| Platform requirement | Why it matters |
|---|---|
| `/etc/fstab` with `x-systemd.growfs` on `/dev/vda` | Without it `systemd-growfs@-.service` never runs, so `--disk=50GB` or `resize` boots with an **unexpanded root filesystem**. `/dev/vda` confirmed as the real root device on live VMs. |
| `exe-setup.service` | Runs `/exe.dev/setup` on every boot, not only the first. Without it `new --setup-script` is accepted and silently does nothing. |
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

### Picking a tag

The tag is read once, at `new`, and never again — the VM gets a disk copy of
whatever it resolved to at that moment, and nothing re-pulls it (see
[Patching](#patching)). So this isn't pinning in the lockfile sense, where the
choice keeps applying. It's a one-shot answer to "which build does this VM start
from".

That makes `:latest` fine most of the time. The caveat is that **exe.dev caches
mutable tags**, so "latest" can mean up to an hour old. Observed, not theorised:
a VM built minutes after a push came up without a newly added package, while the
same build's immutable tag had it — and both tags resolved to an identical digest
at the registry, so the stale copy came from exe.dev's side.

The TTLs are 1 hour for `latest`, `main` and `master`, and 24 hours for
everything else. The tags that *look* specific are therefore cached longest: a
mutable `:<date>` or `:<sha>` can serve yesterday's image for a full day.

Each build publishes four tags. Only one of them is immutable:

| Tag | Moves? | Cached |
|---|---|---|
| `:latest` | yes, every build | 1 h |
| `:<date>` | yes — two builds on the same day overwrite it | 24 h |
| `:<sha>` | yes — the scheduled rebuild reuses the same commit | 24 h |
| **`:<date>.<run>.<attempt>`** — the *build id* | **no** | 24 h, moot |

Reach for the build id when you want a guarantee rather than an hour's slack:
creating a VM to exercise a change you pushed minutes ago, or rebuilding a box
on the same image as one that already exists. Otherwise `:latest` is the easier
call.

A build id looks like `2026-07-26.20.1`. The current list is on the
[package page](https://github.com/ryanlewis/exeslim/pkgs/container/exeslim);
this README quotes the *format* rather than a specific id on purpose, since a
docs-only commit publishes a new build and would immediately stale it.

```sh
ssh exe.dev new --name=my-service --image=ghcr.io/ryanlewis/exeslim:<build-id>
```

Or the digest, immutable by construction:

```sh
ssh exe.dev new --name=my-service \
  --image=ghcr.io/ryanlewis/exeslim@sha256:<digest>
```

### Notes

The package is public, so no `--registry-auth` is needed. If you fork this and
keep the package private, pass a token with `read:packages`:

```sh
ssh exe.dev new --name=my-service \
  --image=ghcr.io/ryanlewis/exeslim:<build-id> \
  --registry-auth='"ryanlewis:ghp_yourtoken"'
```

> exe.dev's SSH command parser splits arguments on spaces, so anything
> containing a space needs nested quoting: `--comment='"a b c"'`. Without it the
> text is parsed as positional arguments and `new` fails.

Want Shelley (and `new --prompt`) on these VMs? Uncomment
`LABEL exe.dev/install-shelley=true` in the Dockerfile.

## Patching

Nothing on the VM patches itself. This image masks `apt-daily.timer`,
`apt-daily-upgrade.timer` and `unattended-upgrades.service` (mirroring exeuntu),
and `unattended-upgrades` is not installed at all. Whether a box that exists to
run one service should reboot-and-upgrade on its own is a decision for whoever
owns that service, not for the base image.

What the weekly rebuild (Mondays, 04:00 UTC) buys you is a *fresh* VM that
starts patched. The build runs `apt-get dist-upgrade` before installing
anything, which matters more here than it looks: this image installs only a
handful of packages by name, so without a dist-upgrade every *other* package in
the base layer would sit at whatever Canonical last shipped, however often the
job reran.

That applies at creation and nowhere else. There is no way to move an existing
VM onto a newer image: `ssh exe.dev` offers `new`, `rm`, `restart`, `cp` and
`resize`, and none of them swap the image under a live VM (`cp` clones the
disk you already have). Once a VM exists, its packages are yours to maintain.
Since the payload here is normally a prebuilt binary and a unit file, replacing
a VM with a fresh one *is* viable — but it is a redeploy you choose to do, not
an upgrade path the platform gives you.

### Keeping a long-lived VM current

By hand:

```sh
sudo apt-get update && sudo apt-get dist-upgrade
```

Or opt in to unattended upgrades. **Unmask before installing** — dpkg cannot
enable a masked unit, so installing first leaves all three units masked with
`Failed to enable unit: ... is masked` buried in the apt output:

```sh
sudo systemctl unmask apt-daily.timer apt-daily-upgrade.timer unattended-upgrades.service
sudo apt-get update && sudo apt-get install -y unattended-upgrades
sudo systemctl start apt-daily.timer apt-daily-upgrade.timer
```

That last line is not redundant. Installing the package *enables* both timers
but leaves them inactive in the running system — `systemctl list-timers` shows
them with no next elapse until they are started or the VM is restarted, which
looks exactly like working automatic patching while doing nothing. Confirm it
actually armed:

```sh
systemctl list-timers 'apt-daily*'
sudo unattended-upgrade --dry-run --verbose
```

The stock allowed origins are `noble` and `noble-security`, which is the right
default for a deployment target: security fixes, no surprise version jumps.

There is no kernel to patch. exe.dev supplies it — `uname -r` reports 6.12.x
rather than Noble's 6.8 — and no `linux-image` package is installed, so
`/var/run/reboot-required` never shows up from a kernel update. A restart is
only needed to move long-running processes onto a patched libc or openssl:
restart the unit, or `ssh exe.dev restart <vm>`.

### Keeping the image current

Renovate keeps the GitHub Actions and the base image digest current
(`renovate.json`). Ubuntu major/minor bumps are deliberately disabled so the
image can't drift off LTS onto an interim release — Renovate's ubuntu
versioning already treats only LTS as stable, so this is belt-and-braces. That
bump is a two-yearly human decision.

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
