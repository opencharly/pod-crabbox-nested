# AGENTS.md — pod-crabbox-nested

The **nested-engine lease image** for the Crabbox `containerd` provider: the
`pod-crabbox-nested` image (`charly.yml`) and the `containerd-nested` candy
(`candy/containerd-nested/`) it composes on top of
`opencharly/pod-crabbox-systemd`. This repo owns exactly the nested delta — the
inner containerd/nerdctl install, the `crun`/`krun` runtimes, the nerdctl bridge
subnet, the clone-identity files, the masked units and the `rtkit` exclusion. The
lease contract and systemd-as-PID-1 belong to the base repos; a change to either
lands there, not here.

Canonical files:

- `charly.yml` — the image: `base: ghcr.io/opencharly/pod-crabbox-systemd` at a
  pinned tag, `entrypoint: [/sbin/init]`, `user: root`.
- `candy/containerd-nested/charly.yml` — the variant's packages, plan steps and
  `check:` steps.
- `.github/workflows/image.yml` — the image build; it triggers on
  `tag-on-merge` **completion** (`workflow_run`), never on `push: tags`
  (`opencharly/.github#277`: tag-on-merge mints tags through the REST API, which
  emits no push event).
- `.github/workflows/tag-on-merge.yml` — the org's CalVer tag + `CHANGELOG/`
  dispatcher, installed by the org bootstrap.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-image:image` — the `candy:` image entry (base, user, entrypoint).
- `/charly-image:layer` — the candy body: `distro:` packages, plan step verbs,
  and the `check:` surface.
- `/charly-distros:cachyos` — the CachyOS base and Arch package semantics.
- `/charly-check:check` — the check verbs, the ADE requirement, and the R10 gate.
- `/charly-internals:git-workflow` — before any git/PR action.

## What is unique here

- **The image tag IS the git tag.** `.github/workflows/image.yml` passes the
  newest `v*` tag (minus its `v`) to `charly box build --tag`, which uses a
  supplied tag verbatim. `defaults.tag: auto` is only the LOCAL-build fallback;
  the CI caller always supplies the tag.
- **The base is a pinned OCI artifact, never copied.** `base:
  ghcr.io/opencharly/pod-crabbox-systemd:<tag>` pulls the published systemd
  image. A repo-scoped `GITHUB_TOKEN` cannot write the base package, so the
  external form pushes only this repo's package. Advancing the base is a one-line
  tag bump, produced first by the systemd repo.
- **The nerdctl-full version is the URL's version.** `candy/containerd-nested`
  carries `NERDCTL_FULL_VERSION`; the `download:` install fetches that exact
  release, and the build check asserts the installed `nerdctl` reports it. Bump
  the var and the check together.
- **Arch package names, not Fedora names.** The plan row names `iptables-nft`,
  `iproute`, `bzip2-libs` and `crun-krun`; on Arch/CachyOS they are provided by
  `iptables`, `iproute2`, `bzip2` and `krun` respectively. `iptables`,
  `iproute2` and `bzip2` are already in the base image and are asserted by
  `check:` steps rather than re-declared. Record the source of any new package
  in a comment.
- **`libkrun.so.1` must stay on the loader path.** Arch installs it at
  `/usr/lib/libkrun.so.1`; do not move it under a `lib64/` subdirectory or the
  inner krun fails to `dlopen` it.
- **`krun` is a systemd unit name and a binary name.** Mask `systemd-udevd` and
  the `sys-kernel-*.mount` units; never enable udev in a privileged lease. Keep
  `rtkit` out.
- **No Docker-API daemon.** The plan's docker-compatible socket is a host-side
  pass-through, not an image artifact; do not add a `dockerd` or claim a Docker
  API the image does not serve.

## Build / validate / test

- `charly box validate` at the repo root — the structural check on the image and
  its candy (including the pulled base image at its pinned tag).
- `charly box generate pod-crabbox-nested` — inspect the emitted Containerfile
  and the baked `ai.opencharly.description` plan.
- `charly check box pod-crabbox-nested` — the build-scope `check:` steps.
- The PR image build (`.github/workflows/image.yml`, `pull_request` path) builds
  the image and publishes nothing; the merge-time `workflow_run` path publishes
  `ghcr.io/opencharly/pod-crabbox-nested:<git-tag>`.

## Modify this repo

- Keep the base tag in step: a base-contract or systemd fix lands in its own repo
  first, then this repo's `base:` tag advances (producer-first).
- A package may be added only from a named source (the base candy, an existing
  `layer-*`/`pod-*` repo's Arch section, the provider's docs, or the Arch package
  index). Record the source in a comment.
- Every claim in the plan needs a deterministic `check:`. A package that merely
  installs without a probe is not proof.
- `charly check box` runs only build-context `check:` steps; a claim that needs a
  running inner engine belongs in a deploy-context bed, not here.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly`. Do not restate its rules here.
