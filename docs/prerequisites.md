# Prerequisites

What a Mac needs before the rest of this repo works. Checked on 2026-09-13 on
the Mac these notes were written on (Apple M3, 8 GB, macOS 27.0). "This Mac" is
what was installed there; "Not verified" marks anything that was not run.

## Hardware and macOS

| Need | Why | This Mac |
|---|---|---|
| **Apple silicon** | `container` requires it (Apple's README). Images and VMs are arm64. | Apple M3, `arm64` |
| **macOS 26 or newer** | Apple's README: "supported on macOS 26", which added the virtualization and networking features it relies on | macOS 27.0 (26A428) |
| Memory | Machines default to half of system memory unless you pass `--memory`. The builder takes 2 CPUs / 2 GB while running, and Ollama's Gemma 4 E2B models take 1.7–3.6 GB. 8 GB works, but only with one model and a 2 GB machine at a time. | 8 GB |
| Disk | A Debian machine image unpacks to about 1.4 GB. Everything `container` stores lives in one folder (below). | that folder is 19 GB here |

## What `container` is, and what it is not

- **Not Docker, and no containerd.** `container` is Apple's own tool, written
  in Swift. Each container and each machine runs in its own lightweight VM.
  Docker Desktop, containerd, `dockerd` or `nerdctl` are not needed and not
  used. No `containerd` process runs.
- **Images are ordinary OCI images,** so it pulls from and pushes to Docker Hub,
  GHCR and so on. A `container machine inspect` shows an
  `io.containerd.image.name` annotation. That is only a standard image label.
- **Its services are launchd agents for your user** (`gui/<uid>`).
  `container system start` registers them:

| launchd label | Program | Role |
|---|---|---|
| `com.apple.container.apiserver` | `/usr/local/bin/container-apiserver` | API server the CLI talks to |
| `com.apple.container.machine-apiserver` | `plugins/machine-apiserver` | machines (`container machine ...`) |
| `com.apple.container.container-core-images` | `plugins/container-core-images` | image store, pulls and pushes |
| `com.apple.container.container-network-vmnet.default` | `plugins/container-network-vmnet` | the VM network: `bridge100`, Mac at `192.168.64.1` |
| `com.apple.container.container-runtime-linux.<name>` | `plugins/container-runtime-linux` | one per running container or machine |

```sh
launchctl list | grep com.apple.container
```

The builder is not a service. It is an ordinary container named `buildkit`
that runs only while `container build` needs it.

**Paths** (from `container system status`):

| What | Where |
|---|---|
| CLI and API server | `/usr/local/bin/container`, `/usr/local/bin/container-apiserver` |
| Plugins | `/usr/local/libexec/container/plugins/` |
| Update and uninstall scripts | `/usr/local/bin/update-container.sh`, `/usr/local/bin/uninstall-container.sh` |
| All data: images, VM disks, service definitions | `~/Library/Application Support/com.apple.container/` |

**After a reboot:** the service definitions live in that data folder, for
example `apiserver/apiserver.plist` with `RunAtLoad`, not in
`~/Library/LaunchAgents`. So launchd has nothing to load at login, and you
should expect to run `container system start` again. Not verified by
rebooting.

## 1. Install `container`

Details and gotchas are in [README section 1](../README.md#1-installing-the-cli).

```sh
# 1. download the signed installer from https://github.com/apple/container/releases
pkgutil --check-signature container-1.4.1-installer-signed.pkg
#    Developer ID Installer: Apple Inc. - Containerization (UPBK2H6LZM)

# 2. install: double-click it, or
open container-1.4.1-installer-signed.pkg       # `sudo installer` needs a real terminal

# 3. start the services and install a Linux kernel (once)
container system start
container system kernel set --recommended      # the prompt from `system start` cannot be answered by scripts

# 4. check
container system status
#    status              running
#    client.version      1.4.1
#    server.version      1.4.1
pkgutil --pkg-info com.apple.container-installer    # version: 1.4.1
```

Defaults on this Mac (`container system property list`):

| Setting | Value |
|---|---|
| builder | 2 CPUs, 2048 MB, image `ghcr.io/apple/container-builder-shim/builder:0.13.1`, Rosetta on |
| a plain `container run` | 4 CPUs, 1 GB |

Update or remove it with Apple's scripts. Neither was run here:

```sh
/usr/local/bin/update-container.sh
/usr/local/bin/uninstall-container.sh -k   # keep images and VM disks
/usr/local/bin/uninstall-container.sh -d   # delete them too
```

## 2. Tools the repo's scripts use

| Tool | Used by | Install | This Mac |
|---|---|---|---|
| `jq` | `bin/mkdebian-machine publish`, `bin/test-mac-ollama`, `bin/test-vm-ollama`, the Claude Code hook | ships with macOS at `/usr/bin/jq` | `jq-1.7.1-apple` |
| `curl` | the Ollama tests, downloads | ships with macOS | 8.7.1 |
| `git` | this repo | `/usr/bin/git` | 2.50.1 |
| Homebrew | everything below | <https://brew.sh> | `/opt/homebrew` |
| `shellcheck` | the Claude Code hook in `.claude/settings.json`, which lints `bin/` on every edit | `brew install shellcheck` | 0.11.0 |
| `gh` | creating the GitHub repo (optional) | `brew install gh` | 2.100.0 |

`bin/mkdebian-machine new` needs only `container`.

## 3. Optional: a local LLM on the Mac's GPU

Only for [README section 15](../README.md#15-using-the-macs-gpu-from-a-vm-local-ollama).
VMs have no GPU, so the model runs on macOS.

| Tool | Install | This Mac |
|---|---|---|
| Ollama | `brew install ollama`, then `brew services start ollama` (creates `~/Library/LaunchAgents/homebrew.mxcl.ollama.plist`) | 0.33.3 |
| Gemma 4 E2B | `ollama pull gemma4:e2b` (1.7 GB loaded) and/or `ollama pull gemma4:e2b-it-qat` (4.3 GB download, 3.6 GB loaded) | both |
| llama.cpp (benchmark only) | `brew install llama.cpp` (Metal build) | 0.4.0 |
| `uv` (MLX benchmark only) | `brew install uv`; `mlx-lm` runs through `uv tool run`, no venv | 0.12.13 |

VMs can only reach Ollama if it listens on all interfaces. Set `OLLAMA_HOST` in
the plist and restart it with `launchctl`, **not** `brew services restart`,
which rewrites the plist ([README section 15](../README.md#mac-side-point-ollama-at-all-interfaces)).

Ports: Homebrew's service does not set `OLLAMA_HOST`, so Ollama's own default
applies (`127.0.0.1:11434`). This Mac had already been set to port **8000**,
which is why the scripts default to `http://127.0.0.1:8000` and
`http://192.168.64.1:8000`. On a fresh install, either set port 8000 too or pass
`--url`. A fresh Ollama install was not tested here; it was already installed.

With a non-default port, the `ollama` CLI needs to be told where the server is:

```sh
OLLAMA_HOST=127.0.0.1:8000 ollama pull gemma4:e2b
```

## 4. Inside the VM

| Tool | Why | Install |
|---|---|---|
| `curl` | `bin/test-vm-ollama` and calling Ollama | `container machine run -n <n> --root -- 'apt-get update && apt-get install -y curl'` (`mkdebian-machine new` and `build/Dockerfile.debian13-systemd` include it) |
| `sudo` (optional) | first boot already grants your user passwordless sudo; only the package is missing | add `-p sudo` to `mkdebian-machine new` |

## Check it all

```sh
sw_vers -productVersion; uname -m                  # 26.0 or newer; arm64
container system status | sed -n 2,3p              # status running, client.version
launchctl list | grep -c com.apple.container       # services registered (4+, plus one per running VM)
for t in jq curl git shellcheck ollama; do printf '%-10s %s\n' "$t" "$(command -v "$t" || echo MISSING)"; done

# with Ollama set up (section 3):
bin/test-mac-ollama && bin/test-vm-ollama
```
