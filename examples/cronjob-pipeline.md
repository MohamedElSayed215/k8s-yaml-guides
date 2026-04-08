# Example: CronJob Pipeline

> A scheduled data pipeline — nightly ETL job that extracts data, transforms it, and loads it into a warehouse. Shows chaining jobs, shared volumes, and cleanup.

---

## Architecture

```
CronJob (nightly-etl, 1 AM UTC)
        │
        ▼
     Job Pod
        │
  initContainers (sequential):
  ┌─────────────────────────────┐
  │  1. extract   → /data/raw   │
  │  2. transform → /data/out   │
  └─────────────────────────────┘
        │
  mainContainer:
  ┌─────────────────────────────┐
  │  3. load → Data Warehouse   │
  └─────────────────────────────┘
        │
  Shared emptyDir volume (/data)
```

---

## 1. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: etl-config
  namespace: data
data:
  # Source database
  SOURCE_DB_HOST: "postgres.production.svc.cluster.local"
  SOURCE_DB_PORT: "5432"
  SOURCE_DB_NAME: "myapp"

  # Data warehouse
  WAREHOUSE_HOST: "warehouse.data.svc.cluster.local"
  WAREHOUSE_DB: "analytics"
  WAREHOUSE_SCHEMA: "public"

  # Pipeline settings
  BATCH_SIZE: "10000"
  EXTRACT_DAYS: "1"              # Extract last N days of data
  PARALLEL_WORKERS: "4"

  # S3 staging bucket
  S3_BUCKET: "my-etl-staging"
  S3_PREFIX: "nightly"

  # Notification
  SLACK_CHANNEL: "#data-alerts"
```

---

## 2. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: etl-secret
  namespace: data
type: Opaque
stringData:
  SOURCE_DB_PASSWORD: "change-me"
  WAREHOUSE_PASSWORD: "change-me"
  AWS_ACCESS_KEY_ID: "change-me"
  AWS_SECRET_ACCESS_KEY: "change-me"
  SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/..."
```

---

## 3. ServiceAccount

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: etl-sa
  namespace: data
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/etl-s3-role
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: etl-role
  namespace: data
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["etl-secret"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: etl-binding
  namespace: data
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: etl-role
subjects:
  - kind: ServiceAccount
    name: etl-sa
    namespace: data
```

---

## 4. PVC for staging data (persistent across jobs)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: etl-staging-pvc
  namespace: data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
```

---

## 5. The CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-etl
  namespace: data
  labels:
    app: nightly-etl
    pipeline: etl

spec:
  schedule: "0 1 * * *"         # Every day at 1:00 AM UTC
  timeZone: "UTC"
  concurrencyPolicy: Forbid     # Never run two ETL jobs at the same time
  successfulJobsHistoryLimit: 7 # Keep a week of history
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 600  # Skip if cluster was down > 10 min at schedule time
  suspend: false

  jobTemplate:
    metadata:
      labels:
        app: nightly-etl

    spec:
      backoffLimit: 1              # Only retry once on failure
      activeDeadlineSeconds: 7200 # Kill if ETL takes > 2 hours
      ttlSecondsAfterFinished: 604800  # Clean up after 7 days

      template:
        metadata:
          labels:
            app: nightly-etl

        spec:
          restartPolicy: OnFailure
          serviceAccountName: etl-sa
          automountServiceAccountToken: false

          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            fsGroup: 2000

          # ── Init containers run sequentially ─────────────────
          initContainers:

            # Step 1: Extract — pull data from source DB to shared volume
            - name: extract
              image: myapp/etl-extract:1.0.0
              imagePullPolicy: IfNotPresent
              command: ["/app/extract.sh"]
              args: ["--days=$(EXTRACT_DAYS)", "--output=/data/raw/", "--batch=$(BATCH_SIZE)"]
              envFrom:
                - configMapRef:
                    name: etl-config
              env:
                - name: SOURCE_DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: etl-secret
                      key: SOURCE_DB_PASSWORD
                - name: JOB_DATE
                  value: ""   # Will be set by the script using $(date)
              resources:
                requests:
                  cpu: "500m"
                  memory: "1Gi"
                limits:
                  cpu: "2"
                  memory: "4Gi"
              volumeMounts:
                - name: pipeline-data
                  mountPath: /data

            # Step 2: Transform — clean, deduplicate, reshape
            - name: transform
              image: myapp/etl-transform:1.0.0
              imagePullPolicy: IfNotPresent
              command: ["/app/transform.py"]
              args:
                - "--input=/data/raw/"
                - "--output=/data/transformed/"
                - "--workers=$(PARALLEL_WORKERS)"
              envFrom:
                - configMapRef:
                    name: etl-config
              env:
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: etl-secret
                      key: AWS_ACCESS_KEY_ID
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: etl-secret
                      key: AWS_SECRET_ACCESS_KEY
              resources:
                requests:
                  cpu: "1"
                  memory: "2Gi"
                limits:
                  cpu: "4"
                  memory: "8Gi"
              volumeMounts:
                - name: pipeline-data
                  mountPath: /data

          # ── Main container: Load ──────────────────────────────
          containers:
            - name: load
              image: myapp/etl-load:1.0.0
              imagePullPolicy: IfNotPresent
              command: ["/app/load.sh"]
              args:
                - "--input=/data/transformed/"
                - "--target=$(WAREHOUSE_DB).$(WAREHOUSE_SCHEMA)"
              envFrom:
                - configMapRef:
                    name: etl-config
              env:
                - name: WAREHOUSE_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: etl-secret
                      key: WAREHOUSE_PASSWORD
                - name: SLACK_WEBHOOK_URL
                  valueFrom:
                    secretKeyRef:
                      name: etl-secret
                      key: SLACK_WEBHOOK_URL

              resources:
                requests:
                  cpu: "500m"
                  memory: "1Gi"
                limits:
                  cpu: "2"
                  memory: "4Gi"

              volumeMounts:
                - name: pipeline-data
                  mountPath: /data
                  readOnly: true    # Load only reads — doesn't write back

          # ── Volumes ──────────────────────────────────────────
          volumes:
            - name: pipeline-data
              persistentVolumeClaim:
                claimName: etl-staging-pvc
                # Use emptyDir if you don't need persistence between jobs:
                # emptyDir:
                #   sizeLimit: "20Gi"

          terminationGracePeriodSeconds: 120
```

---

## 6. Cleanup Job (weekly, removes old staging files)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etl-cleanup
  namespace: data
spec:
  schedule: "0 4 * * 0"          # Sundays at 4 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: busybox:1.35
              command:
                - /bin/sh
                - -c
                - |
                  echo "Cleaning up staging files older than 7 days..."
                  find /data -type f -mtime +7 -name "*.parquet" -delete
                  find /data -type f -mtime +7 -name "*.csv" -delete
                  find /data -type d -empty -delete
                  echo "Cleanup complete. Remaining:"
                  du -sh /data/*
              resources:
                requests:
                  cpu: "50m"
                  memory: "64Mi"
                limits:
                  cpu: "200m"
                  memory: "128Mi"
              volumeMounts:
                - name: pipeline-data
                  mountPath: /data
          volumes:
            - name: pipeline-data
              persistentVolumeClaim:
                claimName: etl-staging-pvc
```

---

## 7. Manually Trigger a Run

```bash
# Trigger the ETL pipeline right now (useful for backfills)
kubectl create job --from=cronjob/nightly-etl manual-etl-$(date +%s) -n data

# Watch the job
kubectl get jobs -n data -w

# Follow logs from each stage
# Extract stage (init container)
kubectl logs -n data -l app=nightly-etl -c extract -f

# Transform stage (init container)
kubectl logs -n data -l app=nightly-etl -c transform -f

# Load stage (main container)
kubectl logs -n data -l app=nightly-etl -c load -f

# See job completion status
kubectl get jobs -n data
```

---

## ⚠️ Gotchas

- `concurrencyPolicy: Forbid` is essential — never run two ETLs simultaneously on the same data.
- `activeDeadlineSeconds` is your safety net — if the pipeline hangs, it gets killed.
- init containers **run to completion in order** — if `extract` fails, `transform` never runs.
- The shared `emptyDir`/PVC volume is available to ALL init containers AND the main container.
- If a Job Pod is killed mid-run (OOM, node failure), `restartPolicy: OnFailure` starts a new Pod — make your pipeline **idempotent** (safe to re-run).
- `ttlSecondsAfterFinished` cleans up the Job object but NOT the PVC — use the cleanup CronJob for that.

---

*← [Background Worker Example](./background-worker.md) | [Back to README](../README.md)*
