# AlmaLinux Raspberry Pi 5 — Headless Bootstrap Guide

> **TL;DR:** AlmaLinux's Raspberry Pi image has specific cloud-init quirks that silently kill headless setups. This repo contains working, copy-paste-ready config files and documents the three pitfalls that aren't in the official docs.

## The Problem

You flash AlmaLinux to an SD card for your Raspberry Pi 5, drop in your `user-data` and `network-config` files, plug it in... and nothing. No ping, no SSH, no sign of life. You re-flash, try again, same result. Hours later you're questioning your sanity.

**This repo exists because the official wiki is missing critical details**, and most cloud-init guides on the internet are written for Ubuntu/Debian where things "just work."

## The Three Pitfalls

### 1. Don't Create a Custom User — Modify `almalinux`

The AlmaLinux RPi image ships with a **hardcoded default user** called `almalinux`. Its cloud-init initialization sequence depends on this user existing. If your `user-data` file creates a different user (like `myuser`), the init process can silently fail and the system won't complete its first boot properly.

**Wrong:**
```yaml
users:
  - name: myuser          # ← This breaks AlmaLinux's init
    groups: wheel
    ...
```

**Right:**
```yaml
users:
  - name: almalinux       # ← Modify the default user instead
    groups: [ adm, systemd-journal ]
    ...
```

You can always create additional users via `runcmd` after the system boots.

### 2. Use `gateway4`, Not `routes`

AlmaLinux 9's bundled `cloud-init` ships with an older version that doesn't reliably translate the newer `routes` syntax into NetworkManager profiles. The translation silently fails, leaving your network interface dead.

**Wrong:**
```yaml
version: 2
ethernets:
  eth0:
    routes:
      - to: default
        via: 192.168.1.1    # ← Newer syntax, silently ignored
```

**Right:**
```yaml
version: 2
ethernets:
  eth0:
    gateway4: 192.168.1.1   # ← Older syntax, actually works
```

### 3. Always Set `optional: true` on `eth0`

Without `optional: true`, cloud-init will hang indefinitely waiting for the interface to come up before proceeding with the rest of initialization. On a headless Pi with no monitor, this is a death sentence — the system appears completely dead.

```yaml
ethernets:
  eth0:
    optional: true           # ← Prevents cloud-init from hanging
```

## Quick Start

### Prerequisites
- Raspberry Pi 5 (also works on Pi 4)
- SD card (16GB+)
- AlmaLinux Raspberry Pi image ([download here](https://wiki.almalinux.org/documentation/raspberry-pi.html))
- Ethernet cable connected to your network

### Step 1: Flash the Image

Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/), [Fedora Media Writer](https://github.com/FedoraQt/MediaWriter), or `dd`:

```bash
# Unmount the disk first
diskutil unmountDisk /dev/diskN

# Flash (macOS — use rdiskN for speed)
xzcat AlmaLinux-9-RaspberryPi-latest.aarch64.raw.xz | sudo dd of=/dev/rdiskN bs=1m
```

> ⚠️ **Do NOT use RPi Imager's "OS Customization" feature.** AlmaLinux doesn't support it and it conflicts with cloud-init. Use these config files instead.

### Step 2: Mount the Boot Partition

After flashing, mount the SD card. A partition named **`CIDATA`** will appear. This is where `config.txt`, `user-data`, and your kernel live.

```bash
diskutil mountDisk /dev/diskN
# The partition mounts at /Volumes/CIDATA
```

### Step 3: Replace `user-data`

Copy `user-data` from this repo into `/Volumes/CIDATA/`, replacing the default one.

**Before you copy**, edit the file to:
1. Replace the example `passwd` hash with your own (generate one with `mkpasswd -m sha-512`)
2. Replace the example SSH key with your own public key

```bash
cp user-data /Volumes/CIDATA/user-data
```

### Step 4: Add `network-config`

This file does **not** exist by default. You must create it.

**Edit `network-config`** to match your network:
- `addresses`: Your desired static IP and subnet
- `gateway4`: Your router's IP
- `nameservers`: Your DNS server(s)

```bash
cp network-config /Volumes/CIDATA/network-config
```

### Step 5: Eject and Boot

```bash
diskutil eject /Volumes/CIDATA
```

Insert the SD card into your Pi, connect ethernet, and power on. Wait **~90 seconds** for the first boot (cloud-init takes extra time on initial run).

### Step 6: SSH In

```bash
ssh almalinux@<YOUR_STATIC_IP>
```

Default password is whatever you set in the `passwd` hash. Your SSH key will also work if you added one.

## File Reference

| File | Purpose |
|------|---------|
| `user-data` | Cloud-init user configuration (hostname, SSH, password) |
| `network-config` | Cloud-init network configuration (static IP, gateway, DNS) |

## After First Boot

Once you're SSH'd in, you can:

```bash
# Create your preferred user
sudo useradd -m -G wheel,adm,systemd-journal -s /bin/bash myuser
sudo passwd myuser

# Set the hostname (if cloud-init didn't)
sudo hostnamectl set-hostname myhostname

# Install packages
sudo dnf update -y
sudo dnf install -y podman vim tmux
```

## Tested On

- AlmaLinux 9 (latest RPi image as of March 2026)
- Raspberry Pi 5 (8GB)
- Headless setup (no HDMI, no keyboard)

## References

- [AlmaLinux Raspberry Pi Wiki](https://wiki.almalinux.org/documentation/raspberry-pi.html)
- [Cloud-init Network Config v2 Docs](https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html)
- [Cloud-init NoCloud Datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)

## License

MIT
