# ReplicaSet

> Ensures a stable set of replica Pods is running at any given time. Usually managed by a Deployment — use a ReplicaSet directly only when you don't need rolling updates.

**apiVersion:** `apps/v1`  
**Kind:** `ReplicaSet`

> ⚠️ **In most cases, use a [Deployment](./02-deployment.md) instead.** Deployments manage ReplicaSets and add rolling updates, rollback, and revision history on top.

---

## Minimal Example

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

---

## Full Annotated Example

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
  namespace: default
  labels:
    app: my-app
    tier: frontend

spec:
  # ─── Desired Replicas ─────────────────────────────────────────
  replicas: 3

  # ─── Selector ─────────────────────────────────────────────────
  # IMMUTABLE after creation — must match template.metadata.labels
  selector:
    matchLabels:
      app: my-app
    # Advanced selector:
    # matchExpressions:
    #   - key: app
    #     operator: In
    #     values: [my-app]
    #   - key: tier
    #     operator: NotIn
    #     values: [test]
    #   - key: environment
    #     operator: Exists

  # ─── Pod Template ─────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: my-app      # MUST match selector.matchLabels
        tier: frontend
    spec:
      containers:
        - name: app
          image: my-app:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

---

## How ReplicaSet Works

```
ReplicaSet Controller loop:
  1. Count Pods matching selector
  2. If count < replicas → create new Pods
  3. If count > replicas → delete excess Pods
  4. If a Pod dies    → create replacement
```

The ReplicaSet **adopts** any existing Pod whose labels match its selector — even Pods it didn't create. This can cause unexpected behavior if label selectors are too broad.

---

## matchExpressions Operators

| Operator | Meaning |
|----------|---------|
| `In` | Label value is in the list |
| `NotIn` | Label value is NOT in the list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does NOT exist |

```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values: [frontend, web]
    - key: environment
      operator: NotIn
      values: [test, dev]
    - key: active
      operator: Exists
```

---

## kubectl Commands

```bash
# Scale
kubectl scale replicaset my-app-rs --replicas=5

# Check status
kubectl get rs my-app-rs

# Describe (shows events)
kubectl describe rs my-app-rs

# See which Deployment owns this RS
kubectl get rs my-app-rs -o yaml | grep ownerReferences
```

---

## ⚠️ Gotchas

- **Never modify the Pod template** of a ReplicaSet directly — existing Pods won't update. Use a Deployment for that.
- `selector` is **immutable** after creation.
- The ReplicaSet doesn't distinguish between a Pod it created and one it adopted — it just counts matching labels.
- If you delete a ReplicaSet, all Pods it owns are deleted unless you use `--cascade=orphan`.

---

*← [Deployment](./02-deployment.md) | [Back to README](../README.md) | Next: [StatefulSet](./04-statefulset.md) →*
