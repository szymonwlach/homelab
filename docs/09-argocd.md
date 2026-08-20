# 09 — ArgoCD

## Problem

GitHub Actions validates and scans on every push — but it doesn't deploy.
After the pipeline passed, the cluster still had to be updated manually:

```bash
git pull
helm upgrade monitoring k8s/charts/monitoring
```

Every change. Every time. By hand.

## What ArgoCD is

ArgoCD is a GitOps controller that runs inside the cluster and watches a Git
repository. When it detects a difference between what's in the repo and what's
running in the cluster, it closes the gap automatically.

No manual deploys. No SSH into the server. No `helm upgrade`.

## Git as the single source of truth

The core idea of GitOps: the repo is not just where the code lives — it's
the definition of what the cluster should look like at any given moment.

If it's in the repo, it should be running. If it's not in the repo, it
shouldn't exist. The cluster is just a reflection of the repo.

This matters because it makes the state of your infrastructure auditable,
reproducible, and reviewable. Every change goes through a commit. Every
commit has an author, a timestamp, and a message.

## The full pipeline

git push
↓
GitHub Actions (lint + template + Trivy)
↓
ArgoCD detects change in repo
↓
automatic cluster sync
↓
zero manual commands


## Application manifest

ArgoCD is configured through a Kubernetes object of kind `Application`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/szymonwlach/homelab
    targetRevision: main
    path: k8s/charts/monitoring
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

`repoURL` and `path` tell ArgoCD where to look. `targetRevision: main` means
it watches the main branch. Every push triggers a sync.

## syncPolicy

Three options that matter:

**`automated`** — ArgoCD syncs without waiting for a manual click. Without
this, you'd have to approve every sync in the UI.

**`prune: true`** — if a file is removed from the repo, the corresponding
resource is removed from the cluster. Without this, deleted manifests leave
orphaned resources behind.

**`selfHeal: true`** — if someone manually changes something in the cluster
with `kubectl`, ArgoCD detects the drift and reverts it to match the repo.
The repo wins. Always.

That last point is what makes GitOps different from just "deploying from Git".
It's not a one-way push — it's continuous reconciliation.

## What happened when selfHeal was tested

Changed a value directly in the cluster with `kubectl`. ArgoCD detected the
drift within minutes and reverted the change. The cluster went back to
matching the repo without any intervention.

That's the guarantee: whatever is in the repo is what runs. No exceptions,
no manual overrides that stick.

## ArgoCD checks every 3 minutes

By default, ArgoCD polls the repo every 3 minutes. For immediate sync, you
can either click Sync in the UI or trigger it via CLI:

```bash
kubectl -n argocd patch application monitoring \
  --type merge \
  -p '{"operation": {"sync": {"revision": "HEAD"}}}'
```

In production, this is solved with a webhook — GitHub notifies ArgoCD
instantly on push. Requires ArgoCD to be reachable from the internet, which
isn't the case in a local homelab.

## What I learned

- **GitOps is not just deploying from Git — it's continuous reconciliation.**
  The cluster constantly converges toward what the repo says, not just on push.
- **selfHeal means the repo is the only way to make lasting changes.**
  Manual `kubectl` edits are temporary. The repo always wins.
- **prune matters.** Without it, removing a file from the repo leaves the
  resource running in the cluster indefinitely.
- **Two applications, two concerns.** Monitoring and Kyverno policies are
  managed as separate ArgoCD Applications — each with its own sync cycle and
  its own path in the repo.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._