# 11 - Sealed Secrets

## Problem

`values.yaml` had the Grafana admin password hardcoded in plaintext:

```yaml
grafana:
  adminPassword: admin
```

The repository is public. Anyone with the link could read the password
directly from GitHub. Moving it into a Kubernetes Secret doesn't help —
Kubernetes Secrets only encode data in base64, which is not encryption:

```bash
echo "YWRtaW4=" | base64 -d
# admin
```

Two seconds. Base64 is a transport format, not a security mechanism.
Committing a Secret YAML to a public repository is the same as committing
the password in plaintext.

Kubernetes Secrets do solve a real problem — they separate credentials
from application code and control access inside the cluster through RBAC.
But they were never designed to be stored in version control.

## What Sealed Secrets is

Sealed Secrets uses asymmetric encryption to make secrets safe to commit:

- **Public key** — encrypts the secret (available locally via kubeseal)
- **Private key** — decrypts it (lives only inside the cluster controller)

The encrypted file can be pushed to a public repository. Without access
to the cluster's private key, it cannot be decrypted.

plaintext password
↓
kubeseal (encrypts with cluster's public key)
↓
SealedSecret YAML (safe to commit to public repo)
↓
sealed-secrets controller (decrypts with private key)
↓
regular Kubernetes Secret (available inside the cluster)


## Installation

Controller inside the cluster:

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
```

CLI tool on the server:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/kubeseal-0.27.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.27.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

## Creating a SealedSecret

One command — create the Secret and encrypt it immediately:

```bash
kubectl create secret generic grafana-secret \
  --namespace monitoring \
  --from-literal=adminPassword=admin \
  --dry-run=client \
  -o yaml | kubeseal --format yaml > k8s/secrets/grafana-sealed-secret.yaml
```

`--dry-run=client` means the Secret never touches the cluster — it goes
straight into the pipe. `kubeseal` encrypts it and writes the SealedSecret.

The output file is safe to commit:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: grafana-secret
  namespace: monitoring
spec:
  encryptedData:
    adminPassword: AgA+heB4JH0Upx561n69...  # unreadable without the private key
```

## Integrating with Grafana

The Deployment was updated to read the password from the Secret
instead of `values.yaml`:

```yaml
# BEFORE — plaintext in values.yaml:
grafana:
  adminPassword: admin

# AFTER — read from Secret:
env:
  - name: GF_SECURITY_ADMIN_PASSWORD
    valueFrom:
      secretKeyRef:
        name: grafana-secret
        key: adminPassword
```

`values.yaml` no longer contains any sensitive data. The password exists
only in encrypted form in the repository, and in decrypted form inside
the cluster — nowhere else.

## One critical note

The private key lives only in the cluster. If the cluster is destroyed
and rebuilt from scratch, existing SealedSecrets cannot be decrypted.
Back up the key:

```bash
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key-backup.yaml
```

Keep this file secure and outside the repository.

## What I learned

- **Base64 is not encryption.** A Kubernetes Secret in a public repository
  is a password with an extra decoding step.
- **Secrets belong in the repository — encrypted.** GitOps only works when
  everything is in Git. Sealed Secrets makes that safe.
- **Asymmetric encryption is the right tool here.** The public key can be
  shared freely. The private key never leaves the cluster.
- **Back up the private key.** Lose it and you lose access to every secret
  in the repository.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._