# CustomResourceDefinition (CRD)

> Extends the Kubernetes API with custom resource types. Once a CRD is installed, you can create instances of your custom resource just like built-in objects (Pods, Deployments, etc.).

**apiVersion:** `apiextensions.k8s.io/v1`  
**Kind:** `CustomResourceDefinition`

---

## Concept

```
You define a CRD:
  → Kubernetes adds a new API endpoint: /apis/<group>/<version>/<plural>
  → You can kubectl get/apply/delete your custom objects
  → A custom controller (operator) watches and acts on those objects
```

---

## Full Annotated Example

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # Format: <plural>.<group>
  name: databases.mycompany.io
  labels:
    app: db-operator

spec:
  # ─── API Group & Names ────────────────────────────────────────
  group: mycompany.io           # Your domain — must be a domain you own

  names:
    kind: Database              # PascalCase singular (used in manifests)
    listKind: DatabaseList      # Auto-generated list type
    plural: databases           # Lowercase plural (used in URLs and kubectl)
    singular: database          # Lowercase singular
    shortNames:
      - db                      # kubectl get db  works the same as kubectl get databases
    categories:
      - all                     # Include in kubectl get all

  # ─── Scope ────────────────────────────────────────────────────
  scope: Namespaced             # Namespaced | Cluster

  # ─── Versions ─────────────────────────────────────────────────
  versions:
    - name: v1alpha1
      served: true              # This version is active (responds to API requests)
      storage: false            # Only ONE version can be storage: true

    - name: v1
      served: true
      storage: true             # This version is stored in etcd

      # ─── Subresources ───────────────────────────────────────
      subresources:
        status: {}              # Enable /status subresource (separate update)
        scale:                  # Enable /scale subresource (compatible with HPA)
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
          labelSelectorPath: .status.labelSelector

      # ─── Printer Columns ──────────────────────────────────────
      # Shown in kubectl get output
      additionalPrinterColumns:
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
        - name: Version
          type: string
          description: The DB engine version
          jsonPath: .spec.version
          priority: 1           # 0 = always shown, 1+ = only with -o wide

      # ─── Schema Validation ────────────────────────────────────
      schema:
        openAPIV3Schema:
          type: object
          required: [spec]

          properties:
            spec:
              type: object
              required: [engine, version, storage]

              properties:
                engine:
                  type: string
                  description: "Database engine type"
                  enum: [postgres, mysql, mariadb]

                version:
                  type: string
                  description: "Engine version (e.g., 15.3)"
                  pattern: '^\d+\.\d+(\.\d+)?$'

                replicas:
                  type: integer
                  description: "Number of replicas"
                  minimum: 1
                  maximum: 10
                  default: 1

                storage:
                  type: object
                  required: [size]
                  properties:
                    size:
                      type: string
                      description: "Storage size (e.g., 20Gi)"
                      pattern: '^\d+(Ki|Mi|Gi|Ti|Pi|Ei)?$'
                    storageClassName:
                      type: string

                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string

                config:
                  type: object
                  description: "Engine-specific configuration"
                  x-kubernetes-preserve-unknown-fields: true  # Allow any fields

                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    schedule:
                      type: string
                      description: "Cron schedule for backups"
                    retentionDays:
                      type: integer
                      minimum: 1
                      maximum: 365
                      default: 30

            status:
              type: object
              x-kubernetes-preserve-unknown-fields: true  # Controller manages this

  # ─── Conversion ───────────────────────────────────────────────
  # (Only needed if you have multiple versions and need to convert between them)
  # conversion:
  #   strategy: Webhook
  #   webhook:
  #     conversionReviewVersions: [v1, v1beta1]
  #     clientConfig:
  #       service:
  #         name: db-operator-webhook
  #         namespace: operators
  #         path: /convert
```

---

## Using the CRD (Creating Custom Resources)

Once the CRD is installed, you create instances like any other Kubernetes object:

```yaml
apiVersion: mycompany.io/v1
kind: Database
metadata:
  name: my-postgres
  namespace: production
spec:
  engine: postgres
  version: "15.3"
  replicas: 3
  storage:
    size: 50Gi
    storageClassName: fast-ssd
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  config:
    max_connections: 200
    shared_buffers: 256MB
  backup:
    enabled: true
    schedule: "0 3 * * *"
    retentionDays: 30
```

```bash
# After applying the CRD, these all work:
kubectl get databases -n production
kubectl get db my-postgres -n production
kubectl describe database my-postgres -n production
kubectl delete database my-postgres -n production
kubectl get all -n production    # Included because of categories: [all]
```

---

## kubectl Commands

```bash
# Apply the CRD
kubectl apply -f database-crd.yaml

# Verify it was created
kubectl get crd databases.mycompany.io

# See the full CRD spec
kubectl describe crd databases.mycompany.io

# See the API resource it created
kubectl api-resources | grep mycompany

# Explain fields (schema documentation)
kubectl explain database
kubectl explain database.spec
kubectl explain database.spec.storage

# List all CRDs in the cluster
kubectl get crds

# Delete the CRD (also deletes ALL instances!)
kubectl delete crd databases.mycompany.io
```

---

## Minimal Operator Pattern

A CRD is just data — you need a **controller** (operator) to act on it:

```go
// Simplified Go operator pseudocode
func Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
    // 1. Fetch the custom resource
    db := &mycompanyv1.Database{}
    client.Get(ctx, req.NamespacedName, db)

    // 2. Create/update a StatefulSet based on the Database spec
    sts := buildStatefulSet(db)
    client.CreateOrUpdate(ctx, sts)

    // 3. Update the status
    db.Status.Phase = "Running"
    db.Status.ReadyReplicas = sts.Status.ReadyReplicas
    client.Status().Update(ctx, db)

    return reconcile.Result{RequeueAfter: 30 * time.Second}, nil
}
```

Popular operator frameworks: **operator-sdk**, **kubebuilder**, **kopf** (Python), **KUDO**.

---

## ⚠️ Gotchas

- Deleting a CRD **deletes all instances (custom resources) of that type** — no recovery.
- CRD names must be `<plural>.<group>` — the format is enforced.
- Schema validation (`openAPIV3Schema`) defaults are NOT applied retroactively to existing objects.
- Only ONE version can have `storage: true` — this is the canonical storage version.
- `x-kubernetes-preserve-unknown-fields: true` turns off validation for that field — use sparingly.
- CRDs are cluster-scoped objects, even if the resources they define are namespaced.

---

*← [Back to README](../README.md)*
