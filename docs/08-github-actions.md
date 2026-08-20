# 08 - GitHub Actions

## Problem

Every change to the Helm chart had to be validated manually before applying
it to the cluster. Nothing stopped a broken manifest or an image with critical
vulnerabilities from reaching production. No automation, no safety net.

## What GitHub Actions is

GitHub Actions is a CI platform built into GitHub. You define a workflow in
a YAML file under `.github/workflows/` and it runs automatically on every
`git push`. No external tools to set up, no separate CI server to maintain.

## The pipeline

git push
↓
Checkout code
↓
Install Helm
↓
helm lint — catches structural and syntax errors
↓
helm template — renders all manifests, catches template errors
↓
Trivy scan — checks container images for known vulnerabilities


Five steps. The first four are free. The last one is what makes it a
DevSecOps pipeline rather than just CI.

## Workflow file

```yaml
name: Helm Lint

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v3

      - name: Lint Helm chart
        run: helm lint k8s/charts/monitoring

      - name: Template Helm chart
        run: helm template monitoring k8s/charts/monitoring

      - name: Scan images with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: prom/prometheus:latest
          format: table
          exit-code: 0
          severity: CRITICAL,HIGH
```

## `uses` vs `run`

`run` executes a plain shell command — exactly what you'd type in a terminal:

```yaml
run: helm lint k8s/charts/monitoring
```

`uses` pulls in a pre-built action written by someone else — a reusable block
of logic you'd otherwise have to write yourself:

```yaml
uses: actions/checkout@v4   # clones your repo onto the runner
uses: azure/setup-helm@v3   # installs Helm
```

The format is `owner/repo@version`. `@v4` pins the version — without it,
a change upstream could silently break your pipeline.

Where to find actions: [github.com/marketplace](https://github.com/marketplace?type=actions).
In practice, searching "setup helm github actions" returns the right one as
the first result.

## Trivy

Trivy scans container images against a database of known CVEs. Running it in
CI means every push checks whether the images you're deploying have known
vulnerabilities — before anything reaches the cluster.

`exit-code: 0` means the pipeline passes even when vulnerabilities are found.
The findings are visible in the logs. Changing it to `exit-code: 1` would
block the deploy on any HIGH or CRITICAL finding — the right setting for
production.

First scan result on `prom/prometheus:latest`:

Total: 8 (HIGH: 8, CRITICAL: 0)


Eight HIGH vulnerabilities in the Go standard library — all fixable by
pinning to a newer image version, which is exactly what Kyverno enforces
downstream.

## What I learned

- **Automation removes an entire class of human error.** A broken chart or
  a vulnerable image can't reach the cluster if the pipeline catches it first.
- **`uses` is a dependency, `run` is a command.** Pin versions on both.
- **Security scanning belongs in CI, not as an afterthought.** Finding a
  vulnerability before deploy costs nothing. Finding it after costs a lot.
- **`exit-code: 0` vs `exit-code: 1` is a policy decision.** Start with
  Audit, then move to Enforce once you understand what's in your images.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._