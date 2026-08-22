<!-- # Homelab

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
- [x] Alerting rules (disk, memory, target down)
- [x] k3s (lightweight Kubernetes)
- [ ] CI/CD with GitHub Actions
- [ ] Security scanning in pipeline (tfsec, Trivy, gitleaks)

## Docs

- [01 - Ubuntu Server setup](docs/01-ubuntu-setup.md)
- [02 - WiFi troubleshooting: Broadcom BCM4360](docs/02-wifi-broadcom.md)
- [03 - Monitoring stack: Prometheus + Grafana](docs/03-monitoring.md)
- [04 - Server hardening: lid switch and IP stability](docs/04-server-hardening.md)
- [05 - Building a custom Docker image](docs/05-custom-dockerfile.md)
- [06 - Kubernetes with k3s](docs/06-k3s.md)

## Notes

Images are pinned to `:latest` for now — a known shortcut. Pinning to explicit
versions is on the list.

## Why this exists

Cloud infrastructure is easy to click together and hard to understand.
This lab is where I break things on purpose and learn how they actually work. -->


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

| Component | Technology |
|-----------|------------|
| OS | Ubuntu Server 24.04 LTS |
| Container runtime | containerd (via k3s) |
| Orchestration | k3s (lightweight Kubernetes) |
| Package management | Helm 3 |
| GitOps | ArgoCD |
| Monitoring | Prometheus + Grafana + node-exporter |
| Alerting | Prometheus alert rules |
| CI/CD | GitHub Actions |
| Security scanning | Trivy |
| Policy enforcement | Kyverno |

## Architecture

```
GitHub repo
    ↓
GitHub Actions (helm lint + template + Trivy scan)
    ↓
ArgoCD (GitOps — auto-sync on push)
    ↓
k3s cluster
    ├── monitoring namespace
    │   ├── Prometheus (Service Discovery via k8s API)
    │   ├── Grafana (PersistentVolume for dashboards)
    │   └── node-exporter (DaemonSet — one per node)
    ├── argocd namespace
    └── kyverno namespace (policy enforcement)
```

## Roadmap

- [x] Ubuntu Server install + SSH access
- [x] Docker installed, first containers running
- [x] Prometheus + Grafana via docker-compose
- [x] Alerting rules (disk, memory, target down)
- [x] k3s (lightweight Kubernetes)
- [x] Migrated monitoring stack from Docker Compose to Kubernetes
- [x] Helm chart for one-command deployment
- [x] GitHub Actions CI pipeline (lint, template validation, Trivy)
- [x] ArgoCD GitOps — push to repo = automatic cluster sync
- [x] Kyverno policy enforcement (disallow :latest tag)
- [x] Sealed Secrets
- [ ] Terraform

## Docs

- [01 - Ubuntu Server setup](docs/01-ubuntu-setup.md)
- [02 - WiFi troubleshooting: Broadcom BCM4360](docs/02-wifi-broadcom.md)
- [03 - Monitoring stack: Prometheus + Grafana](docs/03-monitoring.md)
- [04 - Server hardening: lid switch and IP stability](docs/04-server-hardening.md)
- [05 - Building a custom Docker image](docs/05-custom-dockerfile.md)
- [06 - Kubernetes with k3s](docs/06-k3s.md)
- [07 - Helm](docs/07-helm.md)
- [08 - GitHub Actions](docs/08-github-actions.md)
- [09 - ArgoCD](docs/09-argocd.md)
- [10 - Kyverno](docs/10-kyverno.md)
- [11 - Sealed Secrets](docs/11-selead-secrets.md)

## Why this exists

Cloud infrastructure is easy to click together and hard to understand.
This lab is where I break things on purpose and learn how they actually work.
