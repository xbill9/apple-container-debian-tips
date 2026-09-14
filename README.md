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
| Machine | `debian13-vm` | from `debian13-systemd`, running, the default, **2 CPUs / 2 GB** (not the 4 / 4 GB of section 3), home mounted rw, `curl` 8.14.1 installed |
| Images | `debian13-systemd:latest`, `debian:13` | |
| Containers | none of mine | the old `debian13` (`sleep infinity`) container is gone |
| Builder | `buildkit` | stopped; `container build` starts it on demand |
| Registry logins | `xbill9` @ `registry-1.docker.io` | from `container registry login docker.io` (section 12) |
| Docker Hub | `xbill9/debian13-machine:latest` | public, linux/arm64, built with `mkdebian-machine new` (section 12) |
| GUI | `~/AppleContainerDesktop` | Tauri app, built locally, unsigned |
| MCP server | `~/AppleContainerMCP` | registered with Claude Code at user scope (section 13) |
| shellcheck | 0.11.0, Homebrew | runs from this repo's Claude Code hook on edits to `bin/` |
| Ollama | 0.33.3, Homebrew service on **`0.0.0.0:8000`** | `gemma4:e2b`, `gemma4:e2b-it-qat`, `gemma4:e4b`; VMs use `http://192.168.64.1:8000` (section 15) |
| llama.cpp | 0.4.0, Homebrew (Metal) | `~/models/gguf/gemma-4-E2B_q4_0-it.gguf` (Google's QAT Q4_0) |
| MLX | `mlx-lm` 0.31.3 via `uv tool run` | `mlx-community/gemma-4-E2B-it-qat-4bit` in `~/.cache/huggingface` |

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

---

## 15. Using the Mac's GPU from a VM: local Ollama

A VM gets no GPU. Linux inside `container` sees only virtual CPUs and virtual
devices, with no Metal access. So the model runs on **macOS**, where Ollama
uses the M3 GPU, and the VM calls it over the VM network. This is set up
open on purpose; it is a demo.

### How the VM reaches the Mac

```
debian13-vm  192.168.64.59        (changes on restart: it was .12 before)
    │  default route + DNS → 192.168.64.1
    ▼
vmenet0 ─ bridge100 on the Mac  192.168.64.1   ← the Mac itself on the VM network
    │
Ollama  *:8000  →  Metal GPU
```

- `192.168.64.1` is the Mac's address on `bridge100`. It is the VM's gateway
  and DNS server, and it did not change across VM restarts.
- There is no `host.docker.internal` / `host.container.internal` name; use the IP.
- No port forwarding: the VM connects straight to the Mac. That only works
  because Ollama listens on all interfaces. On `127.0.0.1` the VM cannot
  reach it.
- No firewall rules are needed: the macOS application firewall is off, and the
  VM has no `nft`, `iptables` or `ufw`. With `0.0.0.0` and no firewall, Ollama
  is also reachable from the Wi-Fi network (checked at `<mac-wifi-ip>:8000`).

### Mac side: point Ollama at all interfaces

Ollama runs as a Homebrew launchd service. This Mac's plist had been customized
to `OLLAMA_HOST=127.0.0.1:8000`, `OLLAMA_KV_CACHE_TYPE=q4_0` and
`OLLAMA_FLASH_ATTENTION=1`. Changed only the host:

```sh
P=~/Library/LaunchAgents/homebrew.mxcl.ollama.plist
cp -p "$P" "$P.bak-$(date +%Y%m%d%H%M%S)"
/usr/libexec/PlistBuddy -c 'Set :EnvironmentVariables:OLLAMA_HOST 0.0.0.0:8000' "$P"

# restart so launchd rereads the plist
launchctl bootout gui/$(id -u)/homebrew.mxcl.ollama
launchctl bootstrap gui/$(id -u) "$P"

lsof -nP -iTCP:8000 -sTCP:LISTEN     # ollama  *:8000
```

**Do not use `brew services restart ollama`.** It regenerates the plist from
the formula, which sets only `OLLAMA_FLASH_ATTENTION=1` and
`OLLAMA_KV_CACHE_TYPE=q8_0`. That silently drops `OLLAMA_HOST` (back to
localhost only) and the `q4_0` cache setting.

### VM side: curl

Debian images do not ship `curl`:

```sh
container machine run -n debian13-vm --root -- 'apt-get update && apt-get install -y --no-install-recommends curl ca-certificates'
```

(`build/Dockerfile.debian13-systemd` now includes `curl`, and
`mkdebian-machine new` always has.)

From the VM, verified 2026-09-13:

```sh
# models
container machine run -n debian13-vm -- 'curl -s http://192.168.64.1:8000/v1/models'
# {"data":[{"id":"gemma4:e2b-it-qat"},{"id":"gemma4:e2b"},{"id":"gemma4:e4b"}], ...}

# Ollama native API
container machine run -n debian13-vm -- 'curl -s http://192.168.64.1:8000/api/generate -d "{\"model\":\"gemma4:e2b\",\"prompt\":\"Say hello from a Debian VM in five words.\",\"stream\":false,\"think\":false}"'
# "response":"Hello from Debian VM."

# OpenAI-compatible API (base URL http://192.168.64.1:8000/v1)
container machine run -n debian13-vm -- 'curl -s http://192.168.64.1:8000/v1/chat/completions -H "Content-Type: application/json" -d "{\"model\":\"gemma4:e2b\",\"messages\":[{\"role\":\"user\",\"content\":\"Name one Debian release codename.\"}],\"reasoning_effort\":\"none\"}"'
# "content":"One Debian release codename is **Bookworm**."
```

`ollama ps` on the Mac showed the model at `100% GPU` while the VM called it.
Generation ran at about 45 tokens/s.

### Test on the Mac first: `bin/test-mac-ollama`

Run this before involving a VM. If it passes and `bin/test-vm-ollama` fails,
the problem is the VM or the network path, not Ollama. It checks:

- what listens on the port. VMs need `*:8000`; `127.0.0.1:8000` fails with the fix.
- Ollama answers on localhost, and on `192.168.64.1`, the address VMs use
  (skipped until a container or machine has created `bridge100`)
- the plist's `OLLAMA_HOST`, to spot a `brew services restart` reset
- the model list
- the native API, streaming and the OpenAI-compatible API
- 100% GPU placement (`/api/ps`)
- a second model

Models are unloaded on exit. It exits 0 only when every check passes, 1 on a
failed check, and 2 on a setup error.

```sh
bin/test-mac-ollama                      # http://127.0.0.1:8000, gemma4:e2b + gemma4:e2b-it-qat
bin/test-mac-ollama -m gemma4:e2b --no-qat
```

Run on 2026-09-13 (20 s):

```
== 1. listener on port 8000
  ollama *:8000
  PASS  ollama listens on all interfaces, so VMs can reach it
== 2. HTTP on http://127.0.0.1:8000
  PASS  Ollama 0.33.3 answers on http://127.0.0.1:8000
== 3. VM-facing address 192.168.64.1
  PASS  Ollama answers on http://192.168.64.1:8000 (bridge100), the address VMs use
== 4. launchd config
  info  OLLAMA_HOST in plist: 0.0.0.0:8000
== 5. models
  PASS  gemma4:e2b is installed
== 6. native API
  23 tokens @ 43.6 tok/s, load 5.7 s
  PASS  gemma4:e2b answered through /api/generate
== 7. streaming
  PASS  streaming delivers incremental chunks
== 8. OpenAI-compatible API
  PASS  /v1/chat/completions returned text
== 9. GPU
  PASS  gemma4:e2b loaded 100% on the GPU (1.6 GB)
== 10. second model: gemma4:e2b-it-qat
  33 tokens @ 43.3 tok/s, load 7.5 s
  PASS  gemma4:e2b-it-qat answered through /api/generate
  PASS  gemma4:e2b-it-qat loaded 100% on the GPU (3.3 GB)

10 passed, 0 failed
Ollama works on the Mac. Next: bin/test-vm-ollama
```

With nothing on the port, it stops at step 1 with
`nothing listens on port 9999; start Ollama: launchctl bootstrap gui/501 ...`
and exits 1.

### Then from the VM: `bin/test-vm-ollama`

Runs the whole round trip from inside a machine and exits 0 only if every check
passes. Run `bin/test-mac-ollama` first. It checks:

- the network path
- the model list
- the native API
- streaming
- the OpenAI-compatible API
- 100% GPU placement (asked from the Mac through `/api/ps`)
- a second model

Models are unloaded on exit. It needs `curl` in the machine and `jq` on the Mac.

```sh
bin/test-vm-ollama                                  # debian13-vm, gemma4:e2b + gemma4:e2b-it-qat
bin/test-vm-ollama -n dev-vm -m gemma4:e2b --no-qat
bin/test-vm-ollama --url http://<other-host>:8000   # e.g. a llama.cpp/Ollama box elsewhere
```

Run on 2026-09-13 (26 s):

```
== 1. network path
  PASS  VM reaches http://192.168.64.1:8000 (connect 0.002372 s, Ollama 0.33.3)
== 2. models
  gemma4:e2b-it-qat gemma4:e2b gemma4:e4b
  PASS  gemma4:e2b is listed
== 3. native API
  reply: Debian is a free and open-source operating system that forms the basis for many other Linux distributions.
  22 tokens @ 45.2 tok/s, load 5.4 s
  PASS  gemma4:e2b answered through /api/generate
== 4. streaming
  14 chunks: 1, 2, 3, 4, 5
  PASS  streaming delivers incremental chunks
== 5. OpenAI-compatible API
  PASS  /v1/chat/completions returned text
== 6. GPU (asked from the Mac)
  PASS  gemma4:e2b loaded 100% on the GPU (1.6 GB)
== 7. second model: gemma4:e2b-it-qat
  35 tokens @ 38.9 tok/s, load 12.4 s
  PASS  gemma4:e2b-it-qat answered through /api/generate
  PASS  gemma4:e2b-it-qat loaded 100% on the GPU (3.3 GB)

8 passed, 0 failed
```

A wrong URL fails at step 1 (`no HTTP 200 ... is Ollama listening on 0.0.0.0?`)
and exits 1. A missing machine, missing `curl` or a bad option exits 2.
The GPU check reads `size_vram / size` from Ollama's `/api/ps`, the same numbers
`ollama ps` prints as `100% GPU`.

### Gotcha: Gemma 4 thinks first, and an empty reply means it ran out of tokens

Gemma 4 is a reasoning model. With a small token limit, the whole budget goes
to hidden reasoning and the answer comes back **empty**:

| Request | Result |
|---|---|
| native, `num_predict: 40` | `"response":""`, `done_reason: length` |
| OpenAI, `max_tokens: 20` | `"content":""`, `finish_reason: length`; `message.reasoning` starts `Thinking Process:` |
| native, `"think": false` | `Hello from Debian VM.` in 6 tokens |
| OpenAI, `"reasoning_effort": "none"` | an answer in 11 tokens |
| OpenAI, no limit | `Bookworm`, but 159 tokens, mostly reasoning |

Turn thinking off (`think: false` / `reasoning_effort: "none"`) for quick
answers, or leave room for the reasoning.

### Models on this Mac (M3, 8 GB)

| Model | Quantization | Loaded | Speed | Notes |
|---|---|---|---|---|
| `gemma4:e2b` | Q4_K_M | 1.7 GB | ~44 tok/s | The comfortable choice for 8 GB |
| `gemma4:e2b-it-qat` | Q4_0, QAT (Google) | 3.6 GB | ~38–44 tok/s | Better quality at 4 bits, but free memory dropped to 6% with `debian13-vm` running |
| `gemma4:e4b` | Q4_K_M | not run | — | 9.6 GB on disk, more than this Mac's RAM |

### Benchmark: Ollama vs llama.cpp vs MLX

Same Gemma 4 E2B QAT weights in each engine, 512-token prompt, 128 generated
tokens, 3 runs, one engine loaded at a time, on the Mac itself (not through a
VM), with `debian13-vm` running:

| Engine | Setup | Prompt tok/s | Generate tok/s | Memory |
|---|---|---|---|---|
| llama.cpp 0.4.0 (`llama-bench`) | `gemma-4-E2B_q4_0-it.gguf`, defaults (KV f16) | 671 | **47.6** | 3.1 GB weights |
| llama.cpp 0.4.0 | same, Ollama's settings (flash attn, KV q4_0) | 652 | 42.8 | |
| Ollama 0.33 (HTTP) | `gemma4:e2b-it-qat`, flash attn, KV q4_0 | 567 | 43.9 | 3.6 GB |
| MLX, `mlx-lm` 0.31.3 (`mlx_lm.benchmark`) | `mlx-community/gemma-4-E2B-it-qat-4bit` | **1,857** | 39.5 | 3.9 GB peak |

- **Generation** is within ~20% across engines. On an 8 GB M3 it is limited by
  memory bandwidth more than by the engine. Ollama and llama.cpp are
  essentially equal with the same settings; the `q4_0` KV cache costs about 10%.
- **Prompt processing** is ~3× faster on MLX, which matters for long prompts.
- Caveats: `llama-bench` and `mlx_lm.benchmark` use synthetic tokens, Ollama got
  real text; the MLX 4-bit build comes from the same QAT source but is not
  byte-identical to the GGUF; the Mac had ~3.9 GB of swap in use for all runs.
- `/usr/bin/time -l` reports almost nothing for GPU memory on Apple silicon
  (0.25 GB for llama.cpp); use each engine's own figure.
- A first Ollama run was invalid: an identical warm-up prompt was served from
  Ollama's prompt cache (16,587 "tok/s"), and a raw prompt stopped after 1
  token. Vary the prompt per run and check `eval_count`.

```sh
# reproduce
llama-bench -m ~/models/gguf/gemma-4-E2B_q4_0-it.gguf -p 512 -n 128 -r 3 -ngl 99
uv tool run --from mlx-lm mlx_lm.benchmark --model mlx-community/gemma-4-E2B-it-qat-4bit -p 512 -g 128 -n 3
```

A CUDA build of llama.cpp cannot run on a Mac: CUDA is NVIDIA-only, and on Apple
silicon llama.cpp uses Metal. A CUDA machine elsewhere on the network can serve
the VM the same way, through `llama-server`'s OpenAI-compatible API (not tested
from here).
