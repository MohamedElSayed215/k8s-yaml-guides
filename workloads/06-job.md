# Job

> Creates one or more Pods and ensures a specified number of them successfully complete. Unlike Deployments, Jobs run to completion and stop.

**apiVersion:** `batch/v1`  
**Kind:** `Job`

---

## When to Use

- Database migrations
- Batch data processing
- Report generation
- One-time cluster setup tasks
- Any task that should run once and finish

---

## Minimal Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      restartPolicy: OnFailure     # Required: OnFailure or Never
      containers:
        - name: migrate
          image: my-app:1.0.0
          command: ["python", "manage.py", "migrate"]
```

---

## Full Annotated Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
  namespace: production
  labels:
    app: data-processor
    job-type: batch

spec:
  # ─── Completion Behavior ──────────────────────────────────────
  completions: 5                  # Total successful completions needed (default: 1)
  parallelism: 2                  # Max Pods running simultaneously (default: 1)
  # Patterns:
  # Single job:         completions: 1, parallelism: 1
  # Fixed task queue:   completions: N, parallelism: M (runs N tasks, M at a time)
  # Work queue:         completions: unset, parallelism: M (workers exit when queue empty)

  completionMode: NonIndexed      # NonIndexed | Indexed
  # NonIndexed: any Pod completing counts (default)
  # Indexed: each Pod gets a unique index (JOB_COMPLETION_INDEX env var), useful for sharding

  # ─── Failure Handling ─────────────────────────────────────────
  backoffLimit: 4                 # Retry on failure up to N times (default: 6)
  activeDeadlineSeconds: 3600     # Kill the whole job if it runs longer than 1 hour

  # Per-Pod failure policy (k8s 1.26+)
  podFailurePolicy:
    rules:
      - action: FailJob           # Fail immediately (don't retry) on exit code 42
        onExitCodes:
          containerName: main
          operator: In
          values: [42]
      - action: Ignore            # Treat node preemption as non-failure
        onPodConditions:
          - type: DisruptionTarget

  # ─── TTL After Completion ─────────────────────────────────────
  ttlSecondsAfterFinished: 86400  # Delete Job (and its Pods) 24h after completion
                                  # 0 = delete immediately, unset = keep forever

  # ─── Pod Template ─────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: data-processor

    spec:
      # ─── REQUIRED for Jobs ────────────────────────────────────
      restartPolicy: OnFailure    # Never | OnFailure (Never = pod counts as retry)

      serviceAccountName: batch-sa

      # ─── Containers ───────────────────────────────────────────
      containers:
        - name: processor
          image: data-processor:2.0.0
          imagePullPolicy: IfNotPresent

          command: ["python", "process.py"]
          args:
            - "--input=/data/input"
            - "--output=/data/output"
            - "--batch-size=100"

          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
            - name: JOB_COMPLETION_INDEX   # Available when completionMode: Indexed
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']

          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"

          volumeMounts:
            - name: input-data
              mountPath: /data/input
              readOnly: true
            - name: output-data
              mountPath: /data/output
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: input-data
          persistentVolumeClaim:
            claimName: input-pvc
            readOnly: true
        - name: output-data
          persistentVolumeClaim:
            claimName: output-pvc
        - name: tmp
          emptyDir:
            sizeLimit: "500Mi"

      restartPolicy: OnFailure
      terminationGracePeriodSeconds: 30
```

---

## Completion Patterns

### 1. Single Job (default)
```yaml
spec:
  completions: 1
  parallelism: 1
```
One Pod runs, one success needed.

### 2. Fixed Completion Count
```yaml
spec:
  completions: 10     # Need 10 total successes
  parallelism: 3      # Run 3 at a time
```
Useful for processing N items where each Pod handles one item.

### 3. Work Queue
```yaml
spec:
  # completions: omitted
  parallelism: 4      # 4 workers drain from a queue
```
Workers read from a queue (Redis, SQS, etc.) and exit when empty.

### 4. Indexed Job
```yaml
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed    # Pods 0..4, each with unique JOB_COMPLETION_INDEX
```
Useful for sharded batch processing.

---

## restartPolicy Behavior

| Policy | Container fails | Pod-level |
|--------|----------------|-----------|
| `Never` | Start a **new Pod** (counts toward backoffLimit) | Never restart |
| `OnFailure` | **Restart container** in the same Pod | Restart in place |

---

## kubectl Commands

```bash
# Run a quick job
kubectl create job test --image=busybox -- echo hello

# Watch job progress
kubectl get jobs -w

# View Pod logs from a job
kubectl logs job/data-processor

# Delete job and its pods
kubectl delete job data-processor

# Delete job but keep pods (for debugging)
kubectl delete job data-processor --cascade=orphan
```

---

## ⚠️ Gotchas

- `restartPolicy: Always` is **invalid** for Jobs — use `Never` or `OnFailure`.
- `backoffLimit: 0` means "never retry" — one failure = job failed.
- Without `ttlSecondsAfterFinished`, completed Jobs (and their Pods) remain forever — clean them up or set TTL.
- `activeDeadlineSeconds` kills the entire Job, including running Pods — use it as a safety net.
- With `restartPolicy: Never`, each failed Pod creates a NEW Pod — you can end up with many failed pods.

---

*← [DaemonSet](./05-daemonset.md) | [Back to README](../README.md) | Next: [CronJob](./07-cronjob.md) →*
