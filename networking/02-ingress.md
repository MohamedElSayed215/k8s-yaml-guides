# Ingress

> Manages external HTTP/HTTPS access to Services in a cluster. Provides host-based routing, path-based routing, TLS termination, and load balancing — all in one resource.

**apiVersion:** `networking.k8s.io/v1`  
**Kind:** `Ingress`

> ⚠️ Ingress requires an **Ingress Controller** to be installed (nginx-ingress, Traefik, HAProxy, AWS ALB Ingress, GKE Ingress, etc.). The Ingress object itself does nothing without a controller.

---

## Minimal Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
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

## Full Annotated Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
  annotations:
    # ─── Ingress Class (alternative to spec.ingressClassName) ───
    kubernetes.io/ingress.class: "nginx"

    # ─── nginx-specific annotations ───────────────────────────
    nginx.ingress.kubernetes.io/rewrite-target: /           # Strip path prefix
    nginx.ingress.kubernetes.io/ssl-redirect: "true"        # Force HTTPS
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"      # Max upload size
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # Rate limiting (nginx)
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"

    # Auth
    nginx.ingress.kubernetes.io/auth-url: "http://auth-service.default.svc.cluster.local/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://myapp.example.com/login"

    # cert-manager TLS automation
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  # ─── Ingress Class ────────────────────────────────────────────
  ingressClassName: nginx         # References an IngressClass object

  # ─── TLS ──────────────────────────────────────────────────────
  tls:
    - hosts:
        - myapp.example.com
        - api.example.com
      secretName: myapp-tls       # Secret must exist with tls.crt and tls.key
                                  # cert-manager creates this automatically

  # ─── Rules ────────────────────────────────────────────────────
  rules:
    # Host-based routing: myapp.example.com
    - host: myapp.example.com
      http:
        paths:
          # Path: / → frontend service
          - path: /
            pathType: Prefix      # Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: frontend
                port:
                  number: 80
                  # name: http   # Can use port name instead

          # Path: /api → backend service
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-api
                port:
                  number: 8080

          # Path: /static — exact match only
          - path: /static
            pathType: Exact
            backend:
              service:
                name: static-files
                port:
                  number: 80

    # Separate host: api.example.com
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-api
                port:
                  number: 8080

    # No host = default for all unmatched hostnames
    # - http:
    #     paths:
    #       - path: /
    #         pathType: Prefix
    #         backend:
    #           service:
    #             name: default-backend
    #             port:
    #               number: 80

  # ─── Default Backend ──────────────────────────────────────────
  # Handle requests that don't match any rule
  defaultBackend:
    service:
      name: default-404
      port:
        number: 80
```

---

## pathType Values

| Type | Behavior |
|------|---------|
| `Prefix` | Matches the path prefix (e.g., `/api` matches `/api`, `/api/v1`, `/api/users`) |
| `Exact` | Matches the exact path only (e.g., `/api` matches only `/api`) |
| `ImplementationSpecific` | Controller decides (nginx uses regex, others may differ) |

---

## TLS with cert-manager

```yaml
# 1. Create a ClusterIssuer (once per cluster)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx

---
# 2. Annotate your Ingress — cert-manager creates the Secret automatically
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls         # cert-manager will create/renew this
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

## Common nginx Annotations

```yaml
annotations:
  # Rewrites
  nginx.ingress.kubernetes.io/rewrite-target: /$2   # Strip /api/v1 → /
  nginx.ingress.kubernetes.io/use-regex: "true"

  # WebSocket support
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-http-version: "1.1"

  # Sticky sessions
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
  nginx.ingress.kubernetes.io/session-cookie-expires: "172800"

  # Custom headers
  nginx.ingress.kubernetes.io/configuration-snippet: |
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

  # Client certificate auth
  nginx.ingress.kubernetes.io/auth-tls-secret: "production/ca-secret"
  nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
```

---

## kubectl Commands

```bash
# Apply
kubectl apply -f ingress.yaml

# View all Ingresses
kubectl get ingress -A

# Describe (shows events — useful for debugging)
kubectl describe ingress my-app -n production

# Check TLS certificate (cert-manager)
kubectl get certificate -n production
kubectl describe certificate myapp-tls -n production

# Test routing
curl -H "Host: myapp.example.com" http://<ingress-ip>/api
```

---

## ⚠️ Gotchas

- **An Ingress without an Ingress Controller is useless** — install one first.
- `ingressClassName` takes precedence over the `kubernetes.io/ingress.class` annotation (use `ingressClassName` for k8s 1.18+).
- Path ordering matters in some controllers — more specific paths should come first.
- TLS cert Secrets must be in the **same namespace** as the Ingress.
- HTTP-to-HTTPS redirect must be configured via annotation — the Ingress spec has no native redirect support.
- `pathType: Prefix` on `/api` also matches `/apiv2` — use trailing slashes or `Exact` to be precise.

---

*← [Service](./01-service.md) | [Back to README](../README.md) | Next: [NetworkPolicy](./04-networkpolicy.md) →*
