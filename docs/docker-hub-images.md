# Existing Docker Hub images as Apple `container` machines

Tested on 2026-09-13 on Apple silicon, macOS 27.0, `container` 1.4.1. Each image
was booted as a real machine (`container machine create <image> --cpus 1
--memory 1G`), checked, then deleted.

## Short answer

- **Nothing on Docker Hub is built for Apple `container` machines.** Searches
  for "apple container", "container machine", "systemd debian", "init image"
  and similar return nothing aimed at it.
- **Stock Debian, Ubuntu, Fedora and Rocky Linux images cannot boot as
  machines.** They have no `/sbin/init`.
- **Some images do boot:** stock `alpine:3.22` and `almalinux:9`, plus several
  systemd images published for Ansible testing or as RHEL-family "init"
  variants.
- **None of them is a clean Debian 13 machine.** Each one fails at least one of
  the following:
  - a unique machine-id per machine
  - systemd reporting `running`
  - basic tools present
  - `sudo` installed

  For Debian, build from the official image
  ([bootstrap guide](bootstrap-debian-machine.md)) or use
  `xbill9/debian13-machine:latest`.

## Why the stock images don't work

Distribution images on Docker Hub are built for **containers**. The container
runtime starts one program, and nothing boots, so distributions strip out the
init system.

A **machine** boots. Apple mounts its own script at `/sbin.machine/init`, which
does a little setup and then runs `exec /sbin/init`. That path is hard-coded
(line 74 of the script). The image's `CMD` and `ENTRYPOINT` are ignored. With
no `/sbin/init`, the handoff fails, and the machine is created but stays
`stopped`:

```sh
container machine create debian:13 --name t1
container machine logs t1
```
```
/sbin.machine/init: 74: exec: /sbin/init: not found
```
```sh
container run --rm debian:13 sh -c 'ls -l /sbin/init; ls -ld /sbin'
```
```
ls: cannot access '/sbin/init': No such file or directory
lrwxrwxrwx 1 root root 8 Jul  4 09:05 /sbin -> usr/sbin
```

Checked with `ls` inside each stock image (only `debian:13`, `alpine` and
`almalinux:9` were also booted):

| Stock image | `/sbin/init` | As a machine |
|---|---|---|
| `debian:13` | missing | **does not boot** (log above) |
| `ubuntu:24.04` | missing | cannot boot; not tried |
| `fedora:latest` | missing | cannot boot; not tried |
| `rockylinux/rockylinux:9` | missing | cannot boot; not tried |
| `alpine:3.22` | BusyBox | boots (this is why Apple's docs use Alpine) |
| `almalinux:9` | systemd | boots |

Having `/sbin/init` is only the first requirement. Everything an image needs is
listed in
[README section 14](../README.md#14-what-an-image-needs-to-be-apple-container-machine-friendly).

## Boot test results

"Ready" is the time from `machine create` until the machine accepted a command.
"Missing" means which of `ps ip less curl bash sudo` were absent. All images are
multi-arch with an arm64 variant. In every image that booted, Apple's first-boot
script created your Mac user (uid 501).

| Image | Ready | Init / systemd | machine-id | Missing | Passwordless `sudo` |
|---|---|---|---|---|---|
| `debian:13` (stock) | never | none | — | — | — |
| `alpine:3.22` (stock) | 6 s | BusyBox init, no systemd | none (no systemd) | `curl bash sudo` | no, not installed |
| `almalinux:9` (stock) | 15 s | systemd `running` | unique | `ps ip sudo` | no, not installed |
| `geerlingguy/docker-debian13-ansible` | 15 s | systemd **`degraded`** (`systemd-modules-load`) | **shared** | `less curl` | **yes** |
| `geerlingguy/docker-ubuntu2404-ansible` | 16 s | systemd `running` | **shared** | `less curl` | **yes** |
| `buluma/debian-systemd:latest` (Debian 13) | 15 s | systemd `running` | **shared** | `ps ip less curl sudo` | no, not installed |
| `redhat/ubi9-init` | 4 s | systemd `running` | unique | `less sudo` | no, not installed |
| `almalinux/9-init` | 4 s | systemd `running` | unique | `ip less sudo` | no, not installed |
| `rockylinux/rockylinux:10-ubi-init` | 4 s | systemd `running` | **shared** | `ip less sudo` | no, not installed |
| `xbill9/debian13-machine:latest` (this repo) | 4 s | systemd `running` | unique | `sudo` (not in that build) | add with `-p sudo` |

**shared** means the image carries a machine ID (`/etc/machine-id`, and
`/var/lib/dbus/machine-id` in the Debian/Ubuntu ones). The booted machine
reported exactly that ID, so every machine created from the image gets the same
one:

| Image | ID stored in the image = ID of the booted machine |
|---|---|
| `geerlingguy/docker-debian13-ansible` | `436263104bb5427ea2c328b74e306304` |
| `geerlingguy/docker-ubuntu2404-ansible` | `90e14af3e9084a30b23204d8df448142` |
| `buluma/debian-systemd:latest` | `9275107109894547b4b1ee9cfad473e2` |
| `rockylinux/rockylinux:10-ubi-init` | `38721593dffe4453bd239ab26e18abc0` (in `/etc` only) |

Other things seen:

- **`degraded`** on the Geerling Debian 13 image comes from
  `systemd-modules-load.service`. Apple's kernel ships no loadable modules, so
  that unit fails unless it is masked (README section 7). The same unit did not
  fail on the others.
- **Group name:** on the RHEL-family images, your Mac gid 20 already exists as
  the group `games`, so `id` shows `gid=20(games)`. Apple's script only adds a
  group when the gid is free. This is cosmetic.
- **sudo:** Apple's first boot always writes `/etc/sudoers.d/<user>` with
  `NOPASSWD:ALL`, so sudo works wherever the `sudo` package is installed (the
  two Geerling images).
- **Boot time:** the RHEL-family init images and this repo's image were ready in
  about 4 s; the others took about 15 s.

## Not tested

| Image | Why |
|---|---|
| `jrei/systemd-debian`, `jrei/systemd-ubuntu` | amd64 only; no arm64 variant |
| `nggit/systemd-debian` | arm64, but last updated 2023 and stops at bookworm (Debian 12) |
| `kindest/node` | A ~1 GB Kubernetes-in-Docker node image, not a general OS |
| `geerlingguy/docker-debian12-ansible`, `docker-rockylinux9-ansible`, `redhat/ubi10-init`, `almalinux/10-init` | Same families as images tested above |

## If you want to use one anyway

Inside a machine made from an image marked **shared**, give it its own ID and
restart:

```sh
container machine run -n <n> --root -- 'rm -f /etc/machine-id /var/lib/dbus/machine-id && systemd-machine-id-setup'
container machine stop <n>
```

Or layer the fixes from the bootstrap guide on top of the image. This example
was not built; the lines are the ones tested in step 3 of the guide:

```dockerfile
FROM geerlingguy/docker-debian13-ansible:latest
RUN apt-get update && apt-get install -y --no-install-recommends less curl \
    && rm -rf /var/lib/apt/lists/*
RUN systemctl mask systemd-modules-load.service
RUN : > /etc/machine-id && rm -f /var/lib/dbus/machine-id
```

Which to pick:

- **Debian 13:** build from the official `debian:13`
  ([bootstrap guide](bootstrap-debian-machine.md)), or pull
  `xbill9/debian13-machine:latest`.
- **RHEL-family:** `almalinux/9-init` or `redhat/ubi9-init` boot cleanly with a
  unique ID. Add `sudo` and `less` if you want them.
- **Smallest:** `alpine:3.22` boots in seconds, but has no systemd.

## How this was tested

For each image:

```sh
container run --rm <img> sh -c 'ls -l /sbin/init; cat /etc/machine-id /var/lib/dbus/machine-id'
container machine create <img> --name t --cpus 1 --memory 1G
until container machine run -n t --root -- true 2>/dev/null; do sleep 2; done   # gave up after 60 s
sleep 15
container machine run -n t --root -- '. /etc/os-release; echo "$PRETTY_NAME"; cat /proc/1/comm; systemctl is-system-running; systemctl --failed; cat /etc/machine-id; for t in ps ip less curl bash sudo; do command -v $t || echo "missing $t"; done'
container machine run -n t -- 'id; sudo -n true && echo sudo-ok'
container machine stop t; container machine delete t
```
