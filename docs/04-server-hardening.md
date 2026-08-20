# 04 - Server hardening: lid switch and IP stability

## Problem 1: closing the lid killed the SSH session

Every time the laptop's lid was closed, the remote SSH connection dropped
immediately, as if the server had gone offline. This happened even though
the server was meant to run headless, 24/7, with the lid closed.

## Diagnosis

Linux systems handle a laptop lid close event through `systemd-logind`,
which by default treats it the same as pressing the power button on a
desktop: suspend the machine. On a laptop turned into a server, this is
exactly the wrong behavior; the "screen" closing should mean nothing.

## Fix

```bash
sudo vim /etc/systemd/logind.conf
```

Uncommented and changed:

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Applied without a full reboot:

```bash
sudo systemctl restart systemd-logind
```

## Result

Closing the lid no longer suspends the machine or drops the SSH session.
Verified by closing the lid, waiting several minutes, and reconnecting via
SSH from a separate machine, with `docker ps` still showing all three
containers `Up`.

## Problem 2: the server's IP address changed on every restart

The server's IP address wasn't fixed. It changed slightly after each
reboot (DHCP reassigning a different address from the pool), which meant
having to look up the current address every time before connecting, and
broke any bookmarks, saved SSH commands, or links to Grafana that assumed
a fixed address.

## Fix

Set up a **DHCP address reservation** on the router itself, binding the
server's MAC address to a fixed IP:

- Router admin panel → Advanced → Network → DHCP Server → Address Reservation
- MAC Address: `F0-79-60-1E-67-82`
- Reserved IP: `192.168.0.114`

This is different from setting a static IP on the server itself: the
server still requests an address via DHCP as usual, but the router always
hands out the same one for that MAC address. No netplan changes needed.

## Result

The server now always comes up on the same IP after any restart. SSH
commands, Grafana links, and documentation can all reference a fixed
address without needing to check `ip a` first.

## What I learned

- A laptop repurposed as a server inherits laptop-specific power behavior
  (lid switch, sleep) that has to be explicitly disabled. It isn't a
  server just because it runs Ubuntu Server; it becomes one once these
  assumptions are removed.
- DHCP reservation on the router is the cleaner fix for IP stability
  compared to a static config on the machine itself: it's centralized,
  visible in one place, and survives OS reinstalls.
- Small infrastructure details like these are exactly what tutorials skip,
  and exactly what breaks a "toy" homelab versus a machine you can
  actually rely on.
