# IngressClass

**apiVersion:** `networking.k8s.io/v1` | **Kind:** `IngressClass`

> Defines which Ingress controller handles Ingress resources of this class. Allows multiple Ingress controllers in one cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    # Mark as the default class for Ingresses with no ingressClassName
    ingressclass.kubernetes.io/is-default-class: "true"

spec:
  # ─── Controller ───────────────────────────────────────────────
  controller: k8s.io/ingress-nginx   # nginx controller identifier
  # controller: traefik.io/ingress-controller
  # controller: ingress.k8s.aws/alb

  # ─── Parameters (controller-specific config) ──────────────────
  parameters:
    apiGroup: k8s.nginx.org
    kind: IngressClassParameters
    name: external-lb
    namespace: nginx-ingress
    scope: Namespace               # Namespace | Cluster
```

Use in Ingress:
```yaml
spec:
  ingressClassName: nginx
```

---

# EndpointSlice

**apiVersion:** `discovery.k8s.io/v1` | **Kind:** `EndpointSlice`

> Tracks network endpoints for a Service. Usually managed by Kubernetes automatically — you rarely create these manually.

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc12
  namespace: production
  labels:
    kubernetes.io/service-name: my-service   # Links to a Service

addressType: IPv4    # IPv4 | IPv6 | FQDN

endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
      serving: true
      terminating: false
    nodeName: node-1
    targetRef:
      kind: Pod
      name: my-app-abc12
      namespace: production

ports:
  - name: http
    protocol: TCP
    port: 8080
```

> In practice, EndpointSlices are auto-managed. You only create them manually when bypassing Service discovery (e.g., routing to an external service with a custom IP).

---

*← [Back to README](../README.md)*
