---
name: cosim-gpu-guest
description: Interact with the cosim-gpu guest Linux over the QEMU screen serial console. Use when sending commands to the guest, mounting the 9p share, checking GPU state, building tests in the guest, running tests, or shutting the guest down cleanly.
---

# cosim-gpu Guest Interaction

Use this skill for controlled interaction with a running cosim-gpu guest. The
normal launch path uses `scripts/cosim_launch.sh`, a `cosim-launch` screen
session, and `/tmp/cosim-launch.log`.

## Prerequisites

Confirm the environment before typing into the guest:

```bash
screen -ls 2>/dev/null
docker ps --filter name=gem5-cosim --format '{{.Names}}: {{.Status}}'
tail -20 /tmp/cosim-launch.log
```

Do not assume the session name. Use `cosim-launch` only after it appears in
`screen -ls`.

## Sending Guest Commands

```bash
# Send a command
screen -S cosim-launch -X stuff '<command>\n'

# Send Ctrl-C
screen -S cosim-launch -X stuff $'\x03'

# Read recent output
tail -80 /tmp/cosim-launch.log
```

For commands that take time, capture the log line count before sending the
command and then wait for new output:

```bash
baseline=$(wc -l < /tmp/cosim-launch.log)
screen -S cosim-launch -X stuff '<command>\n'
while true; do
    current=$(wc -l < /tmp/cosim-launch.log)
    if [ "$current" -gt "$((baseline + 3))" ]; then
        tail -80 /tmp/cosim-launch.log
        break
    fi
    sleep 5
done
```

## Mounting the 9p Share

The host launch option `--share-dir <path>` exposes a virtio-9p mount named
`cosim_share`. Mount it in the guest with:

```bash
screen -S cosim-launch -X stuff 'mkdir -p /mnt && mount -t 9p -o trans=virtio,version=9p2000.L cosim_share /mnt\n'
```

Files from the host share then appear under guest `/mnt`.

## Building and Running GPU Tests

```bash
# Copy shared test sources and build
screen -S cosim-launch -X stuff 'mkdir -p /root/tests && cp -r /mnt/* /root/tests/ && make -C /root/tests all 2>&1 | tail -20\n'

# Run all tests
screen -S cosim-launch -X stuff 'make -C /root/tests test 2>&1\n'

# Run one test binary
screen -S cosim-launch -X stuff '/root/tests/build/<test_name> 2>&1\n'
```

Treat test completion as unproven until the serial log shows the test-specific
pass marker or failure output.

## GPU Status Checks

```bash
screen -S cosim-launch -X stuff 'rocm-smi\n'
screen -S cosim-launch -X stuff 'rocminfo 2>/dev/null | head -80\n'
screen -S cosim-launch -X stuff 'dmesg | grep -i amdgpu | tail -40\n'
screen -S cosim-launch -X stuff 'systemctl is-active cosim-gpu-setup; systemctl status cosim-gpu-setup --no-pager\n'
```

`amdgpu` must be present in `lsmod`; a successful service exit alone is not
enough because a runtime blacklist can make `modprobe` exit zero without loading
the module.

## Launching a Session

```bash
# Basic launch with screen log
screen -dmS cosim-launch -L -Logfile /tmp/cosim-launch.log \
    ./scripts/cosim_launch.sh

# Launch with host share and gem5 debug flags
screen -dmS cosim-launch -L -Logfile /tmp/cosim-launch.log \
    ./scripts/cosim_launch.sh --share-dir /path/to/dir --gem5-debug MI300XCosim
```

Wait for a concrete boot marker before guest operations, for example an automatic
login prompt in `/tmp/cosim-launch.log`.

## Clean Shutdown

Prefer an in-guest shutdown when the guest is responsive:

```bash
screen -S cosim-launch -X stuff 'poweroff\n'
```

If QEMU must be terminated from the console, send the QEMU monitor exit chord:

```bash
screen -S cosim-launch -X stuff $'\x01x'
```

Clean residual host resources only after QEMU and gem5 are stopped:

```bash
docker rm -f gem5-cosim 2>/dev/null
rm -f /tmp/gem5-mi300x.sock /dev/shm/mi300x-vram /dev/shm/cosim-guest-ram 2>/dev/null
```
