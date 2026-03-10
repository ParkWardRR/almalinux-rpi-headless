# Troubleshooting

Diagnostic steps when AlmaLinux doesn't respond after boot on a Raspberry Pi.

## Step 1: Verify Physical Layer

Before debugging cloud-init, confirm the Pi is actually powered and connected.

| Check | What to look for |
|---|---|
| **Power LED** | Solid red light on the Pi board |
| **Activity LED** | Green LED should flash during boot (SD card reads) |
| **Ethernet LEDs** | Orange (link) and green (activity) on the ethernet port |

If the **green activity LED never flashes**, the Pi is not reading from the SD card. Re-flash the image.

If **ethernet LEDs are off**, check the cable and switch port.

## Step 2: Check ARP Cache

Even if you can't ping the Pi, it may have broadcast its MAC address on the network during boot.

```bash
# Flush and re-poll
sudo arp -d -a 2>/dev/null
ping -c 1 172.16.1.69  # or your static IP
arp -a | grep -v incomplete
```

If you see the Pi's MAC address (`d8:3a:dd:*` or `2c:cf:67:*` for Pi 5), the NIC is up but cloud-init may have configured the IP incorrectly.

> **Note:** Pi 5's ethernet interface is `end0`, not `eth0`. Check `network-config` uses the correct name for your board.

## Step 3: Scan the Subnet

The Pi may have ignored the static config and fallen back to DHCP.

```bash
# Scan for SSH on your local subnet (adjust range)
nmap -p 22 --open 172.16.0.0/16 -T4

# Or use arp-scan if available
sudo arp-scan --localnet
```

Look for new hosts that appeared after you powered on the Pi.

## Step 4: Check If DHCP Was Assigned

If you have access to your router's admin panel or DHCP server logs, look for a new lease with:
- Hostname: `almalinux.local` (default) or `rubrica` (if cloud-init applied)
- MAC prefix: `d8:3a:dd` or `2c:cf:67` (Raspberry Pi 5)

## Step 5: Verify SD Card Contents

Re-mount the SD card on your computer and verify:

```bash
# Check user-data starts with #cloud-config
head -1 /Volumes/CIDATA/user-data
# Expected: #cloud-config

# Check network-config exists and has content
cat /Volumes/CIDATA/network-config

# Validate YAML syntax (requires python3)
python3 -c "import yaml; yaml.safe_load(open('/Volumes/CIDATA/user-data'))"
python3 -c "import yaml; yaml.safe_load(open('/Volumes/CIDATA/network-config'))"
```

Common issues:
- Missing `#cloud-config` header in `user-data` (must be the **very first line**)
- Tabs instead of spaces in YAML (YAML requires spaces)
- Invisible BOM characters from Windows editors

## Step 6: Check Cloud-Init Logs (Requires Display or SSH)

If you can connect a display (micro-HDMI for Pi 5, full HDMI for Pi 4) or have SSH access:

1. Login as `almalinux` / `almalinux` (default credentials)
2. Check cloud-init status:
   ```bash
   cloud-init status --long
   ```
3. Read the logs:
   ```bash
   sudo cat /var/log/cloud-init.log | tail -100
   sudo cat /var/log/cloud-init-output.log
   ```
4. Check NetworkManager:
   ```bash
   nmcli device status
   nmcli connection show
   ip addr show end0    # "end0" on Pi 5, "eth0" on Pi 4
   ```
5. Verify cloud-init created the NM connection with the correct interface:
   ```bash
   sudo cat /etc/NetworkManager/system-connections/cloud-init-*.nmconnection
   # Look for interface-name= — it must match the real device (end0 or eth0)
   ```

## Step 7: Nuclear Option — DHCP Fallback

If static IP configuration continues to fail, try booting with DHCP first to confirm the hardware works:

Create a minimal `network-config` (use `end0` for Pi 5, `eth0` for Pi 4):
```yaml
version: 2
ethernets:
  end0:
    dhcp4: true
    optional: true
```

If DHCP works, the issue is specifically with static IP translation in cloud-init → NetworkManager. Configure the static IP manually after SSH access:

```bash
# Find the connection name first
nmcli connection show
# Then modify it (name varies — could be "Wired connection 1" or "cloud-init end0")
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8"
sudo nmcli connection up "Wired connection 1"
```

## Known Issues

| Issue | Status | Workaround |
|---|---|---|
| Pi 5 ethernet is `end0`, not `eth0` | **Confirmed** | Use `end0` in `network-config` for Pi 5 |
| Static IP via `network-config` not applied | Under investigation | Use DHCP, then configure via `nmcli` |
| `routes` syntax silently ignored | Confirmed | Use `gateway4` instead |
| Boot hangs without `optional: true` | Confirmed | Always include in `network-config` |
| RPi Imager customization breaks init | Confirmed by AlmaLinux | Flash without customization |
| GPT backup header wrong after partition resize | Confirmed | Run `sudo sgdisk -e /dev/mmcblk0` after first boot |
| System clock off by months on first boot | Expected (no RTC battery) | `chronyd` corrects via NTP; wait ~30s for time sync |
| `bcm2708_fb` probe fails on headless boot | Harmless | Ignore — framebuffer driver not needed without display |
| Schema validation skipped (missing jsonschema) | Confirmed | `sudo dnf install python3-jsonschema` if you need validation |
