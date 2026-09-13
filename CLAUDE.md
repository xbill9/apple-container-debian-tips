# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Notes and tooling for running Debian 13 under Apple's `container` CLI (1.4.1) on Apple silicon. `README.md` is a verified log of what was actually run on this Mac; `bin/mkdebian-machine` automates building a machine-bootable Debian image.

## Keep README.md in sync

Every change to `bin/mkdebian-machine` or the Dockerfile, and every new gotcha found, gets written up in the relevant README section. Only record things that were actually run and verified.

## No TTY in Claude's tool calls

Interactive commands fail or hang here. Never use:
- `container exec -it` → `Operation not supported by device` (drop `-it`)
- `sudo` → Debian images don't have it (`command not found`); use `container machine run -n <n> --root -- <cmd>`. (If the image has sudo, `sudo -n` works: first boot grants the host user NOPASSWD.)
- `container machine run` with no command → same TTY error
- `sudo installer` or `container system start` if it prompts for the kernel (use `container system kernel set --recommended`)

If a user needs an interactive shell, give them the command to run in a real terminal.

## `container` CLI quirks (1.4.1)

- `container image ls`, not `images ls` (fails with a misleading `Plugin 'container-images' not found`).
- `machine run` joins its args and re-parses them with `/bin/bash -c` in the VM, so host quoting is lost. Pass a shell snippet as ONE argument: `container machine run -n <n> -- 'ps aux | head'`. Never `-- sh -c '...'` — it silently runs the wrong command.
- `container builder status` exits 0 even when stopped — grep the output for `not running|stopped`. `container build` auto-starts a stopped builder but never stops it; stop it after if it was stopped before.
- `container run --rm` does not reliably remove the container; check `container ls -a` afterwards.
- No `container commit` and no container→machine conversion; go through `container export` + a `FROM scratch` image.
- No `container push` (`Plugin 'container-push' not found`) — it is `container image push [--scheme http] <ref>`.
- `container machine create` returns before the VM accepts commands; a `machine run` right after it fails with the misleading error `Operation not supported by device` or `... on socket`. Poll `machine run -- true` first. After a restart, `machine run` works before systemd is up (`Failed to connect to system scope bus`); wait a few seconds before `systemctl`.
- `docs/bootstrap-debian-machine.md` is the user-facing step-by-step guide; keep its commands and outputs in sync with README when behavior changes.
- `docs/docker-hub-images.md` records dated boot tests of third-party images. Re-test (throwaway machines, `--cpus 1 --memory 1G`, at most ~3 at once on this 8 GB Mac) before changing its results. `readlink -f /sbin/init` prints a path even when init is missing; check with `ls`.
- `machine create` takes `--cpus`/`--memory`/`--home-mount` directly; no `machine set` + restart needed.
- There is no `machine export`. A machine is snapshotted by tarring its own root into the rw home mount (see `publish` in the script).
- Port 5000 on the host is taken by macOS AirPlay Receiver (answers 403); use another port for a local registry.
- Images built here are arm64-only.

## Machines

- README section 14 is the checklist of what makes an image machine-friendly, including what Apple's `/sbin.machine/init` and `create-user.sh` do. Keep it current.
- A machine image must contain `/sbin/init` (plain `debian:13` does not — the machine is created but never boots). Verify with `container run --rm <image> sh -c 'ls -l /sbin/init'` before `container machine create`.
- `container machine logs <n>` is the first place to look when a machine will not boot.
- `systemctl is-system-running` = `degraded` from `systemd-modules-load.service` is expected unless that unit is masked (the script masks it).
- Any machine image that installs dbus must end with `: > /etc/machine-id && rm -f /var/lib/dbus/machine-id`, or systemd copies the build-time dbus ID into every machine.
- `systemctl is-system-running` says `initializing` for several seconds after first boot; wait before judging it.

## Testing

Claude may run real `container` builds and create/delete machines to test changes, but use throwaway names and clean up afterwards (see `/test-mkdebian-machine`). Do not touch the existing `debian13-vm` machine or `debian13-systemd` image. Never push to Docker Hub or any real registry without being asked — test `publish` against a local `registry:2` container on port 5050 with `--scheme http`.
