# Example: Background Worker

> A production-ready background worker that processes jobs from a queue — with autoscaling, proper RBAC, graceful shutdown, and monitoring.

---

## Architecture

```
Queue (Redis/SQS/RabbitMQ)
        │
        ▼
  Deployment (workers)      ← KEDA ScaledObject or HPA scales based on queue depth
        │
   ┌────┼────┐
   │    │    │
  W0   W1   W2              ← Each worker Pod pulls jobs from the queue
        │
  ConfigMap + Secret        ← Queue URL, credentials, worker config
        │
  ServiceAccount + RBAC     ← Minimal permissions (read config only)
        │
  PVC (optional)            ← Scratch space for large job artifacts
```

---

## 1. Namespace & Quota

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: workers
  labels:
    kubernetes.io/metadata.name: workers
    environment: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: workers-quota
  namespace: workers
spec:
  hard:
    requests.cpu: "16"
    requests.memory: "32Gi"
    limits.cpu: "32"
    limits.memory: "64Gi"
    count/pods: "50"
```

---

## 2. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: worker-config
  namespace: workers
data:
  QUEUE_NAME: "job-queue"
  QUEUE_HOST: "redis.infrastructure.svc.cluster.local"
  QUEUE_PORT: "6379"
  WORKER_CONCURRENCY: "4"       # Jobs processed concurrently per Pod
  WORKER_MAX_RETRIES: "3"
  WORKER_TIMEOUT_SECONDS: "300" # 5 min max per job
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  METRICS_PORT: "9090"
  # Feature flags
  ENABLE_TRACING: "true"
  JAEGER_HOST: "jaeger.monitoring.svc.cluster.local"
```

---

## 3. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: worker-secret
  namespace: workers
type: Opaque
stringData:
  QUEUE_PASSWORD: "change-me"
  DATABASE_URL: "postgresql://worker:password@postgres.production.svc.cluster.local:5432/jobs"
  S3_ACCESS_KEY: "change-me"
  S3_SECRET_KEY: "change-me"
```

---

## 4. ServiceAccount & RBAC

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: worker-sa
  namespace: workers
  annotations:
    # AWS IRSA — lets worker assume an IAM role for S3 access
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/worker-role
automountServiceAccountToken: false

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: worker-role
  namespace: workers
rules:
  # Read config and secrets
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["worker-secret"]
  # Allow workers to write their own status (optional)
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: worker-binding
  namespace: workers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: worker-role
subjects:
  - kind: ServiceAccount
    name: worker-sa
    namespace: workers
```

---

## 5. Worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: job-worker
  namespace: workers
  labels:
    app: job-worker
    component: worker
  annotations:
    kubernetes.io/change-cause: "v1.0.0 — initial deployment"

spec:
  replicas: 2                     # Minimum — HPA will scale up from here
  selector:
    matchLabels:
      app: job-worker

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0           # Never take a worker offline during update

  template:
    metadata:
      labels:
        app: job-worker
        component: worker
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"

    spec:
      serviceAccountName: worker-sa
      automountServiceAccountToken: false

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 2000

      # Spread workers across nodes to avoid single point of failure
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: job-worker

      containers:
        - name: worker
          image: myapp/worker:1.0.0
          imagePullPolicy: IfNotPresent

          # Workers typically don't expose ports, but metrics and health are useful
          ports:
            - name: metrics
              containerPort: 9090
            - name: health
              containerPort: 8081

          # Load all config as environment variables
          envFrom:
            - configMapRef:
                name: worker-config
            - secretRef:
                name: worker-secret

          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Worker ID derived from Pod name (helps with distributed tracing)
            - name: WORKER_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name

          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"

          # Liveness: is the worker process running?
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3

          # Readiness: is the worker ready to accept jobs?
          readinessProbe:
            httpGet:
              path: /ready
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 15

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: scratch
              mountPath: /var/worker/scratch

          # CRITICAL: Graceful shutdown
          # 1. Stop accepting new jobs (preStop hook)
          # 2. Finish current job (terminationGracePeriodSeconds)
          lifecycle:
            preStop:
              exec:
                # Signal the worker to stop accepting new jobs
                command: ["/bin/sh", "-c", "kill -SIGTERM 1"]

      # Allow time to finish current jobs — must be > longest possible job duration
      terminationGracePeriodSeconds: 360   # 6 minutes

      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: "64Mi"
        - name: scratch
          emptyDir:
            sizeLimit: "2Gi"
```

---

## 6. HPA (CPU/Memory based)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: job-worker-hpa
  namespace: workers
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: job-worker
  minReplicas: 2
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30         # Scale up quickly when queue grows
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 600        # Scale down slowly (avoid thrashing)
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

---

## 7. KEDA ScaledObject (Queue-based Autoscaling)

> KEDA is a more powerful option — scales based on queue depth directly (Redis, SQS, RabbitMQ, etc.).

```yaml
# Requires KEDA to be installed in the cluster
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: job-worker-scaledobject
  namespace: workers
spec:
  scaleTargetRef:
    name: job-worker
  minReplicaCount: 0             # Scale to ZERO when queue is empty!
  maxReplicaCount: 30
  cooldownPeriod: 120            # Seconds to wait before scaling to 0
  pollingInterval: 15            # Check queue every 15 seconds
  triggers:
    # Redis list length trigger
    - type: redis
      metadata:
        address: redis.infrastructure.svc.cluster.local:6379
        listName: job-queue
        listLength: "10"         # 1 worker per 10 queued jobs
      authenticationRef:
        name: redis-trigger-auth

---
# KEDA TriggerAuthentication for Redis
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: redis-trigger-auth
  namespace: workers
spec:
  secretTargetRef:
    - parameter: password
      name: worker-secret
      key: QUEUE_PASSWORD
```

---

## 8. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: job-worker-pdb
  namespace: workers
spec:
  minAvailable: 1               # Always keep at least 1 worker running
  selector:
    matchLabels:
      app: job-worker
```

---

## 9. NetworkPolicy (restrict worker outbound)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: worker-egress
  namespace: workers
spec:
  podSelector:
    matchLabels:
      app: job-worker
  policyTypes: [Egress]
  egress:
    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53

    # Redis queue
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: infrastructure
          podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # PostgreSQL
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # External HTTPS (S3, external APIs)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]
      ports:
        - protocol: TCP
          port: 443
```

---

## Deploy & Verify

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml
kubectl apply -f pdb.yaml
kubectl apply -f networkpolicy.yaml

# Watch workers start
kubectl get pods -n workers -w

# Check worker logs
kubectl logs -n workers -l app=job-worker -f

# Simulate load (scale to test HPA)
kubectl scale deployment job-worker -n workers --replicas=10

# Check HPA status
kubectl describe hpa job-worker-hpa -n workers
```

---

*← [Stateful DB Example](./stateful-db.md) | [Back to README](../README.md) | Next: [CronJob Pipeline](./cronjob-pipeline.md) →*
