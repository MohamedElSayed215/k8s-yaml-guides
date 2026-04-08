# DaemonSet

> Ensures a copy of a Pod runs on **every node** (or a selected subset) in the cluster. Automatically adds Pods to new nodes and removes them from deleted nodes.

**apiVersion:** `apps/v1`  
**Kind:** `DaemonSet`

---

## Common Use Cases

- Log collectors (Fluentd, Filebeat, Logstash)
- Metrics agents (Prometheus node-exporter, Datadog agent)
- Network plugins (CNI, kube-proxy)
- Security agents (Falco, intrusion detection)
- Storage daemons (Ceph, GlusterFS)

---

## Minimal Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.6.1
          ports:
            - containerPort: 9100
```

---

## Full Annotated Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
    component: log-collector

spec:
  selector:
    matchLabels:
      app: fluentd                  # Must match template.metadata.labels

  # ─── Update Strategy ──────────────────────────────────────────
  updateStrategy:
    type: RollingUpdate             # RollingUpdate | OnDelete
    rollingUpdate:
      maxSurge: 0                   # DaemonSet: maxSurge OR maxUnavailable, not both
      maxUnavailable: 1             # Update 1 node at a time

  revisionHistoryLimit: 10

  # ─── Pod Template ─────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: fluentd
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "24231"

    spec:
      # ─── IMPORTANT: Tolerate control-plane nodes if needed ────
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master        # older clusters
          operator: Exists
          effect: NoSchedule
        # Run on nodes with any taint (catch-all):
        # - operator: Exists

      # ─── Node Selection ───────────────────────────────────────
      # Without nodeSelector, runs on ALL nodes
      # nodeSelector:
      #   role: log-node
      # Or use affinity for more complex rules:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values: [linux]

      # ─── Security ─────────────────────────────────────────────
      serviceAccountName: fluentd-sa

      # DaemonSets often need host access — be careful with privileges
      securityContext:
        runAsNonRoot: false         # log collectors often need root
        runAsUser: 0

      # ─── Scheduling Priority ──────────────────────────────────
      priorityClassName: system-node-critical  # Prevents eviction on resource pressure

      # ─── Containers ───────────────────────────────────────────
      containers:
        - name: fluentd
          image: fluentd:v1.16-debian-1
          imagePullPolicy: IfNotPresent

          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              valueFrom:
                configMapKeyRef:
                  name: fluentd-config
                  key: elasticsearch_host
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName      # Know which node this runs on

          resources:
            requests:
              cpu: "100m"
              memory: "200Mi"
            limits:
              cpu: "500m"
              memory: "500Mi"

          # Access host filesystem for logs
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluentd/etc/fluent.conf
              subPath: fluent.conf
              readOnly: true

          ports:
            - name: http-input
              containerPort: 9880
            - name: prometheus
              containerPort: 24231

          livenessProbe:
            httpGet:
              path: /fluentd.pod.healthcheck?json=%7B%22log%22%3A+%22health+check%22%7D
              port: 9880
            initialDelaySeconds: 5
            periodSeconds: 30

          securityContext:
            privileged: false
            capabilities:
              add: ["DAC_READ_SEARCH"]   # Read files owned by other users

      # ─── Host Volumes ─────────────────────────────────────────
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluentd-config

      # ─── Termination ──────────────────────────────────────────
      terminationGracePeriodSeconds: 30
```

---

## Run on Specific Nodes Only

### Using `nodeSelector`
```yaml
spec:
  template:
    spec:
      nodeSelector:
        role: gpu-node             # Only run on nodes labeled role=gpu-node
```

### Using node `affinity`
```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values: [amd64]
```

---

## Common Tolerations

```yaml
tolerations:
  # Run on control-plane nodes
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule

  # Run on nodes under memory pressure
  - key: node.kubernetes.io/memory-pressure
    operator: Exists
    effect: NoSchedule

  # Run on uninitialized nodes
  - key: node.cloudprovider.kubernetes.io/uninitialized
    operator: Exists
    effect: NoSchedule

  # Run on not-ready nodes (e.g., network agents)
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoSchedule
```

---

## Update Strategies

### RollingUpdate (default)
```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1       # Update one node at a time
```

### OnDelete
```yaml
updateStrategy:
  type: OnDelete            # Only updates a Pod when you manually delete it
                            # Gives full control over when each node updates
```

---

## kubectl Commands

```bash
# Check how many nodes have the DaemonSet running
kubectl get daemonset fluentd -n logging

# Describe shows DESIRED vs CURRENT vs READY vs UP-TO-DATE vs AVAILABLE
kubectl describe daemonset fluentd -n logging

# Check which nodes are running the Pod
kubectl get pods -n logging -l app=fluentd -o wide

# Trigger rolling restart
kubectl rollout restart daemonset/fluentd -n logging

# Rollout status
kubectl rollout status daemonset/fluentd -n logging
```

---

## ⚠️ Gotchas

- DaemonSet Pods are scheduled by the DaemonSet controller, **not the scheduler** — they bypass some scheduling rules.
- Adding `nodeSelector` or `affinity` restricts which nodes get the Pod — nodes without matching labels run nothing.
- Without explicit tolerations, DaemonSet Pods won't run on tainted nodes (including control-plane nodes).
- `hostPath` volumes are the same path on every node — make sure the path exists or use `type: DirectoryOrCreate`.
- Deleting a DaemonSet deletes all its Pods immediately (no graceful drain).

---

*← [StatefulSet](./04-statefulset.md) | [Back to README](../README.md) | Next: [Job](./06-job.md) →*
