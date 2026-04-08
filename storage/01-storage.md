# Storage: PersistentVolume, PersistentVolumeClaim & StorageClass

> Kubernetes separates storage provisioning (PersistentVolume) from storage consumption (PersistentVolumeClaim), with StorageClass enabling dynamic provisioning.

---

## The Storage Lifecycle

```
StorageClass         →   Defines HOW to provision storage
PersistentVolume     →   Represents actual storage (static or dynamic)
PersistentVolumeClaim →  A request for storage by a user/Pod
```

**Static provisioning:** Admin creates PV → User creates PVC → K8s binds them.  
**Dynamic provisioning:** User creates PVC with storageClassName → K8s creates PV automatically.

---

# StorageClass

**apiVersion:** `storage.k8s.io/v1`  
**Kind:** `StorageClass`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # Mark as default

# ─── Provisioner ──────────────────────────────────────────────────
# Who creates the volume — depends on your cloud/storage backend
provisioner: ebs.csi.aws.com           # AWS EBS CSI driver
# provisioner: pd.csi.storage.gke.io   # GCP Persistent Disk
# provisioner: disk.csi.azure.com      # Azure Disk
# provisioner: kubernetes.io/no-provisioner  # No dynamic provisioning (local)

# ─── Parameters ───────────────────────────────────────────────────
# Provisioner-specific settings
parameters:
  type: gp3                    # AWS: gp2, gp3, io1, io2, sc1, st1
  iops: "3000"                 # For io1/io2/gp3
  throughput: "125"            # MB/s for gp3
  encrypted: "true"
  # kmsKeyId: "arn:aws:kms:..."  # Custom KMS key

# ─── Reclaim Policy ───────────────────────────────────────────────
reclaimPolicy: Delete          # Delete | Retain
# Delete: PV is deleted when PVC is deleted (data lost!)
# Retain: PV is kept when PVC is deleted (manual cleanup needed)

# ─── Volume Binding Mode ──────────────────────────────────────────
volumeBindingMode: WaitForFirstConsumer   # Immediate | WaitForFirstConsumer
# Immediate: Create volume as soon as PVC is created
# WaitForFirstConsumer: Wait until a Pod is scheduled (respects topology)
# → Use WaitForFirstConsumer for regional clusters to avoid zone mismatch

# ─── Volume Expansion ─────────────────────────────────────────────
allowVolumeExpansion: true     # Allow PVCs to be resized (edit storage request)

# ─── Mount Options ────────────────────────────────────────────────
mountOptions:
  - discard
  - noatime
```

### Common StorageClass Examples

```yaml
# GCP SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Local storage (no dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

---

# PersistentVolume (PV)

**apiVersion:** `v1`  
**Kind:** `PersistentVolume`

> Usually created automatically by a StorageClass. Create manually for static provisioning (e.g., NFS, local disk, pre-provisioned cloud volumes).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv-01
  labels:
    type: local
    speed: fast

spec:
  # ─── Capacity ─────────────────────────────────────────────────
  capacity:
    storage: 100Gi

  # ─── Access Modes ─────────────────────────────────────────────
  accessModes:
    - ReadWriteOnce           # RWO: one node can mount read/write
    # - ReadOnlyMany          # ROX: many nodes can mount read-only
    # - ReadWriteMany         # RWX: many nodes can mount read/write (NFS, etc.)
    # - ReadWriteOncePod      # RWOP: one Pod can mount read/write (k8s 1.22+)

  # ─── Reclaim Policy ───────────────────────────────────────────
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle(deprecated)
  # Retain:  Keep data when PVC deleted. PV becomes "Released" — must be manually reclaimed
  # Delete:  Delete volume when PVC deleted (cloud volumes get deleted!)

  # ─── StorageClass ─────────────────────────────────────────────
  storageClassName: fast-ssd   # Must match PVC's storageClassName for binding

  # ─── Volume Mode ──────────────────────────────────────────────
  volumeMode: Filesystem       # Filesystem | Block
  # Filesystem: mount as filesystem (default)
  # Block: raw block device (for databases that manage their own filesystem)

  # ─── Mount Options ────────────────────────────────────────────
  mountOptions:
    - hard
    - nfsvers=4.1

  # ─── Node Affinity (for local volumes) ───────────────────────
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: [node-1]

  # ─── Volume Source ────────────────────────────────────────────
  # Choose ONE of these:

  # NFS
  nfs:
    server: nfs-server.example.com
    path: /exports/data

  # Local disk
  # local:
  #   path: /mnt/data

  # AWS EBS (static)
  # awsElasticBlockStore:
  #   volumeID: vol-0abc123def456
  #   fsType: ext4

  # GCE PD (static)
  # gcePersistentDisk:
  #   pdName: my-disk
  #   fsType: ext4

  # hostPath (single-node dev only)
  # hostPath:
  #   path: /data/pv-01
  #   type: DirectoryOrCreate

  # CSI (modern approach)
  # csi:
  #   driver: ebs.csi.aws.com
  #   volumeHandle: vol-0abc123def456
  #   fsType: ext4
  #   volumeAttributes:
  #     storage.kubernetes.io/csiProvisionerIdentity: ...
```

---

# PersistentVolumeClaim (PVC)

**apiVersion:** `v1`  
**Kind:** `PersistentVolumeClaim`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: production
  labels:
    app: my-app

spec:
  # ─── Access Modes ─────────────────────────────────────────────
  accessModes:
    - ReadWriteOnce

  # ─── Volume Mode ──────────────────────────────────────────────
  volumeMode: Filesystem

  # ─── Storage Request ──────────────────────────────────────────
  resources:
    requests:
      storage: 20Gi             # Request 20Gi of storage

  # ─── StorageClass ─────────────────────────────────────────────
  storageClassName: fast-ssd    # Use this StorageClass for dynamic provisioning
  # storageClassName: ""        # Bind to a PV with no StorageClass (static, exact)

  # ─── Selector (for static binding to specific PV) ─────────────
  selector:
    matchLabels:
      type: local
    # matchExpressions:
    #   - key: speed
    #     operator: In
    #     values: [fast, ultra]

  # ─── Volume Name (bind to specific PV) ───────────────────────
  # volumeName: data-pv-01     # Pin to a specific PV by name

  # ─── Datasource (clone or snapshot) ─────────────────────────
  # dataSource:
  #   name: existing-pvc
  #   kind: PersistentVolumeClaim
  #   # Or restore from snapshot:
  #   # name: my-snapshot
  #   # kind: VolumeSnapshot
  #   # apiGroup: snapshot.storage.k8s.io
```

---

## Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: app-data
      persistentVolumeClaim:
        claimName: app-data-pvc
        readOnly: false

  containers:
    - name: app
      volumeMounts:
        - name: app-data
          mountPath: /data
```

---

## PVC Lifecycle & Status

```
PVC Status:
  Pending  → Waiting for a PV to be bound (or provisioned)
  Bound    → Successfully bound to a PV
  Lost     → Bound PV no longer exists

PV Status:
  Available  → Free, not yet bound to a PVC
  Bound      → Bound to a PVC
  Released   → PVC was deleted, but PV data retained (needs manual cleanup)
  Failed     → Automatic reclamation failed
```

---

## Resize a PVC

```yaml
# Edit the PVC to increase storage (StorageClass must have allowVolumeExpansion: true)
spec:
  resources:
    requests:
      storage: 50Gi   # Increase from 20Gi to 50Gi
```
```bash
kubectl edit pvc app-data-pvc
# Or:
kubectl patch pvc app-data-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

---

## Access Modes by Storage Backend

| Backend | RWO | ROX | RWX | RWOP |
|---------|-----|-----|-----|------|
| AWS EBS | ✅ | ❌ | ❌ | ✅ |
| GCP PD | ✅ | ✅ | ❌ | ✅ |
| Azure Disk | ✅ | ❌ | ❌ | ✅ |
| NFS | ✅ | ✅ | ✅ | ❌ |
| CephFS | ✅ | ✅ | ✅ | ✅ |
| Local | ✅ | ❌ | ❌ | ✅ |

---

## ⚠️ Gotchas

- **PVCs created from StatefulSet `volumeClaimTemplates` are NOT deleted** when the StatefulSet is deleted.
- `reclaimPolicy: Delete` on a StorageClass **permanently deletes cloud volumes** when the PVC is deleted.
- You can only **expand** PVCs, never shrink them.
- `WaitForFirstConsumer` means the PVC stays `Pending` until a Pod using it is scheduled.
- Deleting a Pod does NOT delete its PVC — PVCs are independent objects.
- A PVC can only be bound to ONE PV at a time, but a PV can only be claimed by ONE PVC.

---

*← [Secret](../config/02-secret.md) | [Back to README](../README.md) | Next: [ServiceAccount](../rbac/01-serviceaccount.md) →*
