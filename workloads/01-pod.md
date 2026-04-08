# Pod

> The smallest and simplest Kubernetes object. A Pod represents a single instance of a running process — one or more tightly-coupled containers sharing network and storage.

**apiVersion:** `v1`  
**Kind:** `Pod`

---

## Minimal Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
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
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  namespace: production
  labels:
    app: my-app           # Used by Services and selectors
    version: "1.0"
  annotations:
    prometheus.io/scrape: "true"   # Custom annotations for tooling
    prometheus.io/port: "8080"

spec:
  # ─── Scheduling ───────────────────────────────────────────────
  nodeName: node-1                # Pin to a specific node (usually avoid this)
  nodeSelector:                   # Schedule only on nodes with these labels
    kubernetes.io/os: linux
    disktype: ssd

  affinity:
    nodeAffinity:                 # Prefer or require specific node types
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [us-east-1a, us-east-1b]
    podAntiAffinity:              # Spread Pods across nodes for HA
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: my-app

  tolerations:                    # Allow scheduling on tainted nodes
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

  priorityClassName: high-priority  # From a PriorityClass object

  # ─── Service Account & Security ───────────────────────────────
  serviceAccountName: my-service-account
  automountServiceAccountToken: false  # Disable if Pod doesn't call the API

  securityContext:                # Pod-level security (applies to all containers)
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000                 # Volume ownership group
    seccompProfile:
      type: RuntimeDefault

  # ─── Init Containers ──────────────────────────────────────────
  # Run to completion before main containers start
  initContainers:
    - name: init-db
      image: busybox:1.35
      command: ['sh', '-c', 'until nc -z db 5432; do echo waiting; sleep 2; done']

  # ─── Main Containers ──────────────────────────────────────────
  containers:
    - name: app
      image: my-app:1.0.0
      imagePullPolicy: IfNotPresent   # Always | Never | IfNotPresent

      # Ports (informational — doesn't actually expose)
      ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090

      # Environment Variables
      env:
        - name: APP_ENV
          value: "production"
        - name: DB_PASSWORD           # From a Secret
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: APP_NAME              # From a ConfigMap
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app_name
        - name: POD_NAME              # From the Pod itself (Downward API)
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

      # Load all keys from a ConfigMap as env vars
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret

      # Resources — always set these!
      resources:
        requests:                   # Minimum guaranteed resources
          cpu: "100m"               # 100 millicores = 0.1 CPU
          memory: "128Mi"
        limits:                     # Maximum allowed
          cpu: "500m"
          memory: "512Mi"

      # Volume Mounts
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        - name: data-volume
          mountPath: /data
        - name: tmp
          mountPath: /tmp

      # Health Checks
      livenessProbe:               # Restarts container if this fails
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15    # Wait before first probe
        periodSeconds: 20
        failureThreshold: 3

      readinessProbe:              # Removes Pod from Service if this fails
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10

      startupProbe:                # Gives slow-starting containers more time
        httpGet:
          path: /started
          port: 8080
        failureThreshold: 30
        periodSeconds: 10          # Up to 300s (30×10) to start

      # Container-level security
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]  # Only if needed

      # Lifecycle hooks
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo started > /tmp/started"]
        preStop:                   # Graceful shutdown hook
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]

    # Sidecar container
    - name: log-shipper
      image: fluentd:v1.16
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app

  # ─── Volumes ──────────────────────────────────────────────────
  volumes:
    - name: config-volume           # From a ConfigMap
      configMap:
        name: app-config
        items:
          - key: app.properties
            path: app.properties

    - name: app-secret-volume       # From a Secret
      secret:
        secretName: app-secret
        defaultMode: 0400           # Read-only by owner

    - name: data-volume             # From a PersistentVolumeClaim
      persistentVolumeClaim:
        claimName: my-pvc

    - name: tmp                     # In-memory ephemeral storage
      emptyDir:
        medium: Memory
        sizeLimit: "64Mi"

    - name: log-volume              # Shared between containers
      emptyDir: {}

    - name: host-path               # Mount from the host node (use carefully)
      hostPath:
        path: /var/log
        type: DirectoryOrCreate

  # ─── DNS & Networking ─────────────────────────────────────────
  dnsPolicy: ClusterFirst          # ClusterFirst | Default | None
  dnsConfig:
    nameservers: ["1.1.1.1"]
    searches: ["my-namespace.svc.cluster.local"]

  hostNetwork: false               # true = share node's network namespace
  hostname: my-custom-hostname
  subdomain: my-service            # Required for DNS: hostname.subdomain.namespace.svc

  # ─── Restart & Termination ────────────────────────────────────
  restartPolicy: Always            # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30

  # ─── Image Pull ───────────────────────────────────────────────
  imagePullSecrets:
    - name: registry-credentials
```

---

## Key Concepts

### Probe Types
| Type | Command | HTTP | TCP |
|------|---------|------|-----|
| `exec` | `command: [...]` | — | — |
| `httpGet` | — | `path:`, `port:` | — |
| `tcpSocket` | — | — | `port:` |
| `grpc` | — | — | `port:` |

### Resource Units
| Resource | Unit Examples |
|----------|--------------|
| CPU | `100m` (0.1 core), `1`, `2.5` |
| Memory | `128Mi`, `1Gi`, `512M` |

### imagePullPolicy
| Value | Behavior |
|-------|----------|
| `Always` | Pull every time the Pod starts |
| `IfNotPresent` | Use local cache if available |
| `Never` | Only use local image — error if not found |

---

## Common Patterns

### Minimal production Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:1.0.0
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      securityContext:
        runAsNonRoot: true
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
```

### Wait for a dependency (init container pattern)
```yaml
initContainers:
  - name: wait-for-postgres
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
```

---

## ⚠️ Gotchas

- Pods are **ephemeral** — never use a bare Pod in production, use a Deployment.
- `resources.limits` without `resources.requests` defaults requests to = limits.
- `readOnlyRootFilesystem: true` requires explicit `emptyDir` volumes for writable paths like `/tmp`.
- `preStop` hook runs before `SIGTERM` — use it for graceful drain.
- `terminationGracePeriodSeconds` counts from SIGTERM, not from preStop start.

---

*← [Back to README](../README.md) | Next: [Deployment](./02-deployment.md) →*
