# StatefulSet

> Manages stateful applications — each Pod has a stable, unique identity (name, network, storage) that persists across rescheduling. Use for databases, distributed systems, and any app where Pod identity matters.

**apiVersion:** `apps/v1`  
**Kind:** `StatefulSet`

---

## When to Use StatefulSet vs Deployment

| Need | Use |
|------|-----|
| Stable Pod names (`pod-0`, `pod-1`) | StatefulSet |
| Stable DNS per Pod | StatefulSet |
| Persistent storage per Pod | StatefulSet |
| Ordered, graceful rolling updates | StatefulSet |
| Stateless, all Pods are identical | Deployment |

---

## Minimal Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # Required — must match a Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
```

---

## Full Annotated Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres

spec:
  # ─── Required: Name of the governing Headless Service ─────────
  # Each Pod gets DNS: <pod-name>.<serviceName>.<namespace>.svc.cluster.local
  serviceName: postgres-headless

  replicas: 3                     # Pods: postgres-0, postgres-1, postgres-2

  selector:
    matchLabels:
      app: postgres               # Must match template.metadata.labels

  # ─── Update Strategy ──────────────────────────────────────────
  updateStrategy:
    type: RollingUpdate           # RollingUpdate | OnDelete
    rollingUpdate:
      partition: 0                # Only update Pods with ordinal >= partition
      # partition: 2 → only update postgres-2, leaving postgres-0,1 unchanged
      # Useful for canary updates on StatefulSets

  # ─── Pod Management ───────────────────────────────────────────
  podManagementPolicy: OrderedReady   # OrderedReady | Parallel
  # OrderedReady: start/stop in order (0→1→2 up, 2→1→0 down) — default
  # Parallel: start/stop all at once (faster, but ordering guarantees lost)

  # ─── Revision History ─────────────────────────────────────────
  revisionHistoryLimit: 10

  # ─── Pod Template ─────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: postgres

    spec:
      terminationGracePeriodSeconds: 60

      securityContext:
        fsGroup: 999              # postgres user group owns mounted volumes
        runAsUser: 999
        runAsNonRoot: true

      # Init container: set correct permissions on data dir
      initContainers:
        - name: init-data
          image: busybox
          command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data"]
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data

      containers:
        - name: postgres
          image: postgres:15.3
          imagePullPolicy: IfNotPresent

          ports:
            - name: postgres
              containerPort: 5432

          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: database_name
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata

            # StatefulSet Pods know their own identity via Downward API
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"

          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /etc/postgresql/conf.d
              readOnly: true

          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 10

      volumes:
        - name: config
          configMap:
            name: postgres-config

  # ─── Volume Claim Templates ───────────────────────────────────
  # Each Pod gets its OWN PVC: data-postgres-0, data-postgres-1, data-postgres-2
  # PVCs are NOT deleted when the StatefulSet is deleted!
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: postgres
      spec:
        accessModes:
          - ReadWriteOnce           # Each pod has exclusive access to its volume
        storageClassName: fast-ssd  # Reference a StorageClass
        resources:
          requests:
            storage: 50Gi

---
# ─── Required: Headless Service ───────────────────────────────────────────────
# clusterIP: None makes this a "headless" service — no load balancing,
# each Pod gets its own DNS record instead
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None               # ← This makes it headless
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432

---
# ─── Optional: Regular Service for clients ────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## Pod Identity

StatefulSet Pods get **stable, predictable identities**:

```
Pod names:     postgres-0  postgres-1  postgres-2
DNS names:     postgres-0.postgres-headless.production.svc.cluster.local
               postgres-1.postgres-headless.production.svc.cluster.local
               postgres-2.postgres-headless.production.svc.cluster.local

PVC names:     data-postgres-0
               data-postgres-1
               data-postgres-2
```

Pods can find their peers:
```bash
# In an init script, find the replica number from the pod name
ORDINAL=${POD_NAME##*-}           # "postgres-2" → "2"
# Connect to the primary (always ordinal 0 in many setups)
psql -h postgres-0.postgres-headless ...
```

---

## Ordered vs Parallel Pod Management

```
OrderedReady (default):
  Scale up:   postgres-0 → wait ready → postgres-1 → wait ready → postgres-2
  Scale down: postgres-2 → wait gone  → postgres-1 → wait gone  → postgres-0

Parallel:
  Scale up:   postgres-0, postgres-1, postgres-2 all start simultaneously
  Scale down: postgres-2, postgres-1, postgres-0 all stop simultaneously
```

---

## Partition Updates (Canary)

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2    # Only postgres-2 gets the new version
                    # postgres-0 and postgres-1 stay on old version
                    # Useful to test on one replica before rolling all
```

---

## kubectl Commands

```bash
# Scale
kubectl scale statefulset postgres --replicas=5

# Trigger rolling update
kubectl rollout restart statefulset/postgres

# Watch pod creation (ordered)
kubectl get pods -w -l app=postgres

# View PVCs created by the StatefulSet
kubectl get pvc -l app=postgres

# Delete StatefulSet but keep Pods and PVCs
kubectl delete statefulset postgres --cascade=orphan
```

---

## ⚠️ Gotchas

- **`serviceName` is required** and must reference an existing Headless Service (or you can create it first).
- **PVCs created by `volumeClaimTemplates` are never deleted** when the StatefulSet is deleted — you must delete them manually.
- Scaling down to 0 doesn't delete PVCs, so you can safely scale back up and retain data.
- `OrderedReady` means a failed Pod blocks all subsequent operations — if `postgres-1` fails, `postgres-2` won't start.
- Don't use `ReadWriteMany` PVCs with StatefulSets expecting data isolation — each Pod should have `ReadWriteOnce`.

---

*← [ReplicaSet](./03-replicaset.md) | [Back to README](../README.md) | Next: [DaemonSet](./05-daemonset.md) →*
