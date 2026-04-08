# CronJob

> Creates Jobs on a repeating schedule, using standard Unix cron syntax. Useful for periodic tasks like backups, reports, cleanup, and maintenance.

**apiVersion:** `batch/v1`  
**Kind:** `CronJob`

---

## Minimal Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"          # Every day at 2:00 AM
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:1.0
              command: ["/backup.sh"]
```

---

## Full Annotated Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
  labels:
    app: db-backup

spec:
  # ─── Schedule ─────────────────────────────────────────────────
  # Standard cron syntax: minute hour day-of-month month day-of-week
  # Timezone: UTC by default (set timeZone below to change)
  schedule: "0 3 * * *"          # Every day at 3:00 AM UTC

  timeZone: "America/New_York"   # k8s 1.27+ — schedule in local timezone
  # timeZone: "Asia/Cairo"
  # timeZone: "UTC"

  # ─── Concurrency ──────────────────────────────────────────────
  # What to do if the previous job is still running when the next fires
  concurrencyPolicy: Forbid       # Allow | Forbid | Replace
  # Allow:   Run multiple jobs concurrently
  # Forbid:  Skip if previous is still running
  # Replace: Cancel the running job and start a new one

  # ─── History ──────────────────────────────────────────────────
  successfulJobsHistoryLimit: 3   # Keep last N successful jobs (default: 3)
  failedJobsHistoryLimit: 1       # Keep last N failed jobs (default: 1)

  # ─── Starting Deadline ────────────────────────────────────────
  startingDeadlineSeconds: 300    # If job can't start within 5 min of schedule, skip it
                                  # Useful for clusters that were down during scheduled time

  # ─── Suspension ───────────────────────────────────────────────
  suspend: false                  # true = pause all future job runs without deleting

  # ─── Job Template ─────────────────────────────────────────────
  jobTemplate:
    metadata:
      labels:
        app: db-backup
        triggered-by: cronjob

    spec:
      # All Job fields are available here
      backoffLimit: 2
      activeDeadlineSeconds: 1800    # Kill if backup takes > 30 min
      ttlSecondsAfterFinished: 86400 # Clean up jobs after 24h

      template:
        metadata:
          labels:
            app: db-backup

        spec:
          restartPolicy: OnFailure   # Required

          serviceAccountName: backup-sa

          containers:
            - name: backup
              image: postgres:15
              imagePullPolicy: IfNotPresent

              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d-%H%M%S)
                  pg_dump -h $DB_HOST -U $DB_USER $DB_NAME \
                    | gzip > /backup/db-${DATE}.sql.gz
                  echo "Backup completed: db-${DATE}.sql.gz"

              env:
                - name: DB_HOST
                  valueFrom:
                    configMapKeyRef:
                      name: backup-config
                      key: db_host
                - name: DB_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: backup-config
                      key: db_name
                - name: DB_USER
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: username
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: password

              resources:
                requests:
                  cpu: "200m"
                  memory: "256Mi"
                limits:
                  cpu: "1"
                  memory: "1Gi"

              volumeMounts:
                - name: backup-storage
                  mountPath: /backup

          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
```

---

## Cron Schedule Reference

```
# ┌────────── minute       (0–59)
# │ ┌──────── hour         (0–23)
# │ │ ┌────── day of month (1–31)
# │ │ │ ┌──── month        (1–12)
# │ │ │ │ ┌── day of week  (0–6, Sunday=0)
# │ │ │ │ │
# * * * * *
```

| Schedule | Expression |
|----------|-----------|
| Every minute | `* * * * *` |
| Every 5 minutes | `*/5 * * * *` |
| Every hour | `0 * * * *` |
| Every day at midnight | `0 0 * * *` |
| Every day at 3 AM | `0 3 * * *` |
| Every Monday at 9 AM | `0 9 * * 1` |
| Every weekday at 6 PM | `0 18 * * 1-5` |
| First of every month | `0 0 1 * *` |
| Every 15 minutes | `*/15 * * * *` |
| Twice a day (noon & midnight) | `0 0,12 * * *` |

> 💡 Use [crontab.guru](https://crontab.guru) to validate expressions.

---

## Concurrency Policies Compared

```
Schedule: "*/5 * * * *"  (every 5 min)
Job duration: 8 minutes

Allow:
  T=0  → Job A starts
  T=5  → Job B starts (A still running)
  T=8  → Job A completes
  T=10 → Job C starts (B still running)

Forbid:
  T=0  → Job A starts
  T=5  → SKIPPED (A still running)
  T=10 → SKIPPED (A still running)
  T=8  → Job A completes
  T=15 → Job B starts

Replace:
  T=0  → Job A starts
  T=5  → Job A KILLED, Job B starts
  T=10 → Job B KILLED, Job C starts
```

---

## kubectl Commands

```bash
# Manually trigger a job from a CronJob
kubectl create job --from=cronjob/db-backup manual-backup-$(date +%s)

# List all jobs from a CronJob
kubectl get jobs --selector=batch.kubernetes.io/controller-uid=<uid>

# Suspend a CronJob
kubectl patch cronjob db-backup -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob db-backup -p '{"spec":{"suspend":false}}'

# Change schedule
kubectl patch cronjob db-backup -p '{"spec":{"schedule":"0 4 * * *"}}'

# View last job's logs
kubectl logs -l app=db-backup --tail=50
```

---

## ⚠️ Gotchas

- **Timezone is UTC by default** — always double-check when scheduling time-sensitive jobs.
- `startingDeadlineSeconds` is important for clusters with intermittent availability — without it, missed jobs catch up immediately on restart.
- Setting `successfulJobsHistoryLimit: 0` and `failedJobsHistoryLimit: 0` means you can't see historical runs — keep at least 1.
- The CronJob controller runs in the API server — very large numbers of jobs can cause load.
- With `Forbid`, if the cluster is down for a long time, many scheduled runs are skipped — one run starts when the cluster recovers.

---

*← [Job](./06-job.md) | [Back to README](../README.md) | Next: [Service](../networking/01-service.md) →*
