---
name: cosim-gpu-disk-image
description: Edit the cosim-gpu guest disk image with guestmount. Use when adding guest files, systemd services, module configuration, ROM/setup scripts, or other image contents without rebuilding the full packer image.
---

# cosim-gpu Disk Image Editing

Use this skill for direct, auditable edits to the cosim-gpu guest disk image.
The expected image is a raw GPT disk with partition 1 as the root filesystem.

## Safety Rules

- QEMU must be stopped before mounting the image read-write.
- Back up or snapshot the image before risky edits. The image is large, so record
  the chosen backup strategy instead of copying blindly.
- Always unmount with `guestunmount` and wait for writes to flush before the next
  boot.
- Record every file added or changed inside the image when the edit affects boot,
  driver loading, or test behavior.

## Prerequisites

```bash
# QEMU must not be using the disk
screen -ls 2>/dev/null

# guestmount comes from libguestfs-tools
which guestmount
```

Default disk path:

```text
gem5-resources/src/x86-ubuntu-gpu-ml/disk-image/x86-ubuntu-rocm70
```

## Mount Workflow

```bash
DISK="./gem5-resources/src/x86-ubuntu-gpu-ml/disk-image/x86-ubuntu-rocm70"
MOUNTPOINT="/tmp/cosim-disk"
mkdir -p "$MOUNTPOINT"
guestmount -a "$DISK" -m /dev/sda1 --rw "$MOUNTPOINT"
ls "$MOUNTPOINT"/etc/os-release
```

If mounting fails, verify the partition layout:

```bash
fdisk -l "$DISK"
```

For this image, partition 1 is normally the root filesystem. If using a loop
mount manually, the offset is the partition start sector multiplied by 512.

## Common Edits

### Add a systemd service

```bash
cat > "$MOUNTPOINT/etc/systemd/system/my-service.service" <<'EOF'
[Unit]
Description=My Service
After=local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/my-script.sh

[Install]
WantedBy=multi-user.target
EOF

cat > "$MOUNTPOINT/usr/local/bin/my-script.sh" <<'EOF'
#!/bin/bash
echo "Hello from my-service"
EOF
chmod +x "$MOUNTPOINT/usr/local/bin/my-script.sh"

ln -sf /etc/systemd/system/my-service.service \
    "$MOUNTPOINT/etc/systemd/system/multi-user.target.wants/my-service.service"
```

### Add files to the guest

```bash
cp /path/to/local/file "$MOUNTPOINT/root/file"
```

### Configure amdgpu module parameters

```bash
cat > "$MOUNTPOINT/etc/modprobe.d/amdgpu-cosim.conf" <<'EOF'
options amdgpu ip_block_mask=0x67 ppfeaturemask=0 dpm=0 audio=0 ras_enable=0 discovery=2
EOF
```

If the kernel command line blacklists `amdgpu`, the setup service must remove
runtime blacklist files before calling `modprobe`:

```bash
rm -f /run/modprobe.d/*blacklist* 2>/dev/null
modprobe amdgpu ip_block_mask=0x67 ppfeaturemask=0 dpm=0 audio=0 ras_enable=0 discovery=2
```

### Inspect existing services

```bash
ls "$MOUNTPOINT/etc/systemd/system/"
ls "$MOUNTPOINT/etc/systemd/system/multi-user.target.wants/"
```

## Unmount

```bash
guestunmount "$MOUNTPOINT"
sleep 2
mount | grep "$MOUNTPOINT" || echo "unmounted"
```

Do not start QEMU again until the mount has disappeared and the edited files are
confirmed on the next boot.
