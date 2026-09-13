---
name: test-mkdebian-machine
description: End-to-end test of bin/mkdebian-machine — both modes. `new` builds a throwaway image and machine and customizes it; `publish` snapshots that machine and a container into a throwaway local registry, then a machine is created from the pushed image. Cleans up everything. Use after changing the script or when asked to verify it still works.
---

Test `bin/mkdebian-machine` against the real `container` CLI. Use only the throwaway `mkdm-*` names below; never touch `debian13-vm` or `debian13-systemd`. Never push to Docker Hub or any real registry — only to the local registry on port 5050 (5000 is taken by macOS AirPlay).

Pass any extra flags from `$ARGUMENTS` through to the `new` run (e.g. `-d 12 -p "git"`).

`machine run` re-parses its arguments with `bash -c`: pass shell snippets as ONE quoted argument, never `-- sh -c '...'`. No TTY: always pass a command.

0. Baseline: record `container builder status` (running/stopped), `container ls -a`, `container machine ls`, `container image ls`.
1. Static check: `bash -n bin/mkdebian-machine && shellcheck bin/mkdebian-machine`.
2. Pre-clean leftovers (ignore errors): machines `mkdm-new-vm mkdm-pub-vm`, containers `mkdm-registry mkdm-regsnap-test`, images `mkdm-new localhost:5050/mkdm-snap:1 localhost:5050/mkdm-regsnap:1`, and `~/.cache/mkdebian-machine/`.
3. **new** — write a setup script to the scratchpad that installs `jq` and writes `/etc/mkdm-marker`, then:
   `bin/mkdebian-machine new -t mkdm-new -m mkdm-new-vm --cpus 3 --memory 3G --setup <script> $ARGUMENTS`
   Must exit 0 with `ok /sbin/init -> ...` and `ok setup script finished`. Verify:
   - `container machine inspect mkdm-new-vm | jq '.[0] | {status, cpus, memory}'` → running, 3, 3221225472
   - `container machine run -n mkdm-new-vm --root -- 'systemctl is-system-running; cat /etc/mkdm-marker; command -v jq'` → `running`, marker text, jq path
   - `container machine run -n mkdm-new-vm --root -- 'for t in ps less ip ping curl; do command -v "$t" || echo "MISSING $t"; done'`
   - On boot failure, read `container machine logs mkdm-new-vm`.
4. Local registry: `container run -d --name mkdm-registry -p 5050:5000 docker.io/library/registry:2`; `curl -s -o /dev/null -w '%{http_code}' http://localhost:5050/v2/` → 200.
5. **publish a machine**: `bin/mkdebian-machine publish mkdm-new-vm localhost:5050/mkdm-snap:1 --scheme http` → exit 0, pushed. Then round-trip:
   - `container image rm localhost:5050/mkdm-snap:1` (force a real pull), then `container machine create localhost:5050/mkdm-snap:1 --scheme http --name mkdm-pub-vm`
   - Poll `container machine run -n mkdm-pub-vm --root -- true` until it succeeds (create returns before boot).
   - In `mkdm-pub-vm`: `systemctl is-system-running` → running; `/etc/mkdm-marker` and `jq` present (customization carried over); `/etc/machine-id` differs from mkdm-new-vm's (also create a second machine from `mkdm-new` and check its ID differs too — images must not carry `/var/lib/dbus/machine-id`, or systemd copies it into every machine); `ls /etc/ssh/ssh_host_* 2>/dev/null` empty; host user was recreated by first boot (`id <host user>` works, `container machine run -n mkdm-pub-vm -- id -u` = host uid).
6. **publish a container**: `bin/mkdebian-machine publish mkdm-registry localhost:5050/mkdm-regsnap:1 --scheme http` → exit 0. Run it: `container run -d --name mkdm-regsnap-test -p 5051:5000 localhost:5050/mkdm-regsnap:1`, `curl` port 5051 `/v2/` → 200 (command/env carried over).
7. Error paths, each must exit 1 with a clear message: `publish` (no args), `publish no-such-thing localhost:5050/x:1`, `new --setup /etc/hosts` (no -m), `publish a b --scheme ftp`.
8. Clean up, even if a step failed: stop and delete `mkdm-new-vm mkdm-pub-vm`; stop and `container rm` `mkdm-registry mkdm-regsnap-test`; `container image rm` `mkdm-new localhost:5050/mkdm-snap:1 localhost:5050/mkdm-regsnap:1 docker.io/library/registry:2`; `rm -rf ~/.cache/mkdebian-machine`. Restore the builder to its baseline state. Compare `container ls -a` / `machine ls` / `image ls` with the baseline — nothing new may remain.
9. Report pass/fail per step with the relevant output. If anything new was learned, update README.md (and CLAUDE.md if it would change how Claude works here).
