---
name: cosim-gpu-debug
description: Debug QEMU+gem5 MI300X co-simulation failures. Use when guest boot, amdgpu probe, gem5 container startup, cosim socket, MMIO traffic, ROM loading, PSP/SMU masking, ROCm visibility, or GPU test execution fails in cosim-gpu.
---

# cosim-gpu Debug

Use this skill when diagnosing a QEMU+gem5 MI300X co-simulation issue from the
cosim-gpu repository. Keep the workflow evidence-oriented: identify which side
failed, capture the smallest useful logs, then test the specific recovery.

## Baseline

- Work from the cosim-gpu repository root unless a command explicitly names a
  submodule.
- Treat QEMU, the guest serial console, the gem5 container, and shared memory as
  separate failure domains.
- Prefer targeted logs over broad dumps. Save material command output under
  `build/agent/<task-slug>/` when the task is non-trivial.
- Do not restart or delete a running cosim session until the current state has
  been captured or the user explicitly asked for cleanup.

## Environment Check

Check both simulator sides before changing state:

```bash
# gem5 container status
docker ps -a --filter name=gem5-cosim --format '{{.Names}} {{.Status}}'

# QEMU/screen sessions
screen -ls 2>/dev/null

# Shared resources
ls -la /tmp/gem5-mi300x.sock /dev/shm/mi300x-vram /dev/shm/cosim-guest-ram 2>/dev/null
```

Expected resources for the standard launch:

- gem5 container: `gem5-cosim`
- cosim socket: `/tmp/gem5-mi300x.sock`
- guest RAM: `/dev/shm/cosim-guest-ram`
- VRAM: `/dev/shm/mi300x-vram`

## Two-Sided Log Collection

### gem5 side

```bash
# Recent simulator logs
docker logs gem5-cosim 2>&1 | tail -80

# Common high-signal messages
docker logs gem5-cosim 2>&1 | grep -E "warn|error|fatal|panic|GART|cosim|SDMA|MI300X"
```

### QEMU / guest side

If the launch used the default `cosim_launch.sh` screen logging, read
`/tmp/cosim-launch.log`:

```bash
tail -80 /tmp/cosim-launch.log
```

If the session was started as `qemu-cosim`, snapshot the screen without typing
into the guest:

```bash
screen -S qemu-cosim -p 0 -X hardcopy /tmp/qemu-snap.txt
grep -v '^$' /tmp/qemu-snap.txt | tail -40
```

## Guest-Side Inspection

Send commands through the active screen session only after confirming its name.
For `qemu-cosim`:

```bash
screen -S qemu-cosim -p 0 -X stuff 'dmesg | grep -i amdgpu | tail -40\n'
screen -S qemu-cosim -p 0 -X stuff 'lspci; lspci -vvs 00:03.0\n'
screen -S qemu-cosim -p 0 -X stuff 'lsmod | grep amdgpu; rocm-smi; rocminfo 2>/dev/null | head -80\n'
screen -S qemu-cosim -p 0 -X stuff 'systemctl status cosim-gpu-setup.service --no-pager\n'
```

For the default `cosim-launch` session, replace `qemu-cosim -p 0` with
`cosim-launch`.

## Common Failure Patterns

### NULL deref in `amdgpu_atom_parse_data_header`

Symptom: kernel oops at `amdgpu_atom_parse_data_header+0x1b`, often with
`RAX=0`.

Cause: the ROM was not copied to guest physical `0xC0000` before `modprobe`. In
cosim mode the driver's BIOS discovery chain fails unless the ROM is available
through guest RAM for gem5's SMU ROM path.

Recovery:

```bash
dd if=/root/roms/mi300.rom of=/dev/mem bs=1k seek=768 count=128
modprobe amdgpu ip_block_mask=0x67 ppfeaturemask=0 dpm=0 audio=0 ras_enable=0 discovery=2
```

### PSP firmware load failure

Symptom: `PSP load tmr failed!` followed by panic or reset failure.

Cause: PSP init was still enabled. `0x6f` disables SMU but not PSP for this
setup.

Recovery: use `ip_block_mask=0x67`, which disables both PSP and SMU.

### gem5 container exits immediately

Check `docker logs gem5-cosim 2>&1 | tail -80`. Usual causes are Python config
syntax errors, missing shared-memory files, missing socket permissions, or host
OOM during startup.

### QEMU loses connection to gem5

gem5 crashed or closed `/tmp/gem5-mi300x.sock`. Inspect gem5 logs for `fatal`,
`panic`, assertion failures, and the last MMIO or SDMA message before shutdown.

### Driver loads but ROCm reports uninitialized GPU

Check guest `dmesg`, `rocm-smi`, and `systemctl status cosim-gpu-setup`. KIQ
disable timeout `-110` is expected in this cosim path; missing `amdgpu` in
`lsmod` is not.

### Setup service exits successfully but module is absent

Symptom: `cosim-gpu-setup.service` reports success, but `lsmod | grep amdgpu`
is empty.

Cause: a runtime modprobe blacklist from the kernel command line caused
`modprobe` to return success without loading the driver.

Recovery inside the guest setup path:

```bash
rm -f /run/modprobe.d/*blacklist* 2>/dev/null
modprobe amdgpu ip_block_mask=0x67 ppfeaturemask=0 dpm=0 audio=0 ras_enable=0 discovery=2
```

## Debug Flags

Useful launch-time flags:

```bash
./scripts/cosim_launch.sh --gem5-debug MI300XCosim
./scripts/cosim_launch.sh --gem5-debug AMDGPUDevice,PM4PacketProcessor
./scripts/cosim_launch.sh --gem5-debug SDMAEngine
./scripts/cosim_launch.sh --qemu-trace 'mi300x_gem5_*'
```

Record the exact launch command, relevant log paths, and the observed pass or
failure marker before declaring the issue fixed.
