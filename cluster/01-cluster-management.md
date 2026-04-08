# Cluster Management: Namespace, ResourceQuota, LimitRange, HPA & PodDisruptionBudget

---

# Namespace

**apiVersion:** `v1` | **Kind:** `Namespace`

> Virtual clusters within a cluster. Provides isolation, scope, and the ability to apply policies per team/environment.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: backend
    # Required for NetworkPolicy namespaceSelector matching:
    kubernetes.io/metadata.name: production
  annotations:
    owner: "backend-team@example.com"
    cost-center: "engineering"
```

```bash
# Create
kubectl create namespace staging

# Set default namespace for context
kubectl config set-context --current --namespace=production

# Delete (deletes ALL resources in namespace)
kubectl delete namespace staging
```

---

# ResourceQuota

**apiVersion:** `v1` | **Kind:** `ResourceQuota`

> Limits total resource consumption within a namespace. Prevents one team from starving others.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production

spec:
  hard:
    # ─── Compute Resources ────────────────────────────────────
    requests.cpu: "10"            # Total CPU requests in namespace
    requests.memory: "20Gi"       # Total memory requests
    limits.cpu: "20"              # Total CPU limits
    limits.memory: "40Gi"         # Total memory limits

    # ─── Object Count ─────────────────────────────────────────
    count/pods: "50"              # Max Pods
    count/services: "20"          # Max Services
    count/persistentvolumeclaims: "10"
    count/secrets: "50"
    count/configmaps: "30"
    count/deployments.apps: "20"
    count/jobs.batch: "10"

    # ─── Storage ──────────────────────────────────────────────
    requests.storage: "100Gi"     # Total storage across all PVCs
    persistentvolumeclaims: "10"  # Max number of PVCs
    fast-ssd.storageclass.storage.k8s.io/requests.storage: "50Gi"  # Per StorageClass

    # ─── Priority ─────────────────────────────────────────────
    # Limit resources for specific priority classes
    # (Use with scopeSelector below)

  # ─── Scopes ───────────────────────────────────────────────────
  # Apply quota only to resources matching these scopes
  # scopes: [BestEffort, NotBestEffort, Terminating, NotTerminating]

  # Apply quota only to specific priority class
  # scopeSelector:
  #   matchExpressions:
  #     - operator: In
  #       scopeName: PriorityClass
  #       values: [high-priority]
```

```bash
# View quota usage
kubectl describe resourcequota -n production

# Output:
# Name:                production-quota
# Namespace:           production
# Resource            Used    Hard
# --------            ----    ----
# count/pods          12      50
# limits.cpu          2500m   20
# requests.cpu        1200m   10
```

---

# LimitRange

**apiVersion:** `v1` | **Kind:** `LimitRange`

> Sets default resource requests/limits and enforces min/max per container, Pod, or PVC in a namespace. Ensures no container runs without resource constraints.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production

spec:
  limits:
    # ─── Container Limits ─────────────────────────────────────
    - type: Container
      # Default values applied if not specified in the container
      default:
        cpu: "500m"
        memory: "256Mi"
      # Default request values
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      # Minimum allowed
      min:
        cpu: "50m"
        memory: "64Mi"
      # Maximum allowed
      max:
        cpu: "2"
        memory: "2Gi"
      # max/request ratio (prevents large limits with tiny requests)
      maxLimitRequestRatio:
        cpu: "4"                  # Limit can be at most 4× the request
        memory: "2"

    # ─── Pod Limits ───────────────────────────────────────────
    - type: Pod
      max:
        cpu: "4"
        memory: "8Gi"

    # ─── PVC Size Limits ──────────────────────────────────────
    - type: PersistentVolumeClaim
      min:
        storage: "1Gi"
      max:
        storage: "50Gi"
```

```bash
# View limit ranges
kubectl describe limitrange -n production
```

---

# HorizontalPodAutoscaler (HPA)

**apiVersion:** `autoscaling/v2` | **Kind:** `HorizontalPodAutoscaler`

> Automatically scales the number of Pod replicas based on CPU, memory, or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production

spec:
  # ─── Target ───────────────────────────────────────────────────
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment              # Deployment | StatefulSet | ReplicaSet
    name: my-app

  # ─── Scale Bounds ─────────────────────────────────────────────
  minReplicas: 2                  # Never scale below this
  maxReplicas: 20                 # Never scale above this

  # ─── Metrics ──────────────────────────────────────────────────
  metrics:
    # CPU utilization (most common)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # Target 70% CPU utilization

    # Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 500Mi      # Target 500Mi per Pod

    # Custom metric from Prometheus (via custom-metrics-apiserver)
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"     # 1000 RPS per Pod

    # External metric (queue depth, etc.)
    - type: External
      external:
        metric:
          name: sqs_queue_depth
          selector:
            matchLabels:
              queue: my-queue
        target:
          type: AverageValue
          averageValue: "10"

  # ─── Scaling Behavior ─────────────────────────────────────────
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up again
      policies:
        - type: Pods
          value: 4                       # Add at most 4 Pods per period
          periodSeconds: 60
        - type: Percent
          value: 100                     # Or 100% of current replicas
          periodSeconds: 60
      selectPolicy: Max                 # Use the policy that allows most change

    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1                       # Remove at most 1 Pod per period
          periodSeconds: 60
      selectPolicy: Min                 # Use the policy that allows least change
```

```bash
# View HPA status
kubectl get hpa -n production

# Describe (shows current metrics)
kubectl describe hpa my-app-hpa -n production

# Manually trigger scale (override HPA temporarily)
kubectl scale deployment my-app --replicas=10

# Watch HPA in action
kubectl get hpa my-app-hpa -w
```

> ⚠️ HPA requires **Metrics Server** to be installed in the cluster for CPU/memory metrics.

---

# PodDisruptionBudget (PDB)

**apiVersion:** `policy/v1` | **Kind:** `PodDisruptionBudget`

> Limits the number of Pods that can be voluntarily disrupted simultaneously (node drains, cluster upgrades, etc.). Ensures your app stays available during maintenance.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production

spec:
  # ─── Availability Guarantee ───────────────────────────────────
  # Choose ONE of:
  minAvailable: 2               # At least 2 Pods must always be running
  # maxUnavailable: 1           # At most 1 Pod can be unavailable at once
  # minAvailable: "80%"         # Percentage form also works
  # maxUnavailable: "20%"

  # ─── Target Pods ──────────────────────────────────────────────
  selector:
    matchLabels:
      app: my-app

  # ─── Unhealthy Pod Eviction Policy (k8s 1.26+) ───────────────
  unhealthyPodEvictionPolicy: IfHealthyBudget
  # IfHealthyBudget: Evict unhealthy pods only if budget allows (default)
  # AlwaysAllow:     Always evict unhealthy pods (faster drain)
```

### Common PDB Patterns

```yaml
# Stateless app: allow one Pod down at a time
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: frontend

---
# Critical service: require 3 Pods always available
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: payments

---
# Database primary (allow NO voluntary disruption)
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: postgres
      role: primary
```

```bash
# View PDB status (DISRUPTIONS ALLOWED)
kubectl get pdb -n production

# Describe
kubectl describe pdb my-app-pdb -n production
```

> ⚠️ PDB only applies to **voluntary** disruptions (node drain, cluster upgrades). Node failures, OOM kills, and hardware failures are not covered.

---

# PriorityClass

**apiVersion:** `scheduling.k8s.io/v1` | **Kind:** `PriorityClass`

> Assigns scheduling priority to Pods. Higher priority Pods are scheduled first and can preempt lower priority Pods when resources are scarce.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000                   # Higher = more priority (system-critical = 2000001000)
globalDefault: false             # Only one PriorityClass can be globalDefault: true
description: "For critical production services"
preemptionPolicy: PreemptLowerPriority  # PreemptLowerPriority | Never
```

Use in Pod/Deployment:
```yaml
spec:
  priorityClassName: high-priority
```

### Built-in Priority Classes

| Name | Value | Use |
|------|-------|-----|
| `system-cluster-critical` | 2000000000 | kube-apiserver, etcd |
| `system-node-critical` | 2000001000 | kube-proxy, CNI |
| (none) | 0 | Default for all user Pods |

---

*← [RBAC](../rbac/01-rbac.md) | [Back to README](../README.md) | Next: [Examples](../examples/web-app.md) →*
