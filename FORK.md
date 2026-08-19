# Why this fork exists

Fork of [`ryanlewis/exeslim`](https://github.com/ryanlewis/exeslim), which is
the real work — a minimal exe.dev base image. Nothing here improves on it.

We fork rather than consume upstream directly because the VMs this image is
for are the **internet-facing** ones. Consuming `ghcr.io/ryanlewis/exeslim`
would put the base layer of a public service on a personal account we do not
control, rebuilt weekly by an Action we do not control. Owning the build is
worth the small maintenance cost, and it matches the dotfiles convention that
GitHub under `kylelundstedt` is the canonical source of truth.

Rationale, measurements, and the deployment-lane plan live in
[`kylelundstedt/dotfiles`](https://github.com/kylelundstedt/dotfiles) →
`agent_docs/vm-disk-weight.md`.

## Divergence from upstream

Deliberately **additive**, so upstream can be merged cleanly. Nothing upstream
owns is edited except one line:

- `.github/workflows/build.yml` — publish tags use
  `ghcr.io/${{ github.repository_owner }}/exeslim` instead of a hardcoded
  `ghcr.io/ryanlewis/exeslim`. Owner-relative, so it is correct in either
  namespace and is the one change worth offering back upstream. The same job also
  gained a second build/push step for `exeslim-dev`.
- `Dockerfile.dev` — the `exeslim-dev` image: exeslim plus the minimum a *dev* VM
  needs (`git jq unzip libyaml-0-2 openssh-client nginx-light`),
  `DBUS_SESSION_BUS_ADDRESS`, `EXPOSE 9999`, and
  `LABEL exe.dev/install-shelley=true`. A separate file rather than edits to
  `Dockerfile`, so this divergence cannot conflict on an upstream bump, and so the
  deployment lane keeps a base with no toolchain and no Shelley.
- `shelley.socket`, `shelley.service` — the label installs the Shelley binary but
  does not run it; a custom image must supply its own units. Written against
  `shelley serve -h` rather than copied from exeuntu.
- This file.

Everything upstream owns — `Dockerfile`, `init`, `exe-setup.service`,
`tmpfiles-tmp.conf`, `renovate.json` — is unmodified. Verify with
`git diff upstream/main --stat`: every path listed there should be one of the
above.

Which image a VM should use, and why the volatile tooling stays in a script
instead of either image, is documented in
[`kylelundstedt/iv-provision`](https://github.com/kylelundstedt/iv-provision)
(`README.md` → "Why a script, not a custom image", and `bootstrap.md`).

## Authoring boundary

Changes to this fork originate on the **`iv-provision` VM**, the only host
carrying the `repo-exeslim-rw` integration (it also holds `repo-iv-provision-rw`,
so the two repos' pins can be bumped together). Land changes on `main` by pull
request with CI green.

This matters more here than in an ordinary repo: this image is the base layer of
internet-facing VMs, and the reason the fork exists at all is to not depend on a
build we do not control. A second uncontrolled writer would give back part of
what forking bought. See `iv-provision`'s README ("Authoring boundary") for the
full rationale and for what to do when an exception is genuinely needed — attach
the writer deliberately, land the change, detach it again.

## Keeping current

```bash
git fetch upstream
git merge upstream/main
git push origin main          # publishes a fresh build
```

The scheduled Monday 04:00 UTC rebuild runs against **our** copy of the
Dockerfile, so upstream security work only reaches us after a merge. Check for
upstream movement when bumping a pinned VM.

## Consuming

Pin the immutable build ID — never `:latest`. exe.dev caches mutable tags and
a VM created shortly after a push can be served the previous image.

```bash
ssh exe.dev new --name=<vm> --image=ghcr.io/kylelundstedt/exeslim:<date>.<run>.<attempt>
```
