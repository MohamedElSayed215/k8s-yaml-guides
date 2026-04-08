# ☸️ Kubernetes YAML Reference Guides

> A complete, annotated reference for writing Kubernetes manifest files — every object, every field, with real-world examples.

---

## 📚 Table of Contents

### 🐳 Workloads
| Object | Description |
|--------|-------------|
| [Pod](./workloads/01-pod.md) | Smallest deployable unit — one or more containers |
| [Deployment](./workloads/02-deployment.md) | Manages stateless apps with rolling updates |
| [ReplicaSet](./workloads/03-replicaset.md) | Ensures a stable set of replica Pods |
| [StatefulSet](./workloads/04-statefulset.md) | Manages stateful apps with stable identity |
| [DaemonSet](./workloads/05-daemonset.md) | Runs a Pod on every (or selected) node |
| [Job](./workloads/06-job.md) | Runs a task to completion |
| [CronJob](./workloads/07-cronjob.md) | Runs Jobs on a schedule |

### 🌐 Networking
| Object | Description |
|--------|-------------|
| [Service](./networking/01-service.md) | Exposes Pods as a network service |
| [Ingress](./networking/02-ingress.md) | HTTP/S routing to Services |
| [IngressClass](./networking/03-ingressclass.md) | Defines which Ingress controller to use |
| [NetworkPolicy](./networking/04-networkpolicy.md) | Controls Pod-level traffic |
| [EndpointSlice](./networking/05-endpointslice.md) | Tracks network endpoints |

### ⚙️ Configuration
| Object | Description |
|--------|-------------|
| [ConfigMap](./config/01-configmap.md) | Non-sensitive configuration data |
| [Secret](./config/02-secret.md) | Sensitive data (passwords, tokens) |

### 💾 Storage
| Object | Description |
|--------|-------------|
| [PersistentVolume](./storage/01-persistentvolume.md) | Cluster-level storage resource |
| [PersistentVolumeClaim](./storage/02-persistentvolumeclaim.md) | Request for storage by a user |
| [StorageClass](./storage/03-storageclass.md) | Dynamic storage provisioning |
| [VolumeAttachment](./storage/04-volumeattachment.md) | Attaches external volumes to nodes |

### 🔐 RBAC
| Object | Description |
|--------|-------------|
| [ServiceAccount](./rbac/01-serviceaccount.md) | Identity for Pods/processes |
| [Role](./rbac/02-role.md) | Namespace-scoped permissions |
| [ClusterRole](./rbac/03-clusterrole.md) | Cluster-scoped permissions |
| [RoleBinding](./rbac/04-rolebinding.md) | Grants a Role to subjects |
| [ClusterRoleBinding](./rbac/05-clusterrolebinding.md) | Grants a ClusterRole to subjects |

### 🏗️ Cluster Management
| Object | Description |
|--------|-------------|
| [Namespace](./cluster/01-namespace.md) | Virtual cluster for resource isolation |
| [Node](./cluster/02-node.md) | Worker machine in the cluster |
| [ResourceQuota](./cluster/03-resourcequota.md) | Limits resource consumption per namespace |
| [LimitRange](./cluster/04-limitrange.md) | Default/max limits per container |
| [HorizontalPodAutoscaler](./cluster/05-hpa.md) | Auto-scales Pods based on metrics |
| [PodDisruptionBudget](./cluster/06-pdb.md) | Limits voluntary disruptions |
| [PriorityClass](./cluster/07-priorityclass.md) | Assigns scheduling priority to Pods |
| [CustomResourceDefinition](./cluster/08-crd.md) | Extends the Kubernetes API |

### 📦 Real-World Examples
| Example | Description |
|---------|-------------|
| [Full Web App](./examples/web-app.md) | Deployment + Service + Ingress + ConfigMap |
| [Stateful Database](./examples/stateful-db.md) | StatefulSet + PVC + Service + Secret |
| [Background Worker](./examples/background-worker.md) | Deployment + ConfigMap + RBAC |
| [CronJob Pipeline](./examples/cronjob-pipeline.md) | CronJob + Job + PVC |

---

## 🚀 Quick Start

Every YAML file in Kubernetes shares this base structure:

```yaml
apiVersion: <group>/<version>   # API group and version
kind: <ObjectKind>              # The type of object
metadata:                       # Object identity
  name: my-object
  namespace: default            # Omit for cluster-scoped objects
  labels:                       # Key-value pairs for selection
    app: my-app
  annotations:                  # Non-identifying metadata
    description: "My object"
spec:                           # Desired state (most objects)
  ...
status:                         # Current state (managed by k8s, read-only)
  ...
```

## 🔖 Common `apiVersion` Values

| Kind | apiVersion |
|------|-----------|
| Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet, Service, ConfigMap, Secret, ServiceAccount, Endpoints | `v1` or `apps/v1` |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` |
| Job, CronJob | `batch/v1` |
| Ingress, IngressClass | `networking.k8s.io/v1` |
| NetworkPolicy | `networking.k8s.io/v1` |
| Role, ClusterRole, RoleBinding, ClusterRoleBinding | `rbac.authorization.k8s.io/v1` |
| HorizontalPodAutoscaler | `autoscaling/v2` |
| PodDisruptionBudget | `policy/v1` |
| StorageClass, VolumeAttachment | `storage.k8s.io/v1` |
| CustomResourceDefinition | `apiextensions.k8s.io/v1` |

## 💡 Tips

- Use `kubectl explain <resource>` to see all available fields
- Use `kubectl apply -f <file>.yaml` to create/update resources
- Use `kubectl diff -f <file>.yaml` to preview changes before applying
- Use `kubectl dry-run=client -f <file>.yaml` to validate without applying
- Label everything consistently — labels drive selectors, services, and deployments

---

*Happy shipping! ☸️*
