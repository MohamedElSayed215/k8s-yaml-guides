# Example: Stateful Database (PostgreSQL)

> A production-ready PostgreSQL StatefulSet with persistent storage, secrets, headless service, and a client-facing service.

---

## Architecture

```
Client Pods  →  postgres (ClusterIP Service, port 5432)
                    │
                    ▼
              StatefulSet (postgres-0, postgres-1, postgres-2)
              postgres-headless (Headless Service for DNS)
                    │
           ┌────────┼────────┐
           ▼        ▼        ▼
        PVC-0    PVC-1    PVC-2   ← Each gets its own 50Gi volume
       (fast-ssd StorageClass)
```

---

## 1. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: production
type: Opaque
stringData:
  POSTGRES_USER: "appuser"
  POSTGRES_PASSWORD: "change-me-in-production"
  POSTGRES_DB: "myapp"
  # Connection string for client apps
  DATABASE_URL: "postgresql://appuser:change-me-in-production@postgres.production.svc.cluster.local:5432/myapp"
```

---

## 2. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: production
data:
  # PostgreSQL tuning parameters
  postgresql.conf: |
    # Connections
    max_connections = 200

    # Memory
    shared_buffers = 256MB
    effective_cache_size = 768MB
    work_mem = 4MB
    maintenance_work_mem = 64MB

    # WAL
    wal_level = replica
    max_wal_senders = 3
    wal_keep_size = 128MB

    # Logging
    log_timezone = 'UTC'
    log_statement = 'ddl'
    log_min_duration_statement = 1000   # Log queries > 1s

    # Performance
    random_page_cost = 1.1              # SSD: set to 1.1 (default 4.0 is for HDD)
    effective_io_concurrency = 200      # SSD concurrent I/O

  pg_hba.conf: |
    # TYPE  DATABASE  USER      ADDRESS         METHOD
    local   all       all                       trust
    host    all       all       127.0.0.1/32    md5
    host    all       all       ::1/128         md5
    host    all       all       10.0.0.0/8      md5
```

---

## 3. StorageClass (if not already exists)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com       # Change for your cloud provider
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain               # IMPORTANT: Retain for databases!
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## 4. Headless Service (required by StatefulSet)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
  publishNotReadyAddresses: true    # Include pods during initialization
```

---

## 5. Client Service (for apps to connect)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
  sessionAffinity: ClientIP         # Sticky connections (reduces connection churn)
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

---

## 6. StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres

spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady   # Start postgres-0, then 1, then 2

  selector:
    matchLabels:
      app: postgres

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0               # Set to 2 for canary update (update only replica 2 first)

  template:
    metadata:
      labels:
        app: postgres

    spec:
      terminationGracePeriodSeconds: 60

      securityContext:
        fsGroup: 999
        runAsUser: 999
        runAsNonRoot: true

      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: kubernetes.io/hostname    # Never 2 replicas on same node
              labelSelector:
                matchLabels:
                  app: postgres

      # Fix data directory permissions before postgres starts
      initContainers:
        - name: init-permissions
          image: busybox:1.35
          command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data && chmod 700 /var/lib/postgresql/data"]
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          securityContext:
            runAsUser: 0          # root, only for init

      containers:
        - name: postgres
          image: postgres:15.3
          imagePullPolicy: IfNotPresent

          ports:
            - name: postgres
              containerPort: 5432

          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name

          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"

          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false   # postgres writes to data dir
            capabilities:
              drop: ["ALL"]
              add: ["CHOWN", "FOWNER", "SETUID", "SETGID"]

          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: postgres-config
              mountPath: /etc/postgresql/conf.d
              readOnly: true

      volumes:
        - name: postgres-config
          configMap:
            name: postgres-config

  # Each pod gets its own PVC: data-postgres-0, data-postgres-1, data-postgres-2
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: postgres
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

---

## 7. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  minAvailable: 2              # Always keep at least 2 replicas running
  selector:
    matchLabels:
      app: postgres
```

---

## 8. Backup CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 3 * * *"       # Daily at 3 AM
  timeZone: "UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 604800  # 7 days
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:15.3
              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d-%H%M%S)
                  BACKUP_FILE="/backup/postgres-${DATE}.sql.gz"
                  echo "Starting backup to ${BACKUP_FILE}"
                  pg_dump -h postgres.production.svc.cluster.local \
                          -U $POSTGRES_USER \
                          -d $POSTGRES_DB \
                          --no-owner \
                          --no-acl \
                    | gzip > $BACKUP_FILE
                  echo "Backup complete. Size: $(du -sh $BACKUP_FILE | cut -f1)"
                  # Cleanup backups older than 30 days
                  find /backup -name "*.sql.gz" -mtime +30 -delete
                  echo "Cleanup complete."
              env:
                - name: POSTGRES_USER
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: POSTGRES_USER
                - name: POSTGRES_DB
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: POSTGRES_DB
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: POSTGRES_PASSWORD
              resources:
                requests:
                  cpu: "100m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "1Gi"
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: postgres-backup-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup-pvc
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
```

---

## Connection DNS Reference

```bash
# From any Pod in the cluster:
postgresql://appuser:password@postgres.production.svc.cluster.local:5432/myapp

# Direct to a specific replica (via headless service):
postgresql://appuser:password@postgres-0.postgres-headless.production.svc.cluster.local:5432/myapp
postgresql://appuser:password@postgres-1.postgres-headless.production.svc.cluster.local:5432/myapp

# From a Pod in the same namespace — short form:
postgresql://appuser:password@postgres:5432/myapp
```

---

## Deploy & Verify

```bash
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-config.yaml
kubectl apply -f storageclass.yaml
kubectl apply -f postgres-headless-svc.yaml
kubectl apply -f postgres-svc.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f postgres-pdb.yaml
kubectl apply -f postgres-backup-cronjob.yaml

# Watch pods come up in order
kubectl get pods -n production -l app=postgres -w

# Check PVCs were created
kubectl get pvc -n production -l app=postgres

# Verify connectivity
kubectl run psql-test --image=postgres:15 -it --rm \
  --env="PGPASSWORD=change-me-in-production" \
  -- psql -h postgres.production.svc.cluster.local -U appuser -d myapp -c "\l"
```

---

*← [Web App Example](./web-app.md) | [Back to README](../README.md) | Next: [Background Worker Example](./background-worker.md) →*
