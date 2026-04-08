# NetworkPolicy

> Controls which Pods can communicate with each other and with external endpoints. By default, Pods in Kubernetes accept traffic from anywhere — NetworkPolicy lets you lock that down.

**apiVersion:** `networking.k8s.io/v1`  
**Kind:** `NetworkPolicy`

> ⚠️ NetworkPolicy requires a **CNI plugin that supports it** (Calico, Cilium, Weave Net). Flannel does NOT enforce NetworkPolicy. The policy objects can be created without a supporting CNI, but they have no effect.

---

## Default Behavior

```
No NetworkPolicy → All traffic allowed (all Pods can talk to all Pods)
NetworkPolicy applied to a Pod → Only explicitly allowed traffic is permitted
```

When a NetworkPolicy selects a Pod, it becomes **deny-all** for the affected directions, and only listed rules are allowed.

---

## Minimal Example: Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}               # Selects ALL Pods in the namespace
  policyTypes:
    - Ingress                   # Apply to inbound traffic
  # No ingress rules = deny all inbound
```

---

## Full Annotated Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production

spec:
  # ─── Target Pods ──────────────────────────────────────────────
  # Which Pods this policy applies to
  podSelector:
    matchLabels:
      app: backend
      tier: api

  # ─── Policy Types ─────────────────────────────────────────────
  # Which directions to control — if omitted, defaults to Ingress only
  policyTypes:
    - Ingress
    - Egress

  # ─── Ingress Rules ────────────────────────────────────────────
  # Allow inbound traffic from these sources
  ingress:
    # Rule 1: Allow from frontend Pods in same namespace
    - from:
        - podSelector:
            matchLabels:
              app: frontend

      ports:
        - protocol: TCP
          port: 8080

    # Rule 2: Allow from monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring

      ports:
        - protocol: TCP
          port: 9090              # Metrics scraping

    # Rule 3: Allow from specific Pods in specific namespace
    # (AND logic: must match BOTH selectors)
    - from:
        - namespaceSelector:
            matchLabels:
              environment: production
          podSelector:             # Same list item = AND
            matchLabels:
              role: api-gateway

      ports:
        - protocol: TCP
          port: 8080

    # Rule 4: Allow from external IP ranges
    - from:
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24       # Exclude this subnet

      ports:
        - protocol: TCP
          port: 443

  # ─── Egress Rules ─────────────────────────────────────────────
  # Allow outbound traffic to these destinations
  egress:
    # Allow DNS resolution (always needed!)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # Allow to database in same namespace
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Allow to Redis cache
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # Allow HTTPS to external services
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

---

## AND vs OR in `from`/`to`

This is the most confusing part of NetworkPolicy:

```yaml
# OR: Two separate list items = two separate rules
from:
  - podSelector:              # Rule A: from Pods with app=frontend
      matchLabels:
        app: frontend
  - namespaceSelector:        # OR Rule B: from namespace with env=staging
      matchLabels:
        env: staging

# AND: Same list item with both selectors = must match BOTH
from:
  - podSelector:              # AND: must be from namespace env=staging
      matchLabels:            #      AND have label app=frontend
        app: frontend
    namespaceSelector:
      matchLabels:
        env: staging
```

---

## Common Recipes

### Default deny all (namespace isolation)
```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes: [Egress]
```

### Allow only same-namespace traffic
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: {}         # Any Pod in the same namespace
```

### Allow all ingress to specific Pods
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-to-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes: [Ingress]
  ingress:
    - {}                          # Empty rule = allow all sources
```

### Three-tier app (frontend → backend → database)
```yaml
---
# Backend: accept only from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 8080
---
# Database: accept only from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 5432
```

---

## ⚠️ Gotchas

- **NetworkPolicy is additive** — multiple policies selecting the same Pod combine (union).
- Policies only affect the namespace they're in by default — use `namespaceSelector` to cross namespaces.
- **Always allow DNS egress** — forgetting UDP/TCP port 53 to kube-dns breaks name resolution.
- `podSelector: {}` selects ALL Pods in the namespace — this is intentional for default-deny policies.
- Policies don't affect the node's network or host-networked Pods.
- No NetworkPolicy = no enforcement, even if you created the object (wrong CNI plugin).

---

*← [Ingress](./02-ingress.md) | [Back to README](../README.md) | Next: [ConfigMap](../config/01-configmap.md) →*
