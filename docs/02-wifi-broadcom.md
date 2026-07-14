# 02 - No network after install: Broadcom BCM4360

## Problem

Fresh Ubuntu Server 24.04 install on a 2015 MacBook Air.
`ip a` showed only `lo` (loopback, 127.0.0.1).
No WiFi interface at all. Not down, not misconfigured. Simply absent.

## Diagnosis

```bash
ip link                    # only `lo` → interface not present, not just disabled
lspci -nn | grep -i net    # Broadcom BCM4360 802.11ac [14e4:43a0]
```

Root cause: Ubuntu ships no driver for the Broadcom BCM4360 (proprietary, not
included in the default install). The hardware is there, but the kernel has no
way to talk to it.

## The chicken-and-egg problem

Installing the driver requires `apt`. `apt` requires internet. Internet requires
WiFi. WiFi requires the driver.

**Workaround:** USB tethering from a phone. The phone shows up as a wired
interface (`enx...`), needs no extra drivers, and provides internet just long
enough to install the real one.

## Driver install

```bash
sudo apt update
sudo apt install -y broadcom-sta-dkms
sudo modprobe -r b43 ssb bcma
sudo modprobe wl
ip link                    # `wlp3s0` now exists
```

## Second problem: interface up, still no IPv4

`wlp3s0` showed `UP, LOWER_UP` and the logs said "Gained carrier". But DHCP never
assigned an address. Only `inet6 fe80::` (link-local) was present.

The breakthrough came from:

```bash
networkctl status wlp3s0
```

Output:

```
State: degraded (configuring)
Wi-Fi access point: (null) (00:00:00:00:00:00)
```

`access point: (null)` means the card was **not associated with any network at
all**. "Gained carrier" was misleading. The driver brought the interface up, but
no actual WiFi association ever happened.

## Root cause

netplan does not connect to WiFi itself. It delegates association and the WPA
handshake to `wpa_supplicant`. That package **was not installed** on the minimal
server image. No association means no DHCP, which means no IP.

Confirmed by:

```bash
journalctl -u wpa_supplicant    # "No entries" → the service didn't even exist
```

## Fix

```bash
# phone tethered for internet, otherwise apt fails with "Temporary failure resolving"
sudo apt install -y wpasupplicant
sudo systemctl restart systemd-networkd
sudo netplan apply
```

Result: `Wi-Fi access point:` now shows the SSID instead of `(null)`,
DHCP lease acquired, WiFi working standalone without the phone.

## netplan config used

`/etc/netplan/00-installer-config.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlp3s0:
      dhcp4: true
      access-points:
        "SSID":
          password: "REDACTED"
```

YAML is whitespace-sensitive: spaces only, never tabs.

## What I learned

Linux networking fails in layers, and each layer fails differently:

| Layer       | Question                         | Tool                 |
| ----------- | -------------------------------- | -------------------- |
| Driver      | Does the interface exist at all? | `ip link`, `lspci`   |
| Carrier     | Is the radio up?                 | `ip link` (LOWER_UP) |
| Association | Are we actually joined to an AP? | `networkctl status`  |
| DHCP        | Did we get an address?           | `ip a` (inet)        |
| Routing     | Can we reach the internet?       | `ping 8.8.8.8`       |

Key insight: **"carrier is up" does not mean "connected"**. The interface can be
physically alive while being associated with nothing. That distinction cost me an
hour, and taught me more than any tutorial would have.

Also worth knowing: minimal server images ship without tools you assume are always
there (`iwlist`, `wpa_cli`, and in this case `wpa_supplicant` itself). "command
not found" usually means "not installed", not "broken".

## Note on redaction

SSID, WiFi password and local IPs are redacted throughout this repo.
Secrets don't belong in version control. Once committed, removing them in a later
commit is not enough, because they stay in the git history. That's exactly why
pipelines run secret scanners like `gitleaks`.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._
