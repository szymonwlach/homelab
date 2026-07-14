# Homelab

> An 11-year-old laptop, wiped and turned into a real Linux server.
> Everything here was built, broken, and fixed by hand.

Self-hosted Linux homelab running on a 2015 MacBook Air.
Learning DevOps & platform engineering by building — not by watching tutorials.

**Started:** July 2026

## Hardware

- MacBook Air (Early 2015), Intel i5, 8 GB RAM, 250 GB SSD
- macOS wiped, running Ubuntu Server 24.04 LTS (headless)
- Managed remotely over SSH

## Current stack

- Ubuntu Server 24.04 LTS
- Docker (Engine + Compose plugin)
- Prometheus + node-exporter + Grafana — monitoring stack, defined in docker-compose
- Secrets kept in a local `.env`, excluded via `.gitignore`

## Roadmap

- [x] Ubuntu Server install + SSH access
- [x] Docker installed, first containers running
- [x] Prometheus + Grafana via docker-compose
- [ ] Alerting rules (disk, memory, target down)
- [ ] k3s (lightweight Kubernetes)
- [ ] CI/CD with GitHub Actions
- [ ] Security scanning in pipeline (tfsec, Trivy, gitleaks)

## Docs

- [01 - Ubuntu Server setup](docs/01-ubuntu-setup.md)
- [02 - WiFi troubleshooting: Broadcom BCM4360](docs/02-wifi-broadcom.md)
- [03 - Monitoring stack: Prometheus + Grafana](docs/03-monitoring.md)

## Notes

Images are pinned to `:latest` for now — a known shortcut. Pinning to explicit
versions is on the list.

## Why this exists

Cloud infrastructure is easy to click together and hard to understand.
This lab is where I break things on purpose and learn how they actually work.
