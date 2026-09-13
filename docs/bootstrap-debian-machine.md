# Bootstrap a Debian 13 machine from the official image

Every command below was run on 2026-09-13 on this Mac (Apple silicon,
macOS 27.0, `container` 1.4.1), starting from a freshly pulled `debian:13`.
The output shown is what it actually printed.

## Do you even need this?

Apple's `container` runs Linux in two ways:

| | **Container** | **Machine** |
|---|---|---|
| What runs | one program from the image | a whole booted Linux system (systemd) |
| Disk | throwaway | persistent |
| Your Mac home folder | not mounted | mounted at `/Users/<you>` |
| You are | root | your own Mac user (uid 501) |
| Official `debian:13` works? | **yes, as-is** | **no** |

If a container is enough, stop here. The official image works directly (use a
real Terminal window for `-it`):

```sh
container run -it --rm debian:13 bash
```

A machine *boots* the image. Booting means running `/sbin/init`, and the
official Debian image has none: it is built for containers, which never boot.
`container machine create debian:13` creates a machine that never starts. The
fix is a small image on top of `debian:13` that adds an init system and
handles three smaller problems. Details are in
[README section 14](../README.md#14-what-an-image-needs-to-be-apple-container-machine-friendly).

Other Docker Hub images do boot as machines, but none is a clean Debian 13
machine. See [docker-hub-images.md](docker-hub-images.md) for boot tests of
nine of them.

## 0. Prerequisites

`container` is installed (README section 1) and its services are running:

```sh
container system status                     # status  running
container system start                      # if it is not
container system kernel set --recommended   # once; no machine runs without a kernel
```

## 1. Pull the official image

```sh
container image pull docker.io/library/debian:13
```

It is multi-arch, and `container` picks `linux/arm64` on Apple silicon:

```sh
container image inspect debian:13 | jq -c '[.[0].variants[].platform | .os + "/" + .architecture]'
# ["linux/amd64",...,"linux/arm64",...]
```

## 2. See why it cannot boot

```sh
container run --rm debian:13 sh -c 'ls -l /sbin/init; command -v ps ip less sudo; echo done'
```
```
ls: cannot access '/sbin/init': No such file or directory
done
```

There is no init, and none of `ps`, `ip`, `less` or `sudo` exist either.

## 3. Write the Dockerfile

In an empty directory, for example `~/debian-machine/`, create `Dockerfile`:

```dockerfile
FROM debian:13
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        systemd-sysv dbus sudo ca-certificates procps less iproute2 iputils-ping curl \
    && rm -rf /var/lib/apt/lists/*
RUN systemctl mask systemd-modules-load.service
RUN : > /etc/machine-id && rm -f /var/lib/dbus/machine-id
CMD ["/sbin/init"]
```

| Line | Why |
|---|---|
| `systemd-sysv` | Provides `/sbin/init`, the one hard requirement |
| `procps less iproute2 iputils-ping curl ca-certificates` | The basic tools the official image leaves out |
| `sudo` | First boot already gives your Mac user passwordless sudo; only the package is missing |
| `dbus` | The system message bus that systemd services use |
| `systemctl mask systemd-modules-load.service` | Apple's kernel has no loadable modules, so this unit always fails and systemd reports `degraded` |
| `: > /etc/machine-id && rm -f /var/lib/dbus/machine-id` | Installing dbus writes a machine ID into the image. Without this line **every machine from the image shares one ID** |
| `CMD ["/sbin/init"]` | Ignored by machines (they always run `/sbin/init`), but marks this as a machine image |

## 4. Build it

The builder is not running by default, and it holds 2 CPUs and 2 GB while it is:

```sh
cd ~/debian-machine
container builder start
container build -t debian13-machine .
container builder stop
```

The build took about 10 seconds here. Ignore `debconf: ... Readline` lines; they
only mean there is no terminal.

## 5. Check the image before creating a machine

```sh
container run --rm debian13-machine sh -c 'ls -l /sbin/init; wc -c < /etc/machine-id; ls /var/lib/dbus/machine-id'
```
```
lrwxrwxrwx 1 root root 22 Apr 13 19:38 /sbin/init -> ../lib/systemd/systemd
0
ls: cannot access '/var/lib/dbus/machine-id': No such file or directory
```

You want: `/sbin/init` present, a 0-byte machine-id, and no dbus copy.

## 6. Create the machine

```sh
container machine create debian13-machine --name dev-vm --cpus 2 --memory 2G
```

`create` returns in about a second, **before the VM has booted**. A command sent
straight away fails with a misleading error:

```
Error: The operation couldn’t be completed. Operation not supported on socket
```

(other times: `... Operation not supported by device`). Wait until it answers
(13 seconds here):

```sh
until container machine run -n dev-vm --root -- true 2>/dev/null; do sleep 2; done
```

Then give systemd a few more seconds to finish starting.

## 7. Check the machine

```sh
container machine run -n dev-vm -- '. /etc/os-release; echo "$PRETTY_NAME"; uname -srm; systemctl is-system-running; id; sudo -n id -u; nproc'
```
```
Debian GNU/Linux 13 (trixie)
Linux 6.18.35 aarch64
running
uid=501(xbill) gid=20(dialout) groups=20(dialout)
0
2
```

- `running`, not `degraded` or `initializing`: systemd is fully up.
- `id`: you are your Mac user. It was created in the VM on first boot.
- `sudo -n id -u` printed `0`: passwordless sudo works.
- Your Mac home folder is mounted at `/Users/<you>`. Inside the VM, `$HOME` is
  `/home/<you>`.

Pass a command as **one quoted argument**. The machine joins all arguments into
one string and re-parses it with bash, so `-- sh -c '...'` silently runs the
wrong thing.

Interactive shells need a real Terminal window. They fail from scripts and
agent tools with the same "not supported" error:

```sh
container machine run -n dev-vm            # shell as you
container machine run -n dev-vm --root     # shell as root
```

## 8. Customize it

Anything you install lives on the machine's own disk:

```sh
container machine run -n dev-vm --root -- 'apt-get update && apt-get install -y --no-install-recommends git vim-tiny'
```

It survives a restart:

```sh
container machine stop dev-vm                         # takes about 10 s
container machine run -n dev-vm -- 'git --version'    # boots it again
# git version 2.47.3
```

After a restart, `machine run` answers before systemd is up. `systemctl` said
`Failed to connect to system scope bus` for the first few seconds, so wait
before using it.

Deleting the machine deletes these changes. To keep them, add the packages to
the Dockerfile and rebuild, or snapshot the machine into an image (next section).

## 9. Optional: publish it

```sh
container registry login docker.io
container image tag debian13-machine docker.io/<you>/debian13-machine:latest
container image push docker.io/<you>/debian13-machine:latest
```

This is `container image push`; there is no `container push`. It printed no
progress for about 10 minutes on Wi-Fi before finishing. To publish a machine
you customized by hand instead, use `bin/mkdebian-machine publish dev-vm <ref>`
(README section 11). It snapshots the machine and removes your user, keys and
logs first.

## The short way

Steps 3–7 are what `bin/mkdebian-machine` does:

```sh
bin/mkdebian-machine new -t debian13-machine -m dev-vm -p sudo
```

## When something goes wrong

| Symptom | Cause | Fix |
|---|---|---|
| Machine is created but stays `stopped`; `container machine logs <n>` shows `exec: /sbin/init: not found` | Image has no init (plain `debian:13`) | Build the image from step 3 |
| `Operation not supported on socket` / `by device` | Machine still booting, or an interactive shell without a real terminal | Wait (step 6); pass a command, or use a real Terminal window |
| `systemctl is-system-running` → `degraded` | `systemd-modules-load.service` failed | `systemctl mask` it (step 3); harmless otherwise |
| `initializing`, or `Failed to connect to system scope bus` | systemd still starting | Wait a few seconds |
| Two machines report the same `/etc/machine-id` | Image carries `/var/lib/dbus/machine-id` | Last Dockerfile line in step 3; inside an existing machine: `rm -f /etc/machine-id /var/lib/dbus/machine-id && systemd-machine-id-setup`, then restart |
| `sudo: command not found` | Official image has no sudo | Add `sudo` to the package list, or use `--root` |
| A quoted command prints nothing or does the wrong thing | Arguments are joined and re-parsed by bash | Pass the whole command as one quoted argument |
| `Plugin 'container-images' not found` | Typed `container images ls` | `container image ls` |

## Clean up

```sh
container machine stop dev-vm
container machine delete dev-vm
container image rm debian13-machine    # a Debian machine image unpacks to about 1.4 GB
```
