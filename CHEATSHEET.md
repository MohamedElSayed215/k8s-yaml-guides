# ☸️ Kubernetes YAML Quick Reference Cheatsheet

> One-page cheatsheet — copy the skeleton you need and fill it in.

---

## Every YAML file starts with this

```yaml
apiVersion: <group>/<version>
kind: <Kind>
metadata:
  name: my-object
  namespace: default        # omit for cluster-scoped resources
  labels:
    app: my-app
spec:
  ...
```

---

## apiVersion Quick Lookup

```
Pod, Service, ConfigMap, Secret, ServiceAccount, Namespace,
PersistentVolume, PersistentVolumeClaim, ResourceQuota, LimitRange  → v1

Deployment, ReplicaSet, StatefulSet, DaemonSet                       → apps/v1
Job, CronJob                                                         → batch/v1
Ingress, NetworkPolicy, IngressClass                                 → networking.k8s.io/v1
Role, ClusterRole, RoleBinding, ClusterRoleBinding                   → rbac.authorization.k8s.io/v1
HorizontalPodAutoscaler                                              → autoscaling/v2
PodDisruptionBudget                                                  → policy/v1
StorageClass, VolumeAttachment                                       → storage.k8s.io/v1
CustomResourceDefinition                                             → apiextensions.k8s.io/v1
PriorityClass                                                        → scheduling.k8s.io/v1
```

---

## Workloads

### Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests: { cpu: "100m", memory: "128Mi" }
        limits:   { cpu: "500m", memory: "256Mi" }
```

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: my-app }
  template:
    metadata:
      labels: { app: my-app }
    spec:
      containers:
        - name: app
          image: my-app:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: "100m", memory: "256Mi" }
            limits:   { cpu: "500m", memory: "512Mi" }
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 15
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            initialDelaySeconds: 5
```

### StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless    # Required: headless service name
  replicas: 3
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
        - name: postgres
          image: postgres:15
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
```

### Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: OnFailure      # Required: Never or OnFailure
      containers:
        - name: job
          image: my-job:1.0.0
          command: ["python", "run.py"]
```

### CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cron
spec:
  schedule: "0 2 * * *"            # Daily at 2 AM
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: job
              image: my-job:1.0.0
```

---

## Networking

### Service (ClusterIP)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector: { app: my-app }
  ports:
    - port: 80
      targetPort: 8080
```

### Service (LoadBalancer)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  type: LoadBalancer
  selector: { app: my-app }
  ports:
    - port: 80
      targetPort: 8080
```

### Headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector: { app: postgres }
  ports:
    - port: 5432
```

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [myapp.example.com]
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## Config

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  config.yaml: |
    server:
      port: 8080
```

### Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:                       # Auto base64-encoded
  DB_PASSWORD: "my-password"
  API_KEY: "my-api-key"
```

---

## Storage

### PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

---

## RBAC

### ServiceAccount + Role + RoleBinding
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-app-role
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: default
```

---

## Autoscaling & Reliability

### HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### PodDisruptionBudget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: my-app }
```

---

## Common Container Fields

```yaml
containers:
  - name: app
    image: my-app:1.0.0
    imagePullPolicy: IfNotPresent

    ports:
      - name: http
        containerPort: 8080

    env:
      - name: ENV_VAR
        value: "literal-value"
      - name: FROM_CONFIGMAP
        valueFrom:
          configMapKeyRef: { name: my-config, key: my-key }
      - name: FROM_SECRET
        valueFrom:
          secretKeyRef: { name: my-secret, key: my-key }
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name

    envFrom:
      - configMapRef: { name: my-config }
      - secretRef: { name: my-secret }

    resources:
      requests: { cpu: "100m", memory: "128Mi" }
      limits:   { cpu: "500m", memory: "512Mi" }

    livenessProbe:
      httpGet: { path: /healthz, port: 8080 }
      initialDelaySeconds: 15
      periodSeconds: 20

    readinessProbe:
      httpGet: { path: /ready, port: 8080 }
      initialDelaySeconds: 5
      periodSeconds: 10

    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]

    volumeMounts:
      - name: config
        mountPath: /etc/app
        readOnly: true
      - name: tmp
        mountPath: /tmp

volumes:
  - name: config
    configMap:
      name: my-config
  - name: secret-vol
    secret:
      secretName: my-secret
      defaultMode: 0400
  - name: pvc-vol
    persistentVolumeClaim:
      claimName: my-pvc
  - name: tmp
    emptyDir: {}
```

---

## Essential kubectl Commands

```bash
# Apply / Delete
kubectl apply -f file.yaml
kubectl delete -f file.yaml

# Dry run (validate without creating)
kubectl apply -f file.yaml --dry-run=client

# Diff (preview changes)
kubectl diff -f file.yaml

# Rollout
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app
kubectl rollout restart deployment/my-app

# Scale
kubectl scale deployment/my-app --replicas=5

# Logs
kubectl logs -l app=my-app --tail=100 -f

# Exec into a pod
kubectl exec -it <pod-name> -- /bin/sh

# Port forward
kubectl port-forward svc/my-app 8080:80

# Describe (events + status)
kubectl describe pod <pod-name>

# Field docs
kubectl explain deployment.spec.strategy
```

---

*← [Back to README](./README.md)*
