# 01 — Ubuntu Server setup on MacBook Air (2015)

## Goal

Turn an 11-year-old MacBook Air into a headless Linux server,
managed over SSH from a Windows machine.

## Steps

1. Downloaded Ubuntu Server 24.04 LTS ISO
2. Flashed to USB with balenaEtcher
3. Booted MacBook holding Option (⌥), selected "EFI Boot"
4. Installer: selected **English (US)** keyboard layout, even though the physical
   keyboard is German (QWERTZ). Entire disk wiped.
5. **Enabled "Install OpenSSH server"** — without this, no remote access
6. Rebooted, removed installation media

## Result

Headless Ubuntu Server, accessible via `ssh szymon@<ip>` from PowerShell.

## Notes / gotchas

- The laptop has a **physical German (QWERTZ) keyboard**, but I configured the
  system with a **US layout** — so the key caps don't match what's typed
  (Y/Z swapped, special characters in different places).
  Used letters + digits only for the password to avoid layout surprises during
  first login.
- This stops mattering once SSH works — from then on you type on your own
  keyboard and the server's layout is irrelevant.
- Network did NOT work out of the box — see [02-wifi-broadcom.md](02-wifi-broadcom.md)
