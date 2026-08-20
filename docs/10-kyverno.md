# 10 — Kyverno

## Problem

Nothing stopped anyone from deploying a container image tagged `:latest`.
Images tagged `:latest` are unreliable — you don't know what version you're
actually running, and they often contain unpatched vulnerabilities. Trivy
confirmed this: `prom/prometheus:latest` had 8 HIGH vulnerabilities.

Knowing about the problem and enforcing a fix are two different things.

## What Kyverno is

Kyverno is a policy engine for Kubernetes. You define rules — what is and
isn't allowed in the cluster — and Kyverno enforces them automatically on
every resource that gets created or updated.

It works as an **Admission Webhook**: every request to the Kubernetes API
passes through Kyverno before it's accepted. If the resource violates a
policy, Kyverno rejects it before the pod ever starts.

kubectl apply / helm install
↓
Kubernetes API
↓
Kyverno checks policies
↓
✅ compliant → resource created
❌ violation → request rejected with explanation


## The policy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Using the latest tag is prohibited. Use a specific image version."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

`ClusterPolicy` applies across all namespaces. `match` defines which resources
the rule targets — in this case, every Pod. The `pattern` rejects any image
that ends with `:latest`.

## Audit vs Enforce

Kyverno has two modes:

**`Audit`** — violations are recorded and visible in policy reports, but
nothing is blocked. Resources are created normally. Use this to understand
what would break before enforcing anything.

**`Enforce`** — violations are blocked at the API level. The resource is
rejected before it's created. No exceptions.

Starting with `Audit` was the right call. The existing monitoring stack —
Prometheus, Grafana, node-exporter — all used `:latest`. Switching straight
to `Enforce` would have broken every pod on the next restart.

The correct sequence:
1. Deploy policy in `Audit` mode
2. Check policy reports to see what's non-compliant
3. Fix the violations (pin image versions)
4. Switch to `Enforce`

## Policy reports

In `Audit` mode, Kyverno generates reports for every resource:

```bash
kubectl get policyreport -n monitoring
```

NAME KIND NAME PASS FAIL
... Deployment prometheus 0 1
... Deployment grafana 0 1
... DaemonSet node-exporter 0 1


Every `FAIL` was an image tagged `:latest`. After pinning all versions in
`values.yaml`, the reports flipped to `PASS`.

## Testing Enforce mode

After switching to `Enforce`, tried deploying a pod with `:latest`:

```bash
kubectl run test-latest --image=nginx:latest -n monitoring
```

Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
resource Pod/monitoring/test-latest was blocked due to the following policies
disallow-latest-tag:
require-image-tag: 'validation error: Using the latest tag is prohibited.
Use a specific image version.'


Blocked before the pod was created. The error message tells you exactly which
policy rejected it and why.

## Managed by ArgoCD

The policy lives in `k8s/kyverno/` and is managed by a separate ArgoCD
Application. Any change to the policy file triggers an automatic sync —
the same GitOps flow as the monitoring stack.

## What I learned

- **Knowing about a problem and enforcing a fix are different things.**
  Trivy found the vulnerabilities. Kyverno prevents them from being deployed.
- **Start with Audit, move to Enforce.** Jumping straight to Enforce on an
  existing cluster breaks things. Audit first shows you what needs fixing.
- **Policy as code belongs in the repo.** Kyverno policies are YAML files,
  versioned in Git, deployed by ArgoCD. No manual configuration, no drift.
- **The error message is the documentation.** When Enforce blocks a deploy,
  the rejection message tells you exactly what policy was violated and why.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._