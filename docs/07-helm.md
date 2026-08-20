# 07 — Helm

## Problem

After migrating the monitoring stack to Kubernetes, deploying it meant running
`kubectl apply -f` on nine separate files, in the right order, every time.
One command per manifest. No history, no rollback, no way to change a value
without editing a file and reapplying everything.

## What Helm is

Helm is a package manager for Kubernetes. Instead of managing nine YAML files
separately, you pack them into a single **chart** and deploy everything with
one command:

```bash
helm install monitoring k8s/charts/monitoring/
```

Prometheus, Grafana, node-exporter, RBAC, ConfigMaps, Services — all of it,
in one shot. That was the moment it clicked.

## Chart structure

k8s/charts/monitoring/
Chart.yaml # metadata: name, version, description
values.yaml # default values — the only file you edit day-to-day
templates/ # your existing YAML manifests, now parametrized
namespace.yaml
prometheus-deployment.yaml
grafana-deployment.yaml
...


`Chart.yaml` tells Helm what the chart is. `values.yaml` holds the defaults.
`templates/` holds the manifests, with placeholders where values go.

## values.yaml and templating

Instead of hardcoding image versions and replica counts directly in manifests,
you reference them from `values.yaml`:

```yaml
# values.yaml
prometheus:
  image: prom/prometheus:v3.3.1
  replicas: 1

grafana:
  image: grafana/grafana:12.0.1
  adminPassword: admin
  storage: 1Gi
```

```yaml
# templates/prometheus-deployment.yaml
image: {{ .Values.prometheus.image }}
replicas: {{ .Values.prometheus.replicas }}
```

Helm substitutes the placeholders at deploy time. To override a value without
editing the file:

```bash
helm install monitoring k8s/charts/monitoring --set grafana.replicas=3
```

## Versioning and rollback

Every `helm upgrade` creates a new revision. The full history is kept:

```bash
helm history monitoring
REVISION  STATUS      CHART              DESCRIPTION
1         superseded  monitoring-0.1.0   Install complete
2         superseded  monitoring-0.2.0   Upgrade complete
3         deployed    monitoring-0.1.0   Rollback to 1
```

Rolling back to any previous state is one command:

```bash
helm rollback monitoring 1
```

In production, this is the difference between a bad deploy taking down the
service for an hour and being back in thirty seconds.

## Validating before deploying

Two commands worth running before every deploy:

```bash
helm lint k8s/charts/monitoring      # checks chart structure and YAML syntax
helm template monitoring k8s/charts/monitoring  # renders the final manifests
```

`helm template` is the equivalent of `terraform plan` — you see exactly what
will be applied before anything touches the cluster.

## What I learned

- **One command beats nine.** Packaging manifests into a chart removes an
  entire class of human error.
- **values.yaml is the only file you touch day-to-day.** Everything else stays
  stable.
- **Rollback is a first-class feature, not an afterthought.** Helm tracks every
  revision automatically.
- **Validate before you apply.** `helm lint` catches structure errors,
  `helm template` catches template errors. Both are faster than finding out
  after the fact.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._