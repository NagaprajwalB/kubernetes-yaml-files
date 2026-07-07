# Kubernetes — Advanced Reference Guide

A production-grade reference for Kubernetes: architecture, core objects, networking, security, scaling, and troubleshooting.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Objects](#core-objects)
4. [Networking](#networking)
5. [Storage](#storage)
6. [Configuration & Secrets](#configuration--secrets)
7. [Scheduling & Scaling](#scheduling--scaling)
8. [Security](#security)
9. [Observability](#observability)
10. [Deployment Strategies](#deployment-strategies)
11. [Troubleshooting](#troubleshooting)
12. [Production Checklist](#production-checklist)
13. [Useful Commands](#useful-commands)

---

## Overview

Kubernetes (K8s) is a container orchestration platform that automates deployment, scaling, and management of containerized applications. It abstracts infrastructure into a declarative API: you describe the desired state, and controllers continuously reconcile actual state to match it.

**Key properties:**
- **Declarative** — desired state defined in YAML/JSON manifests
- **Self-healing** — restarts failed containers, reschedules pods on node failure
- **Horizontally scalable** — scale pods/nodes based on demand
- **Extensible** — CRDs, Operators, admission webhooks

---

## Architecture

### Control Plane Components

| Component | Role |
|---|---|
| `kube-apiserver` | Front door to the cluster; validates and processes REST requests; only component that talks to etcd |
| `etcd` | Distributed key-value store holding all cluster state |
| `kube-scheduler` | Assigns unscheduled pods to nodes based on resource requirements, affinity, taints |
| `kube-controller-manager` | Runs core controllers (Node, ReplicaSet, Endpoint, Job, etc.) |
| `cloud-controller-manager` | Integrates with cloud provider APIs (LBs, volumes, nodes) |

### Node Components

| Component | Role |
|---|---|
| `kubelet` | Agent on each node; ensures containers described in PodSpecs are running and healthy |
| `kube-proxy` | Maintains network rules for Service routing (iptables/IPVS) |
| Container runtime | containerd, CRI-O, etc. — actually runs containers via CRI |

```
                     ┌─────────────────────────────┐
                     │        Control Plane        │
                     │  ┌────────────┐ ┌─────────┐ │
                     │  │ apiserver  │ │  etcd   │ │
                     │  └────────────┘ └─────────┘ │
                     │  ┌────────────┐ ┌─────────┐ │
                     │  │ scheduler  │ │ ctrl-mgr│ │
                     │  └────────────┘ └─────────┘ │
                     └───────────────┬──────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
        ┌─────▼─────┐          ┌─────▼─────┐          ┌─────▼─────┐
        │  Node 1   │          │  Node 2   │          │  Node 3   │
        │ kubelet   │          │ kubelet   │          │ kubelet   │
        │ kube-proxy│          │ kube-proxy│          │ kube-proxy│
        │ pods...   │          │ pods...   │          │ pods...   │
        └───────────┘          └───────────┘          └───────────┘
```

### Reconciliation Loop

Every controller follows the same pattern:

```
watch(desired state) → compare(actual state) → act(diff) → repeat
```

This is the foundation for building custom Operators (via `controller-runtime` / `kubebuilder`).

---

## Core Objects

### Pod
Smallest deployable unit — one or more containers sharing network namespace and storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  labels:
    app: web
spec:
  containers:
    - name: app
      image: myrepo/web-app:1.4.2
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

### Deployment
Manages ReplicaSets for stateless apps; supports rolling updates and rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: myrepo/web-app:1.4.2
```

### StatefulSet
For stateful apps needing stable network identity and persistent storage (databases, queues).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

### DaemonSet
Ensures one pod per node (log collectors, monitoring agents, CNI plugins).

### Job / CronJob
Run-to-completion or scheduled tasks.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: myrepo/backup-tool:1.0
          restartPolicy: OnFailure
```

### Namespace
Logical isolation boundary for resource quotas, RBAC, network policy scoping.

---

## Networking

### Service Types

| Type | Use case |
|---|---|
| `ClusterIP` (default) | Internal-only access |
| `NodePort` | Exposes on a static port on every node |
| `LoadBalancer` | Provisions cloud LB (AWS ELB, GCP LB, etc.) |
| `ExternalName` | DNS CNAME alias to external service |
| `Headless` (`clusterIP: None`) | Direct pod DNS resolution — used by StatefulSets |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Ingress
HTTP/HTTPS routing layer, typically backed by NGINX, Traefik, or a cloud ingress controller.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["app.example.com"]
      secretName: web-app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80
```

### CNI
Actual pod networking (Calico, Cilium, Flannel) is delegated to a CNI plugin. Cilium and Calico additionally enforce `NetworkPolicy`.

### NetworkPolicy — default-deny example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Then allow specific traffic explicitly (namespace-scoped, label-based) — never rely on flat network access in production.

---

## Storage

- **PersistentVolume (PV)** — cluster-wide storage resource
- **PersistentVolumeClaim (PVC)** — namespaced request for storage
- **StorageClass** — dynamic provisioning template (EBS, PD, Ceph, etc.)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

Use `reclaimPolicy: Retain` for production databases to avoid accidental data loss on PVC deletion.

---

## Configuration & Secrets

- **ConfigMap** — non-sensitive config, env vars, mounted files
- **Secret** — base64-encoded (NOT encrypted by default) sensitive data

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: appuser
  password: change-me
```

**Production hardening:**
- Enable **encryption at rest** for etcd (`EncryptionConfiguration` with a KMS provider)
- Prefer external secret managers (Vault, AWS Secrets Manager, Sealed Secrets, External Secrets Operator) over raw `Secret` objects
- Never commit Secrets to git — use SOPS or Sealed Secrets for GitOps

---

## Scheduling & Scaling

### Resource Requests/Limits
Always set both. Requests drive scheduling decisions; limits enforce ceilings (OOMKill on memory, throttling on CPU).

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Vertical Pod Autoscaler (VPA)
Adjusts requests/limits automatically — useful for right-sizing, but avoid running HPA + VPA on CPU/memory simultaneously (conflicts).

### Cluster Autoscaler
Adds/removes nodes based on unschedulable pods and node utilization.

### Affinity, Anti-Affinity, Taints & Tolerations

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: ["web"]
        topologyKey: "kubernetes.io/hostname"
```

Spread replicas across nodes/zones with `topologySpreadConstraints` for real HA (not just replica count).

### PodDisruptionBudget (PDB)
Protects availability during voluntary disruptions (node drains, upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

---

## Security

### RBAC
Principle of least privilege — bind Roles/ClusterRoles to ServiceAccounts, never to `default` broadly.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security Standards
Enforce via `PodSecurity` admission (namespace labels) — `privileged`, `baseline`, `restricted`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

### SecurityContext (per-pod hardening)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  fsGroup: 10001
  seccompProfile:
    type: RuntimeDefault
containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

### Additional hardening
- Scan images (Trivy, Grype) in CI; block criticals
- Sign images (cosign) and enforce via admission control (Kyverno, OPA Gatekeeper)
- Rotate service account tokens (`BoundServiceAccountTokenVolume` is default since 1.22+)
- Restrict API server access via network policy / private endpoints
- Regularly patch control plane and node OS
- Use `NetworkPolicy` default-deny per namespace
- Enable audit logging on `kube-apiserver`

---

## Observability

| Layer | Tooling |
|---|---|
| Metrics | Prometheus + `kube-state-metrics` + node-exporter |
| Dashboards | Grafana |
| Logs | Fluent Bit / Fluentd → Loki or ELK |
| Tracing | OpenTelemetry Collector → Jaeger/Tempo |
| Alerting | Alertmanager |

**Key SLIs to track:** pod restart count, OOMKills, HPA scaling events, node pressure conditions, apiserver latency, etcd disk fsync latency.

---

## Deployment Strategies

| Strategy | Description |
|---|---|
| **Rolling update** | Default; gradual replacement, zero downtime if configured with PDB |
| **Blue/Green** | Two full environments; switch traffic via Service selector or Ingress |
| **Canary** | Small % of traffic to new version (Argo Rollouts, Flagger) |
| **GitOps** | Argo CD / Flux — cluster state reconciled from a git repo, not manual `kubectl apply` |

### Example: Argo Rollouts canary

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

---

## Troubleshooting

### Pod stuck in `Pending`
```bash
kubectl describe pod <pod> -n <ns>
```
Check `Events` for: insufficient CPU/memory, unbound PVC, node affinity/taint mismatch, no nodes matching selector.

### Pod stuck in `CrashLoopBackOff`
```bash
kubectl logs <pod> -n <ns> --previous
kubectl describe pod <pod> -n <ns>
```
Common causes: app crash on startup, failing liveness probe, missing config/secret, OOMKilled (check `Last State: Terminated, Reason: OOMKilled`).

### Pod stuck in `ImagePullBackOff`
Check image name/tag, registry auth (`imagePullSecrets`), and network egress to the registry.

### Service not routing traffic
```bash
kubectl get endpoints <svc> -n <ns>
```
Empty endpoints usually means the Service selector doesn't match pod labels, or pods aren't Ready (failing readiness probe).

### Node `NotReady`
```bash
kubectl describe node <node>
```
Check kubelet status, disk pressure, memory pressure, network plugin health, and `journalctl -u kubelet` on the node.

### High apiserver/etcd latency
Check etcd disk I/O (etcd is extremely latency-sensitive to disk fsync), object count bloat (too many Events/CRs), and apiserver request metrics (`apiserver_request_duration_seconds`).

### General debugging toolkit
```bash
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl debug node/<node> -it --image=busybox
kubectl top pod -n <ns>
kubectl top node
```

---

## Production Checklist

- [ ] Resource requests/limits set on every container
- [ ] Liveness, readiness, and startup probes configured
- [ ] PodDisruptionBudgets on all critical Deployments/StatefulSets
- [ ] HPA (and Cluster Autoscaler) configured with sane min/max
- [ ] NetworkPolicies enforcing default-deny + explicit allow
- [ ] RBAC least-privilege, no wildcard `*` verbs/resources in prod
- [ ] Pod Security Standard `restricted` enforced per namespace
- [ ] etcd encrypted at rest; etcd + control plane backed up regularly
- [ ] Image scanning + signing in CI/CD pipeline
- [ ] Centralized logging, metrics, tracing, and alerting in place
- [ ] Multi-AZ node spread with topology spread constraints
- [ ] GitOps-based deployment (no manual `kubectl apply` to prod)
- [ ] Runbooks for node failure, etcd restore, and rollback procedures
- [ ] Regular disaster recovery (DR) drills

---

## Useful Commands

```bash
# Context & cluster info
kubectl config get-contexts
kubectl cluster-info
kubectl get nodes -o wide

# Resource inspection
kubectl get all -n <ns>
kubectl describe <resource> <name> -n <ns>
kubectl get <resource> -o yaml

# Rollouts
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>

# Scaling
kubectl scale deployment/<name> --replicas=5 -n <ns>

# Debugging
kubectl logs -f <pod> -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl port-forward svc/<name> 8080:80 -n <ns>

# Apply / diff (GitOps-friendly)
kubectl diff -f manifest.yaml
kubectl apply -f manifest.yaml --dry-run=server

# Node maintenance
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

---

## References

- Official docs: https://kubernetes.io/docs/
- API reference: https://kubernetes.io/docs/reference/generated/kubernetes-api/
- CNCF landscape: https://landscape.cncf.io/