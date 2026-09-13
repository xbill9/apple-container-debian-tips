# Apple `container` + Debian 13 — Setup Notes

Working notes from setting this up on 2026-09-13. Everything here was run and
verified on this machine.

**Host:** Apple silicon (arm64), macOS 27.0 (build 26A428)
**CLI:** `container` 1.4.1, installed to `/usr/local/bin/container`

**Start here:** [docs/bootstrap-debian-machine.md](docs/bootstrap-debian-machine.md)
is a step-by-step guide from a fresh official `debian:13` pull to a working
machine, including why the official image cannot boot as one. The sections
below are the detailed log behind it.

---

## 1. Installing the CLI

Download the signed `.pkg` from https://github.com/apple/container/releases
(1.4.1 was `container-1.4.1-installer-signed.pkg`, ~112 MB).

Verify it before installing:

```sh
pkgutil --check-signature container-1.4.1-installer-signed.pkg
# Expect: Developer ID Installer: Apple Inc. - Containerization (UPBK2H6LZM)
#         Notarization: trusted by the Apple notary service
```

Install it. **`sudo installer` needs a real TTY** — it fails from any
non-interactive context with "a terminal is required to read the password".
Either run it in a real Terminal window, or just launch the GUI installer:

```sh
open container-1.4.1-installer-signed.pkg
```

Then start the service:

```sh
container system start
```

### Gotcha: the kernel prompt

`container system start` stops and asks to download a default Linux kernel.
That prompt cannot be answered non-interactively. Do it explicitly instead:

```sh
container system kernel set --recommended
```

This pulls the Kata Containers kernel (3.32.0 arm64). Without it, nothing runs.

---

## 2. Containers vs. Machines — they are NOT the same thing

This distinction caused the most confusion, so it is worth being precise.

| | **Container** | **Machine** |
|---|---|---|
| What it is | One process in a lightweight VM | A persistent Linux VM |
| Created from | An image, via `container run` | An image, via `container machine create` |
| Needs `/sbin/init`? | **No** | **Yes** — it actually boots |
| Home dir mounted | No | Yes — `/Users/<you>` is there |
| Default user | root | your host user (uid 501) |
| Lists with | `container ls -a` | `container machine ls` |

**You cannot convert a container into a machine.** There is no such command,
and there is no `container commit` in 1.4.1 either. A machine is always created
from an *image*.

To carry a container's modifications over, you have to go through an image:

```sh
container export debian13 -o rootfs.tar
printf 'FROM scratch\nADD rootfs.tar /\n' > Dockerfile
container build -t debian13-snapshot .
container machine create debian13-snapshot --name some-vm
```

---

## 3. The big gotcha: `debian:13` cannot boot as a machine

```sh
container machine create debian:13 --name debian13-vm   # creates, but never boots
container machine logs debian13-vm
# /sbin.machine/init: 74: exec: /sbin/init: not found
```

Standard Debian container images ship **no init system** — containers run a
single process, so they do not need one. A machine boots a kernel and does.
(This is why Apple's own docs use `alpine:3.22`: BusyBox provides `/sbin/init`.)

The machine is created in `stopped` state and will never start. Delete it:

```sh
container machine delete debian13-vm
```

### The fix: build a Debian image that has an init

See `build/Dockerfile.debian13-systemd`. `systemd-sysv` is the package that
provides `/sbin/init`.

```sh
container builder start                    # `container build` auto-starts a stopped builder, but never stops it
cd build
container build -t debian13-systemd -f Dockerfile.debian13-systemd .
```

**Always verify before creating the machine** — this is the check I skipped the
first time:

```sh
container run --rm debian13-systemd sh -c 'ls -l /sbin/init'
# /sbin/init -> ../lib/systemd/systemd
```

Then:

```sh
container machine create debian13-systemd --name debian13-vm
```

Result: Debian 13 (trixie), kernel 6.18.35 aarch64, 4 CPUs, 4 GB, home mounted.

---

## 4. Getting root in a machine

**Use `--root`. Do not use `sudo`.**

```sh
container machine run -n debian13-vm --root          # root shell
container machine run -n debian13-vm --root -- id    # uid=0(root) gid=0(root)
```

By default `machine run` matches your *host* user — you land as
`uid=501(xbill) gid=20(dialout)`.

Your user *is* allowed passwordless sudo: on first boot Apple's setup script
writes `/etc/sudoers.d/xbill` with `NOPASSWD:ALL` (section 14). What is missing
is the `sudo` package — Debian images do not ship it (`sudo: command not found`).
Install it (`mkdebian-machine new -p sudo`) and sudo works with no password:
`container machine run -n <n> -- sudo -n id -u` printed `0` (verified
2026-09-13). An earlier note here said sudo hung on a password prompt; that was
not reproduced.

---

## 5. TTY limitations (this bites constantly)

Anything interactive needs a **real terminal**. Neither agent tool calls nor
Claude Code's `!` prefix allocate a TTY. Symptoms:

- `container exec -it <name> bash` → `NSPOSIXErrorDomain Code=19 "Operation not supported by device"`
- `sudo` → `a terminal is required to read the password`
- `container machine run` with no command → same TTY error

Open a real window instead:

```sh
osascript -e 'tell application "iTerm2" to create window with default profile command "/usr/local/bin/container machine run -n debian13-vm --root"'
```

Non-interactive commands work fine without a TTY — only drop `-it`.

---

## 6. Command syntax notes

- **`container image ls`, not `container images ls`.** The plural form fails
  with a misleading `Plugin 'container-images' not found`.
- **`machine run` joins your arguments with spaces and re-parses them with
  `/bin/bash -c`** inside the machine. Your host-side quoting is lost:
  ```sh
  container machine run -n mkdm-test-vm --root -- echo '$0' 'a    b'
  # /bin/bash a b          ($0 expanded in the VM, spaces collapsed)
  ```
  So `-- sh -c 'command -v ps less'` silently runs the wrong thing (`sh -c command`
  with `-v` as `$0`) and prints nothing. Pass the whole shell snippet as **one
  argument** instead — it is already run by bash:
  ```sh
  container machine run -n debian13-vm -- 'ps aux | head'
  container machine run -n debian13-vm -- 'for t in ps ip; do command -v "$t"; done'
  ```
  The cause is Apple's `/sbin.machine/init`, which runs
  `exec <login shell> -c "$*"` — `$*` joins the arguments with spaces (section 14).
- **`container rm <id>`** removes a container; **`container image rm <ref>`**
  removes the image. Separate things. Removing `alpine:latest` reclaimed
  **1.17 GB** — far more than the ~8 MB download, because that figure is the
  compressed size, not the unpacked rootfs plus per-image VM overhead.
- `--rm` on `container run` did **not** reliably remove the container; a stopped
  one was left behind. Check `container ls -a` and clean up.

---

## 7. Expected noise, safely ignored

- `systemctl is-system-running` reports **`degraded`**, with
  `systemd-modules-load.service` failed. Expected: Apple supplies a prebuilt
  kernel with no matching modules to load.
- `debconf: unable to initialize frontend: Readline / Teletype` during
  `apt-get install` — just the absence of a TTY. It falls back to
  Noninteractive and succeeds.

---

## 8. Minimal image — tools you will want

Debian container images are stripped. `ps`, `free`, `ip`, `less` are all absent.

```sh
container machine run -n debian13-vm --root -- apt-get update
container machine run -n debian13-vm --root -- apt-get install -y procps less vim-tiny iproute2
```

Installed into the machine's own disk, so it persists across restarts — but it
is **lost if the machine is deleted**. To make it permanent, add the packages to
the Dockerfile and rebuild the image.

---

## 9. Current state on this Mac

Checked 2026-09-13 with `container ls -a`, `container machine ls`,
`container image ls` and `container registry ls`.

| Thing | Name | Notes |
|---|---|---|
| Machine | `debian13-vm` | from `debian13-systemd`, running, the default, **2 CPUs / 2 GB** (not the 4 / 4 GB of section 3), home mounted rw |
| Images | `debian13-systemd:latest`, `debian:13` | |
| Containers | none of mine | the old `debian13` (`sleep infinity`) container is gone |
| Builder | `buildkit` | stopped; `container build` starts it on demand |
| Registry logins | `xbill9` @ `registry-1.docker.io` | from `container registry login docker.io` (section 12) |
| Docker Hub | `xbill9/debian13-machine:latest` | public, linux/arm64, built with `mkdebian-machine new` (section 12) |
| GUI | `~/AppleContainerDesktop` | Tauri app, built locally, unsigned |
| MCP server | `~/AppleContainerMCP` | registered with Claude Code at user scope (section 13) |
| shellcheck | 0.11.0, Homebrew | runs from this repo's Claude Code hook on edits to `bin/` |

Shells:

```sh
container machine run -n debian13-vm                 # machine, as xbill
container machine run -n debian13-vm --root          # machine, as root

# a throwaway container again, if you want one
container run -d --name debian13 debian:13 sleep infinity
container exec debian13 cat /etc/os-release          # no -it without a real terminal
```

### Version drift worth remembering

Both `AppleContainerDesktop` and `AppleContainerMCP` are built and validated
against container CLI **1.2.2**, but this Mac runs **1.4.1**. I checked every
JSON command the GUI depends on (`ls`, `image ls`, `machine ls`, `volume ls`,
`network ls`, `system status`, `builder status` — all with `--format json`) and
all seven work on 1.4.1. The MCP server's `check_environment` may still warn
about the version, and newer 1.3/1.4 flags are not wrapped by either project.

---

## 10. Useful commands

```sh
# System
container system start / status
container system kernel set --recommended

# Containers
container run -d --name <n> <image> sleep infinity
container ls -a
container exec <name> <cmd>              # non-interactive, no TTY needed
container logs <name>
container rm <id>
container prune                          # remove all stopped

# Images
container image ls
container image rm <ref>
container build -t <tag> .
container export <container> -o out.tar
container image tag <src> <ref>
container image push [--scheme http] <ref>   # there is no `container push`

# Machines
container machine ls
container machine create <image> --name <n> [--cpus 4 --memory 8G]   # returns before it boots
container machine run -n <n> [--root] [-- cmd]
container machine logs <n>               # first stop when boot fails
container machine set -n <n> cpus=4 memory=8G home-mount=ro   # needs restart
container machine stop <n> / delete <n>

# Builder (needed before `container build`)
container builder status / start
```

---

## 11. `bin/mkdebian-machine` — two workflows

```sh
~/apple-container-debian-tips/bin/mkdebian-machine --help
```

### Way 1 — `new`: from nothing to a customized machine

Wraps section 3 into one command: generates the Dockerfile, starts the builder
if needed (and stops it again), builds, **verifies `/sbin/init` before you
commit to it**, creates the machine, waits for it to boot, then runs your own
setup script inside it as root.

```sh
# setup-dev.sh is any shell script; it runs inside the machine as root
mkdebian-machine new -m dev-vm -p "git vim" --setup ./setup-dev.sh

# Debian 12, dev tools, 4 CPUs / 8 GB
mkdebian-machine new -d 12 -p "build-essential git vim" -m build-vm --cpus 4 --memory 8G
```

`new` is the default, so the old form `mkdebian-machine -m dev-vm` still works.
`-p` packages are baked into the image; `--setup` changes only that machine.
Keep customizing by hand afterwards with `container machine run -n dev-vm --root`.

Beyond `systemd-sysv` it installs `procps less iproute2 iputils-ping curl
ca-certificates dbus`, and **masks `systemd-modules-load.service`** so systemd
no longer reports `degraded` (section 7).

For sudo, `-p sudo` is enough: first boot already grants your host user
passwordless sudo (section 4). `--sudo-uid $(id -u)` additionally bakes a
`#<uid>` sudoers rule into the image itself — rarely needed, and it weakens
every machine made from that image.

### Way 2 — `publish`: a working machine or container to Docker Hub

```sh
container registry login docker.io
mkdebian-machine publish dev-vm docker.io/youruser/debian-machine:13
```

Give it a **machine** or a **container** by name; it works out which.

- **Machine:** there is no `machine export`, so the machine tars its own root
  disk (live, `--one-file-system`) into a work directory under `~/.cache` —
  machines mount your home folder, so the host can read the file. Needs
  `home-mount=rw`. The image is built `FROM scratch` and scrubbed of what
  `container machine create` added for your host user (its `/etc/passwd` entry,
  `/etc/sudoers.d/<user>`, `/home/<user>`), SSH host keys, apt lists, logs and
  root's shell history. Removing the user is what makes a machine created from
  the image run first-boot user setup again — it runs only when `id <user>`
  fails (section 14). `/etc/.machine.initialized` is a per-machine mount and is
  left out.
- **Container:** `container export`, plus the container's command, environment
  and working directory from `container inspect`, so the image starts the way
  the container did.

`--no-push` builds and tags only. `--scheme http` targets a plain-HTTP registry.

Verified round trip on 2026-09-13, against a throwaway local registry rather
than Docker Hub:

```sh
container run -d --name mkdm-registry -p 5050:5000 docker.io/library/registry:2
mkdebian-machine new -t mkdm-new -m mkdm-new-vm --cpus 3 --memory 3G --setup setup.sh
mkdebian-machine publish mkdm-new-vm localhost:5050/mkdm-snap:1 --scheme http   # 243 MB, ~15 s
container image rm localhost:5050/mkdm-snap:1                                   # force a real pull
container machine create localhost:5050/mkdm-snap:1 --scheme http --name mkdm-pub-vm
```

`mkdm-pub-vm` booted with the setup script's changes intact (a marker file,
`jq`), no SSH host keys, and the host user re-created by first boot. Publishing
the registry *container* the same way and running the result on port 5051
served `/v2/` normally.

**Port 5000 is taken on macOS** by AirPlay Receiver (it answers 403), hence 5050.

### Bugs found while testing it

`container builder status` **exits 0 even when the builder is not running**, so
`if ! container builder status` never fires. Match on the output text instead —
and on 1.4.1 a stopped builder prints a table row with STATE `stopped`, not
"not running", so match both:

```sh
if container builder status 2>&1 | grep -Eqi 'not running|stopped'; then ...
```

The first version only matched `not running`. Found by `/test-mkdebian-machine`
on 2026-09-13: with the builder stopped, the check never fired, `container build`
started the builder by itself, and the script left it running (2 CPUs / 2 GB).

**`machine create` returns before the VM accepts commands.** A `machine run`
straight afterwards fails with the TTY error from section 5,
`Operation not supported by device`, and works about 3 s later. The script now
polls `container machine run -n <n> --root -- true` first. The original version
never noticed, because it applied CPUs and memory with `machine set` + stop +
`machine run -- true || true`, which swallowed the error;
`machine create --cpus --memory` does that directly.

**`tar` of a live machine:** excluding `./sys/*` is not enough — tar still
reads the `/sys` mountpoint and exits 1 with `./sys: file changed as we read it`.
Exclude `./proc ./sys ./dev ./run ./tmp` as whole directories and recreate them
in the image.

**Every machine had the same machine-id** (`a87018e5…`), across images built
separately and even from a snapshot whose `/etc/machine-id` was emptied.
Installing `dbus` writes a real `/var/lib/dbus/machine-id` at build time, and
systemd copies it into an empty `/etc/machine-id` at boot. Both the `new`
Dockerfile and the `publish` scrub now run
`: > /etc/machine-id && rm -f /var/lib/dbus/machine-id`. Verified afterwards:
two machines from one image and one from the published snapshot got three
different IDs, all with systemd `running`.

Images built before the fix still hand out one shared ID. `debian13-systemd`
does: it carries `f4958da6…` in both `/etc/machine-id` and
`/var/lib/dbus/machine-id`, so `debian13-vm` had that ID and every other machine
created from the image would too. Inside such a machine, as root, then restart
it so journald and dbus pick up the new ID:

```sh
container machine run -n <n> --root -- 'rm -f /etc/machine-id /var/lib/dbus/machine-id && systemd-machine-id-setup'
container machine stop <n>
```

Done on `debian13-vm` 2026-09-13: `f4958da6…` → `c311f41b…`, kept across the
restart. After boot `/var/lib/dbus/machine-id` is a symlink to
`/etc/machine-id`, and journald writes to `/var/log/journal/c311f41b…`; the old
logs stay in `/var/log/journal/f4958da6…` (read them with `journalctl -D`). The
machine's IP changed on restart (`.12` → `.59`).

---

## 12. Publishing images to Docker Hub

Yes — these are ordinary OCI images, nothing Apple-specific in the format.
`mkdebian-machine publish` (section 11) does all of this. By hand:

```sh
container registry login docker.io          # Docker Hub username + access token
container image tag debian13-machine docker.io/<youruser>/debian-machine:13
container image push docker.io/<youruser>/debian-machine:13
```

**There is no `container push` in 1.4.1** — it fails with
`Plugin 'container-push' not found`. It is `container image push`. For a
plain-HTTP registry add `--scheme http`, both to `image push` and to
`machine create` when pulling back.

Done for real on 2026-09-13:

```sh
printf '%s' "$TOKEN" | container registry login docker.io --username xbill9 --password-stdin
mkdebian-machine new -t debian13-machine --push docker.io/xbill9/debian13-machine:latest
container image rm docker.io/xbill9/debian13-machine:latest debian13-machine
container machine create docker.io/xbill9/debian13-machine:latest --name mkdm-hub-vm
```

- The login is stored as **`registry-1.docker.io`**, not `docker.io`
  (`container registry ls`).
- 7 blobs, 70.8 MB, **10 min 27 s** over Wi-Fi — and `--progress plain` printed
  **nothing** until it finished. It is not hung; `nettop -P` shows
  `container-core-images` sending.
- Pulled back into a fresh machine, it booted in ~4 s: Debian 13, systemd
  `running`, a fresh machine-id, the host user created on first boot.
- The repository was created public automatically (`is_private: false`).

Use an **access token** from Docker Hub → Account Settings → Personal access
tokens, not your password.

Caveats:
- The repo namespace must match your Docker Hub username, and the repo must
  exist or your account must allow auto-create on push.
- Images built here are **arm64 only**. On Docker Hub they will be pulled by
  x86 users who will get an architecture mismatch. `container build` has no
  multi-arch buildx equivalent; publish as arm64-only and say so, or build the
  amd64 variant separately.
- Pushing publishes it to the world on a free Docker Hub account. Make the
  repo private first if that is not what you want.
- A published *machine* snapshot contains everything you installed or wrote in
  it. The scrub removes only the items listed in section 11 — check for tokens,
  keys and credentials in `/root`, `/etc` and `/opt` before pushing.
- `systemd-sysv` in the image is unusual for a *container* image. Anyone using
  it as a plain container gets systemd as PID 1, which is not what they expect.
  Document it as a machine image.

---

## 13. MCP server registered with Claude Code

`~/AppleContainerMCP` is wired in at **user scope** (all projects), recorded in
`~/.claude.json`:

```sh
claude mcp add apple-container --scope user -- /usr/bin/env \
  FASTMCP_SHOW_SERVER_BANNER=false /opt/homebrew/bin/uv \
  --directory /Users/xbill/AppleContainerMCP run --quiet apple-container-mcp

claude mcp list     # apple-container: ✔ Connected
```

Dependencies installed with `uv sync` in that directory. Verified over stdio —
an `initialize` handshake returns `serverInfo: apple-container-mcp`.

**MCP servers are loaded when a session starts**, so a running session will not
see the tools until it is restarted.

---

## 14. What an image needs to be Apple `container` machine friendly

Nothing on Docker Hub targets `container machine` (searched 2026-09-13); the
one Debian machine image found elsewhere, `ghcr.io/mikluko/machine-debian`,
ships a fixed machine-id in its files. Stock
Debian, Ubuntu, Fedora and Rocky images have no `/sbin/init` and cannot boot.
A handful of images do boot — stock `alpine:3.22` and `almalinux:9`, plus
several systemd images — but each fails at least one item below; four of them
give every machine the same machine-id. Boot-tested in
[docs/docker-hub-images.md](docs/docker-hub-images.md). This is the checklist,
from what 1.4.1 does at boot and from what broke while testing here.
`mkdebian-machine new` applies all of it.

### What a machine does with your image

Every machine gets Apple's scripts mounted read-only at `/sbin.machine/`
(read them with `container machine run -n <n> --root -- cat /sbin.machine/init`):

- **Boot:** `/sbin.machine/init` writes the machine name to `/etc/hostname`,
  hands the forwarded SSH agent socket (`/var/host-services/ssh-auth.sock`) to
  your uid, then runs **`exec /sbin/init`** — hard-coded. The image's `CMD` and
  `ENTRYPOINT` are never used.
- **First boot:** if `id <host user>` fails, it runs `/etc/machine/create-user.sh`
  when the image has one, otherwise the built-in `/sbin.machine/create-user.sh`.
  The built-in one appends your user to `/etc/group`, `/etc/passwd` and
  `/etc/shadow` directly (no `useradd`), copies `/etc/skel` into
  `/home/<user>`, and writes **`/etc/sudoers.d/<user>` with `NOPASSWD:ALL`**.
  Then it writes `/etc/.machine.initialized`.
- **`machine run`:** runs `exec <your login shell> -c "$*"` — the shell from
  `/etc/passwd`, whose default on Debian/Ubuntu is `DSHELL` from
  `/etc/adduser.conf`. `"$*"` is why your arguments are joined and re-parsed
  (section 6).

### Required

| Need | Why | Without it |
|---|---|---|
| **`/sbin/init`** | Apple's init `exec`s it unconditionally | Machine is created but never starts: `exec: /sbin/init: not found` (section 3). Debian/Ubuntu: install `systemd-sysv`. |
| **arm64 variant** | Apple silicon; `machine create` defaults to `--arch arm64` | Pick another image. Images built here with `container build` are arm64. |
| **`/bin/sh`, `getent`, `cp`, `chown`, `mkdir`, `tr`** | used by the first-boot user script | Script requirement; a distroless or `scratch` image lacks them (not tested). |
| **a login shell** | `machine run` execs it with `-c` | Debian: `/bin/bash`. |

### Do not bake in

| Leave out | Why | Fix |
|---|---|---|
| **A machine-id** | `dbus` writes `/var/lib/dbus/machine-id` at build time and systemd copies it into an empty `/etc/machine-id` at boot, so **every machine shares one ID** — verified, section 11 | Last layer: `RUN : > /etc/machine-id && rm -f /var/lib/dbus/machine-id` |
| **The host user** (its lines in passwd/shadow/group, `/etc/sudoers.d/<user>`, `/home/<user>`) | First-boot setup only runs when `id <user>` fails, so a baked-in user means no fresh setup | Only an issue for images snapshotted from a machine; `mkdebian-machine publish` removes them |
| **`/etc/.machine.initialized`** | A per-machine mount | Exclude it from snapshots |
| **SSH host keys** | Would be shared by every machine and published | `rm -f /etc/ssh/ssh_host_*` |

### Recommended

| Add | Why |
|---|---|
| `RUN systemctl mask systemd-modules-load.service` | Apple's kernel ships no modules on disk, so the unit always fails and systemd reports `degraded` (section 7) |
| `procps less iproute2 iputils-ping curl ca-certificates` | Debian images lack `ps`, `free`, `ip`, `less` (section 8) |
| **`sudo`** | Just the package: first boot already grants your host user passwordless sudo. Verified: `container machine run -n <n> -- sudo -n id -u` → `0` |
| `CMD ["/sbin/init"]` | Unused by machines, but marks the image as a machine image — as a plain container it would start systemd as PID 1 |
| `/etc/machine/create-user.sh` (optional) | Replaces Apple's first-boot user setup completely — it must also create the user and sudoers entry (from the script; not tested) |

Minimal Debian 13 version — what `mkdebian-machine new -p sudo` generates:

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

### Checking an image

```sh
container run --rm <img> sh -c 'ls -l /sbin/init; wc -c /etc/machine-id; ls /var/lib/dbus/machine-id'
#   /sbin/init present, machine-id 0 bytes, no dbus copy
container machine create <img> --name t1
container machine create <img> --name t2
sleep 10                                                   # create returns before boot
container machine run -n t1 --root -- 'systemctl is-system-running; cat /etc/machine-id'
container machine run -n t2 --root -- cat /etc/machine-id  # must differ from t1
container machine run -n t1 -- 'id; sudo -n true && echo sudo-ok'
```

Or run `/test-mkdebian-machine` in Claude Code for the full round trip.
