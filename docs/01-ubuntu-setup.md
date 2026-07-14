# 01 — Ubuntu Server setup on MacBook Air (2015)

## Goal

Turn an 11-year-old MacBook Air into a headless Linux server,
managed over SSH from a Windows machine.

## Steps

1. Downloaded Ubuntu Server 24.04 LTS ISO
2. Flashed to USB with balenaEtcher
3. Booted MacBook holding Option (⌥), selected "EFI Boot"
4. Installer: chose **English (US)** keyboard layout despite the physical German
   (QWERTZ) keyboard. The US layout puts the characters you actually need in a
   terminal (`/`, `|`, `-`, `{}`) in their standard positions instead of behind
   AltGr combos. Entire disk wiped.
5. **Enabled "Install OpenSSH server"**. Without this, no remote access.
6. Rebooted, removed installation media

## Result

Headless Ubuntu Server, accessible via `ssh szymon@<ip>` from PowerShell.

## Notes / gotchas

- Physical keyboard is German (QWERTZ); the system is configured as US layout.
  The key caps don't match what's typed (Y/Z swapped, special chars moved), but
  typing shell commands is far easier. A deliberate trade-off.
- Used letters + digits only for the password, to avoid layout surprises at first
  login.
- Irrelevant after SSH works. From then on you type on your own keyboard and the
  server's layout stops mattering.
- Network did NOT work out of the box. See [02-wifi-broadcom.md](02-wifi-broadcom.md)
