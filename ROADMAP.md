# Roadmap

Future enhancements and ideas for this project. Contributions welcome.

---

## Planned

### Automated First-Boot Script

> **In plain terms:** Right now you do several manual steps after your first SSH login (fix the disk header, update packages, set a static IP). A single script could do all of that for you in one command.

- Single `curl | bash` or `runcmd` script that handles:
  - GPT backup header repair (`sgdisk -e`)
  - System update (`dnf update -y`)
  - Static IP configuration via `nmcli`
  - Timezone and NTP verification
- Could be shipped as a `first-boot.sh` in this repo or embedded in `user-data` runcmd

### Pi 4 Verified Templates

> **In plain terms:** Everything here has been tested on Pi 5. Pi 4 uses a different ethernet name (`eth0` vs `end0`) and may have other quirks. We want a separate set of templates confirmed working on Pi 4.

- Provide a `network-config.pi4` example with `eth0`
- Test and verify the full boot flow on Raspberry Pi 4 (4GB and 8GB)
- Document any Pi 4-specific kernel warnings or differences

### Static IP via `nmcli` Automation

> **In plain terms:** Instead of telling people "set your static IP manually after boot," we could include a cloud-init runcmd that does it automatically during first boot — so the Pi comes up with the right IP from the start.

- Provide a `user-data` variant that configures static IP via `runcmd` + `nmcli` instead of relying on `network-config`
- This bypasses the broken cloud-init → NetworkManager translation entirely
- Template with placeholder IP/gateway/DNS for easy customization

### Wi-Fi Configuration Guide

> **In plain terms:** Wi-Fi doesn't work through cloud-init on this image, but it can be set up after you're logged in. A step-by-step guide would help people who can't use ethernet permanently.

- Post-boot `nmcli` commands for WPA2/WPA3 networks
- `wpa_supplicant` re-enable instructions (our template disables it by default)
- Regulatory domain (`setregdomain`) fix — currently fails silently on boot

---

## Under Consideration

### Ansible/Automated Provisioning Playbook

> **In plain terms:** Ansible is a tool that lets you configure many machines at once from your laptop. An Ansible playbook could take a freshly booted Pi and install everything you need — containers, monitoring, firewalls — without manual SSH commands.

- Ansible playbook for post-boot provisioning (packages, firewall rules, container setup)
- Inventory template for multi-Pi clusters
- Roles for common server patterns (web server, container host, DNS)

### Container Image Compatibility Matrix

> **In plain terms:** The Pi 5 uses ARM chips (aarch64), and not all Docker/Podman container images are built for ARM. Pulling an x86 image gives a confusing "Exec format error." A compatibility list would save people from this trial-and-error.

- Document which popular container images have `linux/arm64` builds
- Provide examples of multi-arch image pulls with `--platform`
- Known-broken images and alternatives (e.g., kea-dhcp4 x86-only images fail with `Exec format error` on aarch64)

### cloud-init Schema Validation in CI

> **In plain terms:** Right now there's no automated check that the YAML template files are valid. A GitHub Action could validate them on every commit so typos and syntax errors get caught before anyone copies them onto an SD card.

- GitHub Actions workflow to validate `user-data`, `network-config`, and `meta-data`
- YAML lint + cloud-init schema check (`cloud-init schema --config-file`)
- Prevents accidental syntax errors from reaching users

### RTC (Real-Time Clock) HAT Guide

> **In plain terms:** The Pi doesn't have a battery-backed clock, so every boot starts with the wrong time until it reaches an NTP server. An RTC HAT (a small add-on board with a coin battery) keeps the clock accurate even without internet. We'd document how to set one up.

- Document DS3231 or PCF8523 RTC HAT setup on AlmaLinux
- `dtoverlay` configuration for `/boot/config.txt`
- Verify `chronyd` behavior with and without RTC

### SELinux Policy Tuning

> **In plain terms:** AlmaLinux has SELinux (a security system) enabled by default. Some container operations and services generate "Permission denied" warnings. Documenting the right policy tweaks would clean up those log warnings without disabling security.

- Document common SELinux denials seen on RPi (BPF, container scopes)
- Provide targeted policy modules rather than `setenforce 0`
- Integration with container workloads (podman + SELinux)

---

## Completed

| Item | Date | Notes |
|---|---|---|
| Fix `eth0` → `end0` for Pi 5 | March 2026 | Interface name mismatch caused silent static IP failures |
| Add `runcmd` to disable bluetooth/wifi | March 2026 | Reduces boot warnings and attack surface |
| Document GPT backup header fix | March 2026 | `sgdisk -e` needed after partition resize |
| Document system clock drift on first boot | March 2026 | Expected behavior — `chronyd` corrects via NTP |
| Verified working on AlmaLinux 10.1 / Pi 5 | March 2026 | Full headless boot with cloud-init NoCloud |
| Initial cloud-init templates | March 2026 | `user-data`, `network-config`, `meta-data` |

---

## How to Contribute

Have an idea or want to tackle one of these items? Open an issue or pull request. For larger features, open an issue first to discuss the approach.
