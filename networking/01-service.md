# Service

> Exposes a set of Pods as a stable network endpoint. Provides load balancing, service discovery, and a consistent IP/DNS name regardless of which Pods are running behind it.

**apiVersion:** `v1`  
**Kind:** `Service`

---

## Service Types Overview

| Type | Accessible From | Use Case |
|------|----------------|----------|
| `ClusterIP` | Inside cluster only | Internal service-to-service communication |
| `NodePort` | Node IP + port | Dev/testing, simple external access |
| `LoadBalancer` | External IP (cloud LB) | Production external access |
| `ExternalName` | Inside cluster | Alias for external DNS name |
| Headless (`clusterIP: None`) | Inside cluster | StatefulSets, direct Pod DNS |

---

## ClusterIP (Default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  type: ClusterIP                 # Default — can omit this line
  
  # ─── Selector ─────────────────────────────────────────────────
  # Routes traffic to Pods matching ALL these labels
  selector:
    app: my-app
    tier: backend
  
  # ─── Ports ────────────────────────────────────────────────────
  ports:
    - name: http                  # name is required when multiple ports
      protocol: TCP               # TCP | UDP | SCTP (default: TCP)
      port: 80                    # Port exposed by the Service (other services call this)
      targetPort: 8080            # Port on the Pod container (or port name)
      # targetPort: http          # Use named port from container spec
  
  # ─── Session Affinity ─────────────────────────────────────────
  sessionAffinity: None           # None | ClientIP
  # sessionAffinity: ClientIP
  # sessionAffinityConfig:
  #   clientIP:
  #     timeoutSeconds: 10800     # 3 hours (default)
```

---

## NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - name: http
      port: 80                    # ClusterIP port
      targetPort: 8080            # Pod port
      nodePort: 30080             # External port on each node (30000–32767)
                                  # Omit to auto-assign
```
Access via: `http://<any-node-ip>:30080`

---

## LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
  annotations:
    # AWS
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    # GCP
    # cloud.google.com/load-balancer-type: External
    # Azure
    # service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
  
  # Optional: specify IP (if your cloud supports it)
  loadBalancerIP: "34.100.200.1"
  
  # Restrict who can access the LB
  loadBalancerSourceRanges:
    - "10.0.0.0/8"
    - "172.16.0.0/12"
  
  # Only route to nodes with matching Pods (reduces extra hop)
  externalTrafficPolicy: Local    # Cluster (default) | Local
```

---

## Headless Service (for StatefulSets)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None                 # No virtual IP — direct DNS to each Pod
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

DNS records:
```
postgres-headless.default.svc.cluster.local       → all Pod IPs (round-robin)
postgres-0.postgres-headless.default.svc.cluster.local → specific Pod IP
postgres-1.postgres-headless.default.svc.cluster.local → specific Pod IP
```

---

## ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.example.com  # CNAME alias — no selector, no ports needed
```

Inside the cluster, `external-db.production.svc.cluster.local` resolves to `mydb.example.com`.

---

## Full Annotated Example (Multi-Port Service)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
  annotations:
    # Human-readable description (for tooling/docs)
    kubernetes.io/description: "Main application service — HTTP and gRPC"

spec:
  type: ClusterIP
  
  selector:
    app: my-app
  
  ports:
    # HTTP API
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    
    # gRPC
    - name: grpc
      protocol: TCP
      port: 9090
      targetPort: 9090
    
    # Prometheus metrics (usually internal only)
    - name: metrics
      protocol: TCP
      port: 9100
      targetPort: 9100
  
  # Keep connection to same Pod for 3 hours
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  
  # Publish Not-Ready addresses (useful for StatefulSets during init)
  publishNotReadyAddresses: false

  # IP family policy
  ipFamilyPolicy: SingleStack     # SingleStack | PreferDualStack | RequireDualStack
  ipFamilies:
    - IPv4
```

---

## Service DNS

Every Service gets automatic DNS entries:

```
<service-name>.<namespace>.svc.cluster.local
```

Within the same namespace, just use `<service-name>`.

```bash
# From a Pod in the same namespace:
curl http://my-app/api

# From a Pod in another namespace:
curl http://my-app.production/api

# Fully qualified:
curl http://my-app.production.svc.cluster.local/api
```

---

## `externalTrafficPolicy` Explained

| Policy | Behavior | Pros | Cons |
|--------|---------|------|------|
| `Cluster` (default) | Route to any node, then to any Pod | Even load distribution | Extra hop, source IP lost |
| `Local` | Route only to Pods on the receiving node | Preserves source IP, no extra hop | Uneven distribution if Pods aren't spread |

---

## kubectl Commands

```bash
# Expose a deployment quickly
kubectl expose deployment my-app --port=80 --target-port=8080

# List services
kubectl get svc

# Get service details
kubectl describe svc my-app

# Test from within cluster
kubectl run tmp --image=busybox -it --rm -- wget -qO- http://my-app

# Port-forward for local testing
kubectl port-forward svc/my-app 8080:80
```

---

## ⚠️ Gotchas

- A Service with no matching Pods is valid but routes to nothing — check label selectors carefully.
- `targetPort` must match the `containerPort` in the Pod spec (or use named ports for clarity).
- `LoadBalancer` provisions a cloud load balancer — this usually has a cost even with 0 traffic.
- `externalTrafficPolicy: Local` on a NodePort/LB drops traffic if there's no Pod on that node.
- Headless Services (`clusterIP: None`) with no selector create no DNS records — add a selector or manage Endpoints manually.

---

*← [CronJob](../workloads/07-cronjob.md) | [Back to README](../README.md) | Next: [Ingress](./02-ingress.md) →*
