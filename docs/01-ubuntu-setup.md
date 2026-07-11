# 01 — Ubuntu Server setup on MacBook Air (2015)

## Goal

Turn an 11-year-old MacBook Air into a headless Linux server,
managed over SSH from a Windows machine.

## Steps

1. Downloaded Ubuntu Server 24.04 LTS ISO
2. Flashed to USB with balenaEtcher
3. Booted MacBook holding Option (⌥), selected "EFI Boot"
4. Installer: German keyboard layout (physical QWERTZ), entire disk wiped
5. **Enabled "Install OpenSSH server"** — without this, no remote access
6. Rebooted, removed installation media

## Result

Headless Ubuntu Server, accessible via `ssh szymon@<ip>` from PowerShell.

## Notes / gotchas

- Physical keyboard is German (QWERTZ) — Y/Z swapped, special chars remapped.
  Used letters+digits only for the password to avoid layout surprises.
- Network did NOT work out of the box — see [02-wifi-broadcom.md](02-wifi-broadcom.md)
