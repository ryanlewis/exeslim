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

Deliberately near-zero, so upstream can be merged cleanly:

- `.github/workflows/build.yml` — publish tags use
  `ghcr.io/${{ github.repository_owner }}/exeslim` instead of a hardcoded
  `ghcr.io/ryanlewis/exeslim`. Owner-relative, so it is correct in either
  namespace and is the one change worth offering back upstream.
- This file.

Everything else — `Dockerfile`, `init`, `exe-setup.service`,
`tmpfiles-tmp.conf` — is upstream's, unmodified.

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
