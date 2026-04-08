# Deployment

> Manages stateless applications by maintaining a desired number of Pod replicas. Handles rolling updates, rollbacks, and scaling.

**apiVersion:** `apps/v1`  
**Kind:** `Deployment`

---

## Minimal Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
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
          ports:
            - containerPort: 80
```

---

## Full Annotated Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    team: backend
  annotations:
    deployment.kubernetes.io/revision: "3"
    kubernetes.io/change-cause: "Bumped image to v2.1.0"  # kubectl rollout history shows this

spec:
  # ─── Replica Count ────────────────────────────────────────────
  replicas: 3                     # Desired number of Pods

  # ─── Selector ─────────────────────────────────────────────────
  # IMMUTABLE after creation — must match template.metadata.labels
  selector:
    matchLabels:
      app: my-app
      # matchExpressions is also supported:
      # matchExpressions:
      #   - key: app
      #     operator: In
      #     values: [my-app, my-app-v2]

  # ─── Update Strategy ──────────────────────────────────────────
  strategy:
    type: RollingUpdate             # RollingUpdate | Recreate
    rollingUpdate:
      maxSurge: 1                   # Extra Pods above desired during update (int or %)
      maxUnavailable: 0             # Pods that can be unavailable during update (int or %)
      # maxSurge: 25%               # Percentage form also works
      # maxUnavailable: 25%

  # ─── Revision History ─────────────────────────────────────────
  revisionHistoryLimit: 10          # How many old ReplicaSets to keep for rollback

  # ─── Min Ready ────────────────────────────────────────────────
  minReadySeconds: 5                # Seconds a new Pod must be ready before it's considered available

  # ─── Progress Deadline ────────────────────────────────────────
  progressDeadlineSeconds: 600      # Fail the deployment if it hasn't progressed in this many seconds

  # ─── Pod Template ─────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: my-app               # Must match spec.selector.matchLabels
        version: "2.1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"

    spec:
      # ─── Scheduling ───────────────────────────────────────────
      affinity:
        podAntiAffinity:           # Spread replicas across nodes for HA
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: my-app

      topologySpreadConstraints:   # Fine-grained zone/node spreading (k8s 1.19+)
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app

      # ─── Security ─────────────────────────────────────────────
      serviceAccountName: my-app-sa
      automountServiceAccountToken: false

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000

      # ─── Containers ───────────────────────────────────────────
      containers:
        - name: app
          image: my-app:2.1.0
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090

          env:
            - name: APP_ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: db_host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password

          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /started
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: config
              mountPath: /etc/app
              readOnly: true
            - name: tmp
              mountPath: /tmp

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]   # Allow connections to drain

      # ─── Termination ──────────────────────────────────────────
      terminationGracePeriodSeconds: 60

      # ─── Volumes ──────────────────────────────────────────────
      volumes:
        - name: config
          configMap:
            name: app-config
        - name: tmp
          emptyDir: {}

      # ─── Image Pull ───────────────────────────────────────────
      imagePullSecrets:
        - name: registry-secret
```

---

## Update Strategies

### RollingUpdate (default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # Always have at least 3 running during update
    maxUnavailable: 0     # Zero-downtime deployment
```
Rolls out new Pods gradually while taking down old ones.

### Recreate
```yaml
strategy:
  type: Recreate
```
Terminates **all** old Pods before creating new ones. Causes downtime — use for apps that can't run multiple versions simultaneously.

---

## Essential kubectl Commands

```bash
# Create or update
kubectl apply -f deployment.yaml

# Check rollout status
kubectl rollout status deployment/my-app

# View rollout history
kubectl rollout history deployment/my-app

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to a specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# Scale replicas
kubectl scale deployment/my-app --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/my-app app=my-app:2.2.0

# Pause a rollout
kubectl rollout pause deployment/my-app

# Resume a rollout
kubectl rollout resume deployment/my-app

# Force restart all pods (zero-downtime)
kubectl rollout restart deployment/my-app
```

---

## Common Patterns

### Blue/Green Deployment
```yaml
# Blue (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      slot: blue
  template:
    metadata:
      labels:
        app: my-app
        slot: blue
    spec:
      containers:
        - name: app
          image: my-app:1.0.0
---
# Switch traffic by updating Service selector from slot: blue → slot: green
```

### Canary Deployment
```yaml
# Stable: 9 replicas of v1
# Canary: 1 replica of v2 — 10% of traffic goes to canary
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app          # Same label as stable — Service selects both
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
        - name: app
          image: my-app:2.0.0
```

---

## ⚠️ Gotchas

- `selector` is **immutable** — you cannot change it after creation. Delete and recreate.
- `maxUnavailable: 0` + `maxSurge: 0` is **invalid** (would make rollout impossible).
- `replicas: 0` is valid and useful — suspends the Deployment without deleting it.
- The Deployment only manages Pods matching its `selector` — pods with extra labels are fine.
- Set `minReadySeconds > 0` if your readiness probe takes time to stabilize.
- `revisionHistoryLimit: 0` disables rollback — don't do this unless storage is a concern.

---

*← [Pod](./01-pod.md) | [Back to README](../README.md) | Next: [ReplicaSet](./03-replicaset.md) →*
