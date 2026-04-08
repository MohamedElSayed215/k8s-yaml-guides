# ConfigMap

> Stores non-sensitive configuration data as key-value pairs. Decouples configuration from container images, allowing you to change config without rebuilding images.

**apiVersion:** `v1`  
**Kind:** `ConfigMap`

---

## Minimal Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
```

---

## Full Annotated Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: my-app

# ─── data: UTF-8 string values ────────────────────────────────────
data:
  # Simple key-value pairs
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_PORT: "5432"
  FEATURE_FLAG_DARK_MODE: "true"

  # Multi-line config file (literal block scalar)
  app.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://postgres:5432/mydb
    spring.jpa.hibernate.ddl-auto=validate
    logging.level.root=INFO

  # JSON config
  config.json: |
    {
      "database": {
        "host": "postgres",
        "port": 5432,
        "pool_size": 10
      },
      "cache": {
        "host": "redis",
        "ttl": 300
      }
    }

  # YAML config
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      host: postgres
      name: myapp
    redis:
      host: redis
      port: 6379

  # nginx.conf
  nginx.conf: |
    server {
      listen 80;
      server_name _;
      location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }

# ─── binaryData: base64-encoded binary values ─────────────────────
# Use for binary files (images, keystores, etc.)
binaryData:
  truststore.jks: <base64-encoded-content>

# ─── immutable: prevent changes after creation (k8s 1.21+) ────────
immutable: false                  # true = cannot be updated (must delete & recreate)
                                  # Improves performance at large scale
```

---

## Using ConfigMap in Pods

### Method 1: As Environment Variables (single key)
```yaml
containers:
  - name: app
    env:
      - name: APP_ENV
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: APP_ENV
            optional: false       # Pod fails to start if key is missing
```

### Method 2: All Keys as Environment Variables
```yaml
containers:
  - name: app
    envFrom:
      - configMapRef:
          name: app-config
          optional: false
      # Keys become env var names — dots in key names become underscores
      # e.g., app.properties → not a valid env var name, skip or use subPath
```

### Method 3: As a Volume (full ConfigMap as files)
```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
      defaultMode: 0644           # File permissions (octal)

containers:
  - name: app
    volumeMounts:
      - name: config-volume
        mountPath: /etc/app       # Each key becomes a file: /etc/app/APP_ENV, etc.
        readOnly: true
```

### Method 4: Specific File from ConfigMap (subPath)
```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config

containers:
  - name: nginx
    volumeMounts:
      - name: config-volume
        mountPath: /etc/nginx/nginx.conf
        subPath: nginx.conf       # Mount only this key as a specific file path
        readOnly: true
```

### Method 5: Specific Keys to Specific Files
```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
        - key: app.properties
          path: application.properties  # Override the filename
          mode: 0400
        - key: config.json
          path: config/settings.json   # Can include subdirectory
```

---

## Creating ConfigMaps with kubectl

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# From a file (key = filename, value = file content)
kubectl create configmap nginx-config \
  --from-file=nginx.conf

# From a file with custom key name
kubectl create configmap app-config \
  --from-file=config.properties=./local-config.properties

# From a directory (all files become keys)
kubectl create configmap app-config \
  --from-file=./config/

# From an env file (KEY=VALUE format)
kubectl create configmap app-config \
  --from-env-file=.env.production
```

---

## ConfigMap Updates

When a ConfigMap is updated:
- **Environment variables**: NOT updated — Pod must be restarted.
- **Volume mounts**: Updated automatically (after ~1 min kubelet sync period).

> ⚠️ `subPath` mounts do **NOT** update automatically — restart the Pod.

---

## ⚠️ Gotchas

- ConfigMaps are **not encrypted** — never store passwords, tokens, or certificates here. Use [Secret](./02-secret.md).
- Max size is **1 MiB** — large configs should be split or stored externally.
- Keys must match the regex `[-._a-zA-Z0-9]+` — no spaces.
- Setting `immutable: true` is a one-way door — you must delete and recreate the ConfigMap to change it.
- `envFrom` with keys containing dots (e.g., `app.properties`) creates invalid env var names — use `env.valueFrom.configMapKeyRef` for those.

---

*← [NetworkPolicy](../networking/04-networkpolicy.md) | [Back to README](../README.md) | Next: [Secret](./02-secret.md) →*
