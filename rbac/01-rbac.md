# RBAC: ServiceAccount, Role, ClusterRole, RoleBinding & ClusterRoleBinding

> Role-Based Access Control (RBAC) controls who can do what with Kubernetes resources. Every action in Kubernetes — `get`, `create`, `delete` — requires explicit permission.

---

## RBAC Model

```
Subject (who)  →  RoleBinding  →  Role (what)
ServiceAccount         or          ClusterRole
User           →  ClusterRoleBinding
Group
```

| Object | Scope | What it does |
|--------|-------|-------------|
| `ServiceAccount` | Namespace | Identity for a Pod/process |
| `Role` | Namespace | Defines permissions within a namespace |
| `ClusterRole` | Cluster-wide | Defines permissions across all namespaces (or for cluster resources) |
| `RoleBinding` | Namespace | Grants a Role (or ClusterRole) to subjects in a namespace |
| `ClusterRoleBinding` | Cluster-wide | Grants a ClusterRole to subjects cluster-wide |

---

# ServiceAccount

**apiVersion:** `v1` | **Kind:** `ServiceAccount`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  labels:
    app: my-app
  annotations:
    # AWS IRSA (IAM Roles for Service Accounts)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
    # GCP Workload Identity
    # iam.gke.io/gcp-service-account: my-app@my-project.iam.gserviceaccount.com

# Don't auto-mount the service account token if the Pod doesn't need it
automountServiceAccountToken: false

# Pull secrets to attach to all Pods using this SA
imagePullSecrets:
  - name: registry-credentials
```

Use in Pod:
```yaml
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: false   # Also disable at Pod level
```

---

# Role

**apiVersion:** `rbac.authorization.k8s.io/v1` | **Kind:** `Role`

Grants permissions **within a single namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
  labels:
    app: my-app

rules:
  # ─── Rule format ────────────────────────────────────────────
  # apiGroups: which API group (core group = "")
  # resources: which resource types
  # verbs: which actions
  # resourceNames: (optional) restrict to specific named resources

  # Read Pods and their logs
  - apiGroups: [""]             # Core group: Pod, Service, ConfigMap, Secret, etc.
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

  # Read ConfigMaps
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

  # Read/write a specific Secret only
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    resourceNames: ["app-secrets"]   # Restrict to this named resource only

  # Full access to Deployments
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Read events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
```

---

# ClusterRole

**apiVersion:** `rbac.authorization.k8s.io/v1` | **Kind:** `ClusterRole`

Grants permissions **cluster-wide** or for **non-namespaced resources** (Nodes, PersistentVolumes, etc.).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
  labels:
    app: my-app

rules:
  # Read all resources in all namespaces
  - apiGroups: ["", "apps", "batch", "extensions"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]

  # Access non-namespaced resources
  - apiGroups: [""]
    resources: ["nodes", "persistentvolumes", "namespaces"]
    verbs: ["get", "list", "watch"]

  # Access storage resources
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "volumeattachments"]
    verbs: ["get", "list", "watch"]

  # Access CRDs
  - apiGroups: ["apiextensions.k8s.io"]
    resources: ["customresourcedefinitions"]
    verbs: ["get", "list", "watch"]

  # Metrics
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]

  # Non-resource URLs (e.g., /healthz, /metrics)
  - nonResourceURLs: ["/healthz", "/metrics", "/readyz"]
    verbs: ["get"]
```

---

# RoleBinding

**apiVersion:** `rbac.authorization.k8s.io/v1` | **Kind:** `RoleBinding`

Grants a Role **or** ClusterRole to subjects within a **specific namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production           # Permission only applies in this namespace

# ─── Role Reference ───────────────────────────────────────────────
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role                      # Role or ClusterRole
  name: app-role                  # Name of the Role/ClusterRole to grant

# ─── Subjects ─────────────────────────────────────────────────────
# Who gets these permissions
subjects:
  # ServiceAccount (most common for Pods)
  - kind: ServiceAccount
    name: my-app-sa
    namespace: production         # Required for ServiceAccount

  # Human user
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io

  # Group
  - kind: Group
    name: backend-team
    apiGroup: rbac.authorization.k8s.io
```

---

# ClusterRoleBinding

**apiVersion:** `rbac.authorization.k8s.io/v1` | **Kind:** `ClusterRoleBinding`

Grants a ClusterRole to subjects **cluster-wide** (all namespaces).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-reader-binding

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-reader

subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring

  - kind: Group
    name: cluster-admins
    apiGroup: rbac.authorization.k8s.io
```

---

## Complete Example: App with Minimal RBAC

```yaml
---
# ServiceAccount for the Pod
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
automountServiceAccountToken: false

---
# Role: read ConfigMaps and Secrets in production namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["app-secrets"]

---
# RoleBinding: grant the role to the SA
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-app-role
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: production
```

---

## Common Verbs Reference

| Verb | HTTP Equivalent | Description |
|------|----------------|-------------|
| `get` | GET (single) | Read a specific resource |
| `list` | GET (collection) | List all resources of a type |
| `watch` | GET+watch | Stream changes to resources |
| `create` | POST | Create a new resource |
| `update` | PUT | Replace an existing resource |
| `patch` | PATCH | Partially update a resource |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE | Delete all of a resource type |
| `*` | All | All verbs (admin) |

## Common API Groups

| API Group | Resources |
|-----------|----------|
| `""` (core) | pods, services, configmaps, secrets, namespaces, nodes, PV, PVC |
| `apps` | deployments, replicasets, statefulsets, daemonsets |
| `batch` | jobs, cronjobs |
| `networking.k8s.io` | ingresses, networkpolicies |
| `rbac.authorization.k8s.io` | roles, rolebindings, clusterroles |
| `autoscaling` | horizontalpodautoscalers |
| `storage.k8s.io` | storageclasses, volumeattachments |
| `policy` | poddisruptionbudgets |
| `apiextensions.k8s.io` | customresourcedefinitions |
| `*` | All groups |

---

## kubectl Commands

```bash
# Check what a ServiceAccount can do
kubectl auth can-i get pods --as=system:serviceaccount:production:my-app-sa -n production

# Check all permissions for a user
kubectl auth can-i --list --as=jane@example.com -n production

# View all roles in a namespace
kubectl get roles,rolebindings -n production

# View cluster-level roles
kubectl get clusterroles,clusterrolebindings

# Describe a role (shows all rules)
kubectl describe role my-app-role -n production

# Create a role quickly
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n production
```

---

## ⚠️ Gotchas

- `RoleBinding` referencing a `ClusterRole` only grants permissions **in the binding's namespace**.
- `ClusterRoleBinding` grants permissions in **all namespaces** — use carefully.
- `automountServiceAccountToken: false` should be set on both the SA and the Pod for full effect.
- RBAC is additive — no "deny" rules. If any binding allows it, it's allowed.
- Subject names are case-sensitive — `Jane` ≠ `jane`.
- `kubectl auth can-i` is your best debugging tool for RBAC issues.

---

*← [Storage](../storage/01-storage.md) | [Back to README](../README.md) | Next: [Namespace & ResourceQuota](../cluster/01-namespace.md) →*
