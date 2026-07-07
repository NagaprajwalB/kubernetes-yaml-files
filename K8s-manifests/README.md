# Kubernetes YAML Manifest Library

A complete set of example manifests for every commonly used Kubernetes object type, organized by category. Replace image names, resource sizes, and labels with your own values before applying.

## Folder structure

```
01-workloads/                 Pod, ReplicaSet, ReplicationController, Deployment, StatefulSet, DaemonSet, Job, CronJob

02-services-networking/       Service (ClusterIP/NodePort/LoadBalancer/ExternalName/Headless), Ingress, NetworkPolicy

03-storage/                   PersistentVolume, PersistentVolumeClaim, StorageClass

04-config/                    ConfigMap, Secret, pod consuming both

05-autoscaling-scheduling/    HPA, VPA, PodDisruptionBudget, affinity/topology spread, PriorityClass

06-security-rbac/             ServiceAccount, Role/RoleBinding, ClusterRole/ClusterRoleBinding, hardened SecurityContext

07-policy/                    Namespace (Pod Security Standards), ResourceQuota, LimitRange, CRD
```

## Usage

```bash
# Apply a single manifest
kubectl apply -f 01-workloads/deployment.yaml

# Apply an entire category
kubectl apply -f 02-services-networking/

# Apply everything (review carefully first — some resources are cluster-scoped)
kubectl apply -R -f .

# Validate without applying
kubectl apply -f 01-workloads/deployment.yaml --dry-run=server
```

## Notes

- Files with multiple `---`-separated documents (e.g. Role + RoleBinding) can be applied together in one `kubectl apply -f`.
- Namespaced examples assume a `production` namespace — create it first (`07-policy/namespace-pod-security.yaml`) or edit the `namespace:` field.
- `Secret` examples use placeholder values — never commit real secrets to git.
- Some resources (VPA, CRDs) require their respective controllers/CRDs to already be installed in the cluster.
