# AlmaLinux Raspberry Pi — Headless Cloud-Init Bootstrap

[![License: Blue Oak 1.0.0](https://img.shields.io/badge/license-Blue%20Oak%201.0.0-2D6DB4)](https://blueoakcouncil.org/license/1.0.0)
[![AlmaLinux 9](https://img.shields.io/badge/AlmaLinux-9-354C7A)](https://almalinux.org)
[![Raspberry Pi 5](https://img.shields.io/badge/Raspberry%20Pi-5-C51A4A)](https://www.raspberrypi.com/products/raspberry-pi-5/)

Cloud-init configuration templates and documentation for running AlmaLinux headless on **Raspberry Pi 5** (and Pi 4).

The official [AlmaLinux Raspberry Pi wiki](https://wiki.almalinux.org/documentation/raspberry-pi.html) covers the basics, but leaves out critical details that cause silent boot failures on headless setups. This repo documents the known pitfalls and provides ready-to-use templates.

> **Status:** Active investigation. These templates represent the best-known configuration based on the official AlmaLinux wiki and community findings. If you discover additional pitfalls or fixes, please open an issue or PR.

---

## Known Pitfalls

Cloud-init on AlmaLinux behaves differently than on Ubuntu or Raspberry Pi OS. The following mistakes will silently kill your headless setup with no error output.

### 1. Do Not Create Custom Users — Modify `almalinux`

The AlmaLinux RPi image ships with a hardcoded default user called `almalinux` (password: `almalinux`). The cloud-init initialization depends on this user existing in the `user-data` template.

Creating a different user in `user-data` can break the init sequence. Modify the existing `almalinux` user instead, then create additional users via `runcmd` or after SSH access is established.

<details>
<summary><b>❌ Incorrect</b> — Creating a custom user</summary>

```yaml
users:
  - name: myuser
    groups: wheel
    sudo: ALL=(ALL) NOPASSWD:ALL
```
</details>

<details>
<summary><b>✅ Correct</b> — Modifying the default user</summary>

```yaml
users:
  - name: almalinux
    groups: [ adm, systemd-journal ]
    sudo: [ "ALL=(ALL) NOPASSWD:ALL" ]
```
</details>

### 2. Use `gateway4`, Not `routes`

AlmaLinux 9 bundles a version of cloud-init that does not reliably translate the newer `routes` syntax into NetworkManager connection profiles. The translation fails silently, leaving the interface with no default route.

<details>
<summary><b>❌ Incorrect</b> — Newer routes syntax</summary>

```yaml
ethernets:
  eth0:
    routes:
      - to: default
        via: 192.168.1.1
```
</details>

<details>
<summary><b>✅ Correct</b> — Legacy gateway4 syntax</summary>

```yaml
ethernets:
  eth0:
    gateway4: 192.168.1.1
```
</details>

### 3. Always Set `optional: true`

Without this flag, cloud-init blocks indefinitely waiting for the interface during the network stage. On a headless Pi with no display output, the system appears completely dead.

```yaml
ethernets:
  eth0:
    optional: true
```

### 4. Do Not Use RPi Imager OS Customization

The Raspberry Pi Imager's built-in "OS Customization" (username, Wi-Fi, locale settings) is **not supported** by AlmaLinux. Enabling it [conflicts with AlmaLinux's cloud-init initialization](https://wiki.almalinux.org/documentation/raspberry-pi.html#burn-raspberry-pi-image) and can prevent the system from creating the default user entirely.

Flash the image without customization. Use the `user-data` file for all configuration.

---

## Quick Start

### Prerequisites

| Requirement | Notes |
|---|---|
| Raspberry Pi 5 or 4 | Tested on Pi 5 (8GB) |
| SD card | 16 GB minimum |
| Ethernet | Wi-Fi config via cloud-init is not supported on AlmaLinux |
| AlmaLinux RPi image | [Download](https://wiki.almalinux.org/documentation/raspberry-pi.html#download-image) the latest `.raw.xz` |

### 1. Flash the Image

```bash
# macOS
diskutil unmountDisk /dev/diskN
xzcat AlmaLinux-9-RaspberryPi-latest.aarch64.raw.xz | sudo dd of=/dev/rdiskN bs=1m status=progress

# Linux
sudo umount /dev/sdX*
xzcat AlmaLinux-9-RaspberryPi-latest.aarch64.raw.xz | sudo dd of=/dev/sdX bs=1M status=progress
```

### 2. Mount `CIDATA`

After flashing, a FAT partition labeled **`CIDATA`** becomes available. This is the boot partition containing `config.txt` and the default `user-data`.

```bash
# macOS
diskutil mountDisk /dev/diskN
ls /Volumes/CIDATA/

# Linux
sudo mount /dev/sdX1 /mnt
ls /mnt/
```

### 3. Edit and Copy `user-data`

1. Open [`user-data`](user-data) from this repo
2. Replace the `passwd` hash — generate yours with:
   ```bash
   # Requires mkpasswd (part of whois package on most distros)
   mkpasswd -m sha-512
   ```
3. Replace the SSH public key with your own
4. Copy to the SD card:
   ```bash
   cp user-data /Volumes/CIDATA/user-data   # macOS
   cp user-data /mnt/user-data              # Linux
   ```

### 4. Create `network-config`

This file **does not exist by default**. You must create it.

1. Open [`network-config`](network-config) from this repo
2. Set your static IP, gateway, and DNS servers
3. Copy to the SD card alongside `user-data`:
   ```bash
   cp network-config /Volumes/CIDATA/network-config   # macOS
   cp network-config /mnt/network-config              # Linux
   ```

### 5. Eject, Insert, Boot

```bash
# macOS
diskutil eject /Volumes/CIDATA

# Linux
sudo umount /mnt
```

Insert the SD card, connect ethernet, and power on. First boot takes **~90 seconds** due to cloud-init initialization.

### 6. Connect

```bash
ssh almalinux@<YOUR_STATIC_IP>
```

---

## After First Boot

```bash
# Update the system
sudo dnf update -y

# Create your preferred user
sudo useradd -m -G wheel,adm,systemd-journal -s /bin/bash myuser
sudo passwd myuser

# Install common tools
sudo dnf install -y podman vim tmux htop

# Set hostname (if cloud-init didn't apply it)
sudo hostnamectl set-hostname myhostname
```

---

## File Reference

```
.
├── README.md          # This guide
├── LICENSE            # Blue Oak Model License 1.0.0
├── user-data          # Cloud-init user configuration template
├── network-config     # Cloud-init static IP network template
└── TROUBLESHOOTING.md # Diagnostic steps when things go wrong
```

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for diagnostic steps when the Pi doesn't respond after boot.

---

## References

- [AlmaLinux Raspberry Pi Wiki](https://wiki.almalinux.org/documentation/raspberry-pi.html)
- [Cloud-init Network Config v2](https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html)
- [Cloud-init NoCloud Datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
- [Cloud-init User Data Examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html)

## Contributing

Issues and pull requests are welcome. If you've gotten AlmaLinux running headless on a Pi with a configuration not covered here, please share your findings.

## License

[Blue Oak Model License 1.0.0](https://blueoakcouncil.org/license/1.0.0) — see [LICENSE](LICENSE).
