# pod-crabbox-nested

The **nested-engine CachyOS lease image** for the Crabbox `containerd` provider —
the [`pod-crabbox-systemd`](https://github.com/opencharly/pod-crabbox-systemd)
lease contract with an inner containerd, an inner krun, and nerdctl.

## What it provides

Every lease contract property of the base image is inherited unchanged:

| Property | Value |
|---|---|
| Base | `ghcr.io/opencharly/pod-crabbox-systemd:2026.308.2335` (CachyOS) |
| SSH | `sshd` on **2222**, public-key only, user `crabbox` |
| Sudo | `crabbox` has passwordless sudo |
| Inputs | read-only `/crabbox`: `authorized_keys`, `lease.env`, `bootstrap.sh` |
| Readiness | `/usr/local/libexec/crabbox/ready` — polled over SSH until exit 0 |
| PID 1 | `/sbin/init` → `/usr/lib/systemd/systemd` |

This variant adds the inner engine:

| Property | Value |
|---|---|
| Inner containerd | pinned `nerdctl-full` 2.1.2 unpacked into `/usr/local` |
| Inner CLI | `nerdctl` |
| Enablement | `containerd.service` enabled at boot |
| Rate limit | `ExecStartPre=-systemctl reset-failed containerd.service` before every start |
| Runtimes | `crun`, `krun` (`libkrun` + `libkrunfw` on the loader path) |
| GPU/Vulkan | `virglrenderer`; the `libbz2.so.1.0` soname libkrun requests |
| Inner network | `/etc/nerdctl/nerdctl.toml` `bridge_ip = "10.99.0.1/24"` |
| Clone identity | empty `/etc/machine-id`, `MACAddressPolicy=none` |
| Masked | `systemd-udevd.service`; `sys-kernel-{config,debug,tracing}.mount` |
| Not installed | `rtkit` |

The nerdctl-full archive is fetched from the release URL pinned by the
`NERDCTL_FULL_VERSION` var in `candy/containerd-nested/charly.yml` with charly's
`download:` verb, and the installed `nerdctl` must report that version in the
build check.

## Not provided

The plan's "docker-compatible socket for the Actions runner" has no image-side
mechanism in the sources this repo can cite. The package list in
`CONTAINERD-IMAGES.md` §2 names no Docker API daemon, and the org's documented
docker socket is the crabbox local-container provider's **host** socket
pass-through (`--local-container-docker-socket`, mounted at
`/var/run/docker.sock`), which is a consumer action. The image therefore ships
containerd + nerdctl and does not claim a Docker-API endpoint.

## How to use it

The image is run by the Crabbox `containerd` provider. It runs as a pod
(nested) and as a krun VM. Under crun/runc it needs the runtime's systemd mode;
under krun, the guest's PID-1 handoff:

```sh
# crun / runc
--systemd=always
# krun
-e KRUN_INIT_PID1=1 --ulimit nofile=524288:524288
```

The inner krun needs `/dev/kvm` passed to the outer lease. The lease inputs are
the same read-only `/crabbox` directory the base image uses.

## Layout

- `charly.yml` — the `pod-crabbox-nested` image (pinned base image + systemd entrypoint).
- `candy/containerd-nested/` — the inner engine, its units and configuration,
  and the `check:` steps that prove the row.
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `.github/workflows/image.yml` — builds on `pull_request` and publishes at the
  merge tag, so the image tag equals the git tag.
- `CHANGELOG/` — per-CalVer history.

## Related

- [`opencharly/pod-crabbox-systemd`](https://github.com/opencharly/pod-crabbox-systemd) —
  the systemd lease image this one imports.
- [`opencharly/pod-crabbox`](https://github.com/opencharly/pod-crabbox) — the lease
  base image.
- [`CONTAINERD-IMAGES.md`](https://github.com/opencharly/opencharly/blob/main/CONTAINERD-IMAGES.md)
  §2 — the lease-image contract and this image's row.
