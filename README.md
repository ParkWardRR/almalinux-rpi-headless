# AlmaLinux Raspberry Pi — Headless Cloud-Init Bootstrap

[![License: Blue Oak 1.0.0](https://img.shields.io/badge/license-Blue%20Oak%201.0.0-2D6DB4)](https://blueoakcouncil.org/license/1.0.0)
[![AlmaLinux 10](https://img.shields.io/badge/AlmaLinux-10.1-354C7A)](https://almalinux.org)
[![Raspberry Pi 5](https://img.shields.io/badge/Raspberry%20Pi-5-C51A4A)](https://www.raspberrypi.com/products/raspberry-pi-5/)
[![Status: Verified Working](https://img.shields.io/badge/status-verified%20working-brightgreen)](https://github.com/ParkWardRR/almalinux-rpi-headless)

Cloud-init configuration templates for running **AlmaLinux headless on Raspberry Pi 5** (and Pi 4).

The official [AlmaLinux Raspberry Pi wiki](https://wiki.almalinux.org/documentation/raspberry-pi.html) covers the basics but leaves out critical details that cause **silent boot failures** on headless setups. This repo documents the pitfalls and provides verified, copy-paste-ready templates.

> **Status: ✅ Verified working** on AlmaLinux 10.1 (Heliotrope Lion) / Raspberry Pi 5 (8GB) — March 2026.

---

## Known Pitfalls

### 1. The `meta-data` File Ships Empty — This Is the #1 Killer

**This is the most critical issue.** The stock AlmaLinux RPi image includes an **empty** `meta-data` file. Cloud-init's NoCloud datasource requires a valid `instance-id` in `meta-data` to trigger processing. Without it, **cloud-init silently skips your `user-data` and `network-config` entirely** — the system boots with defaults and no SSH access.

```yaml
# meta-data — YOU MUST CREATE THIS
instance-id: my-rpi-v1
local-hostname: my-rpi
dsmode: local
```

### 2. Don't Create Custom Users — Modify `almalinux`

The image ships with a hardcoded default user `almalinux` (password: `almalinux`). Creating a different user in `user-data` can break the init sequence.

<details>
<summary><b>❌ Wrong</b> — Custom user</summary>

```yaml
users:
  - name: myuser          # Breaks AlmaLinux init
    groups: wheel
```
</details>

<details>
<summary><b>✅ Right</b> — Modify default user</summary>

```yaml
users:
  - name: almalinux       # Modify the existing user
    groups: [ adm, systemd-journal ]
```
</details>

### 3. Use DHCP — Static IP via `network-config` Is Unreliable

AlmaLinux's cloud-init has known issues translating `network-config` into NetworkManager profiles. Both v1 and v2 formats can silently fail, leaving the interface dead.

**The reliable approach:** Boot with DHCP, then configure static IP manually via `nmcli` after SSH access.

### 4. Use `gateway4`, Not `routes` (If You Must Use Static)

If you insist on static IP via `network-config`, use the legacy `gateway4` syntax and always include `optional: true`:

```yaml
ethernets:
  eth0:
    gateway4: 192.168.1.1   # NOT routes: [{to: default, via: ...}]
    optional: true           # Prevents boot hang
```

### 5. Don't Use RPi Imager OS Customization

The Raspberry Pi Imager's built-in "OS Customization" [conflicts with AlmaLinux's cloud-init](https://wiki.almalinux.org/documentation/raspberry-pi.html#burn-raspberry-pi-image). Flash the image without customization.

---

## Quick Start

### Prerequisites

| Requirement | Notes |
|---|---|
| Raspberry Pi 5 or 4 | Tested on Pi 5 (8GB) |
| SD card | 16 GB minimum |
| Ethernet cable | Wi-Fi config via cloud-init is not supported |
| AlmaLinux RPi image | [Download](https://wiki.almalinux.org/documentation/raspberry-pi.html#download-image) |

### 1. Flash the Image

```bash
# macOS
diskutil unmountDisk /dev/diskN
xzcat AlmaLinux-*-RaspberryPi-*.raw.xz | sudo dd of=/dev/rdiskN bs=1m status=progress

# Linux
sudo umount /dev/sdX*
xzcat AlmaLinux-*-RaspberryPi-*.raw.xz | sudo dd of=/dev/sdX bs=1M status=progress
```

> ⚠️ **Do NOT use RPi Imager's "OS Customization" feature.**

### 2. Mount `CIDATA` and Copy All Three Files

```bash
# macOS
diskutil mountDisk /dev/diskN

# Copy all three files (edit user-data first — see below)
cp user-data /Volumes/CIDATA/user-data
cp network-config /Volumes/CIDATA/network-config
cp meta-data /Volumes/CIDATA/meta-data    # ← CRITICAL — replaces the empty default

diskutil eject /Volumes/CIDATA
```

**Before copying `user-data`**, edit it to:
1. Replace the `passwd` hash with your own (`mkpasswd -m sha-512`)
2. Replace the SSH public key with yours

### 3. Boot and Connect

Insert SD card → connect ethernet → power on → wait **~90 seconds**.

```bash
# Find the Pi's DHCP-assigned IP from your router, then:
ssh almalinux@<DHCP_IP>
```

---

## File Reference

```
.
├── README.md          # This guide
├── LICENSE            # Blue Oak Model License 1.0.0
├── user-data          # Cloud-init user config (SSH, password, hostname)
├── network-config     # DHCP network config (recommended)
├── meta-data          # Instance ID (CRITICAL — don't skip this)
└── TROUBLESHOOTING.md # Diagnostic steps when things go wrong
```

---

## After First Boot

```bash
# Update the system
sudo dnf update -y

# Create additional users if needed
sudo useradd -m -G wheel,adm,systemd-journal -s /bin/bash myuser
sudo passwd myuser

# Set static IP via nmcli (if needed)
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8"
sudo nmcli connection up "Wired connection 1"

# Install tools
sudo dnf install -y podman vim tmux htop
```

---

## Verified Configuration

This exact configuration was tested and confirmed working:

| Component | Version |
|---|---|
| AlmaLinux | 10.1 (Heliotrope Lion) |
| Raspberry Pi | 5 (8GB) |
| Setup | Fully headless (no HDMI, no keyboard) |
| Network | DHCP via ethernet |
| Date verified | March 2026 |

## References

- [AlmaLinux Raspberry Pi Wiki](https://wiki.almalinux.org/documentation/raspberry-pi.html)
- [Cloud-init NoCloud Datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
- [Cloud-init Network Config v2](https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html)

## Contributing

Issues and pull requests welcome. If you've found additional pitfalls or workarounds, please share.

## License

[Blue Oak Model License 1.0.0](https://blueoakcouncil.org/license/1.0.0) — see [LICENSE](LICENSE).
