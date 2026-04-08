# Secret

> Stores sensitive data such as passwords, OAuth tokens, SSH keys, and TLS certificates. Similar to ConfigMap but with base64 encoding and additional access controls.

**apiVersion:** `v1`  
**Kind:** `Secret`

> ⚠️ **Secrets are NOT encrypted by default** — they are only base64-encoded. Enable [Encryption at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) and use RBAC to restrict access.

---

## Secret Types

| Type | Use Case |
|------|----------|
| `Opaque` | Arbitrary user-defined data (default) |
| `kubernetes.io/service-account-token` | ServiceAccount token |
| `kubernetes.io/dockerconfigjson` | Private registry credentials |
| `kubernetes.io/tls` | TLS certificate and private key |
| `kubernetes.io/ssh-auth` | SSH private key |
| `kubernetes.io/basic-auth` | Basic authentication credentials |
| `bootstrap.kubernetes.io/token` | Node bootstrap token |

---

## Minimal Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                       # Automatically base64-encoded on creation
  username: admin
  password: S3cur3P@ssw0rd!
```

---

## Full Annotated Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
  labels:
    app: my-app

type: Opaque

# ─── stringData: plain text (gets base64-encoded automatically) ───
# Easier to write and review than base64
stringData:
  DB_USERNAME: "appuser"
  DB_PASSWORD: "S3cur3P@ss!"
  API_KEY: "sk-abc123def456"
  REDIS_PASSWORD: "redis-secret"

# ─── data: already base64-encoded values ──────────────────────────
# echo -n "value" | base64
data:
  JWT_SECRET: "bXktc2VjcmV0LWp3dC1rZXk="   # base64("my-secret-jwt-key")

# ─── immutable: prevent changes (k8s 1.21+) ──────────────────────
immutable: false
```

---

## TLS Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

Create from files:
```bash
kubectl create secret tls myapp-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

---

## Docker Registry Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

Create from docker config:
```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@example.com
```

Use in Pod spec:
```yaml
spec:
  imagePullSecrets:
    - name: registry-credentials
```

---

## Using Secrets in Pods

### Method 1: Single key as env var
```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: DB_PASSWORD
        optional: false
```

### Method 2: All keys as env vars
```yaml
envFrom:
  - secretRef:
      name: app-secrets
      optional: false
```

### Method 3: As volume (files)
```yaml
volumes:
  - name: secrets-vol
    secret:
      secretName: app-secrets
      defaultMode: 0400           # Read-only by owner (recommended)

containers:
  - volumeMounts:
      - name: secrets-vol
        mountPath: /etc/secrets
        readOnly: true
        # Creates files: /etc/secrets/DB_PASSWORD, /etc/secrets/API_KEY, etc.
```

### Method 4: Specific file (subPath)
```yaml
volumes:
  - name: tls-secret
    secret:
      secretName: myapp-tls

containers:
  - volumeMounts:
      - name: tls-secret
        mountPath: /etc/ssl/certs/tls.crt
        subPath: tls.crt
        readOnly: true
      - name: tls-secret
        mountPath: /etc/ssl/private/tls.key
        subPath: tls.key
        readOnly: true
```

---

## Creating Secrets with kubectl

```bash
# From literal values (not stored in shell history with careful quoting)
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD='S3cur3P@ss!' \
  --from-literal=API_KEY='sk-abc123'

# From a file
kubectl create secret generic ssh-key \
  --from-file=id_rsa=~/.ssh/id_rsa

# From multiple files
kubectl create secret generic app-certs \
  --from-file=./certs/

# From an env file (KEY=VALUE format)
kubectl create secret generic app-secrets \
  --from-env-file=.env.production
```

---

## Encode/Decode Base64

```bash
# Encode
echo -n "my-password" | base64
# Output: bXktcGFzc3dvcmQ=

# Decode
echo "bXktcGFzc3dvcmQ=" | base64 --decode
# Output: my-password
```

> ⚠️ `-n` flag suppresses the trailing newline — always use it, or your decoded value will have a `\n` appended.

---

## Best Practices

### 1. Use External Secret Managers
Consider using tools that sync secrets from external stores:
- **External Secrets Operator** → AWS Secrets Manager, GCP Secret Manager, Vault
- **Sealed Secrets** → Encrypt secrets for safe git storage
- **Vault Agent Injector** → HashiCorp Vault integration

### 2. Restrict Access with RBAC
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["app-secrets"]   # Only this specific secret
```

### 3. Enable Encryption at Rest
Add to kube-apiserver config (`--encryption-provider-config`):
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: [secrets]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-32-byte-key>
      - identity: {}
```

---

## ⚠️ Gotchas

- Secrets are only base64-encoded, **not encrypted** — treat them as sensitive but not secret without encryption at rest.
- `kubectl get secret my-secret -o yaml` reveals the base64 content — restrict RBAC `get` on secrets.
- Secrets in Pod environment variables are accessible to all processes in the container.
- Volume-mounted secrets are more secure — files can have strict permissions (`0400`).
- `subPath` volume mounts do NOT auto-update when the Secret changes.
- Secrets must be in the **same namespace** as the Pod that uses them.

---

*← [ConfigMap](./01-configmap.md) | [Back to README](../README.md) | Next: [PersistentVolume](../storage/01-persistentvolume.md) →*
