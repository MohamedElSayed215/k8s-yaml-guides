# Example: Full Web Application

> A complete, production-ready example combining Deployment, Service, Ingress, ConfigMap, Secret, HPA, and PDB.

---

## Architecture

```
Internet
    │
    ▼
Ingress (nginx)
    │   myapp.example.com
    ▼
Service (ClusterIP) ← HPA (2-20 replicas)
    │
    ▼
Deployment (frontend + sidecar)
    │           │
    ▼           ▼
ConfigMap    Secret
    │
    ▼
Service (backend)
    │
    ▼
Deployment (backend API)
    │
    ▼
Service (postgres, headless)
    │
    ▼
StatefulSet (postgres)
    │
    ▼
PVC ← StorageClass (fast-ssd)
```

---

## 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    environment: production
    kubernetes.io/metadata.name: my-app
```

---

## 2. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: my-app
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "postgres.my-app.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "myapp"
  REDIS_HOST: "redis.my-app.svc.cluster.local"
  ALLOWED_ORIGINS: "https://myapp.example.com"
```

---

## 3. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: my-app
type: Opaque
stringData:
  DB_PASSWORD: "change-me-in-production"
  JWT_SECRET: "change-me-in-production"
  API_KEY: "change-me-in-production"
```

---

## 4. ServiceAccount + RBAC

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: my-app
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["app-secret"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-role-binding
  namespace: my-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: backend-role
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: my-app
```

---

## 5. Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: my-app
  labels:
    app: backend
  annotations:
    kubernetes.io/change-cause: "Initial deployment"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: backend-sa
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: backend
      containers:
        - name: backend
          image: myapp/backend:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: JWT_SECRET
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
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
            - name: tmp
              mountPath: /tmp
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 60
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: "100Mi"
```

---

## 6. Backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: my-app
  labels:
    app: backend
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9090
      targetPort: 9090
```

---

## 7. HPA for Backend

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

---

## 8. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
  namespace: my-app
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: backend
```

---

## 9. Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

---

## 10. ResourceQuota & LimitRange

```yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-app-quota
  namespace: my-app
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    count/pods: "50"
    count/services: "20"
    requests.storage: "200Gi"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: my-app-limits
  namespace: my-app
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "4"
        memory: "4Gi"
```

---

## Deploy Everything

```bash
# Apply all manifests in order
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl apply -f pdb.yaml
kubectl apply -f ingress.yaml
kubectl apply -f quota.yaml

# Or apply a whole directory
kubectl apply -f ./manifests/

# Watch rollout
kubectl rollout status deployment/backend -n my-app

# Verify everything is running
kubectl get all -n my-app
```

---

*← [Back to README](../README.md) | Next: [Stateful Database Example](./stateful-db.md) →*
