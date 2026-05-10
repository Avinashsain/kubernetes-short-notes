# ☸️ Kubernetes Complete Notes

> **Stack:** KIND · Minikube · kubectl  
> **Last Updated:** May 2026

---

## 📖 Table of Contents

1. [Tools Setup](#tools-setup)
2. [Core Concepts](#core-concepts)
3. [kubectl Cheatsheet](#kubectl-cheatsheet)
4. [Manifest Files](#manifest-files)
5. [Workload Comparison](#workload-comparison)
6. [Storage — PV & PVC](#storage--pv--pvc)
7. [Networking — Services](#networking--services)
8. [Quick Tips](#quick-tips)

---

## 🛠️ Tools Setup

### KIND — Kubernetes IN Docker
```bash
brew install kind
kind --version
kind create cluster --name avi-cluster --config kind-config.yml
kubectl cluster-info --context kind-avi-cluster
```

### Minikube
```bash
brew install minikube
minikube version
minikube start --driver=docker              # Local machine
minikube start --driver=docker --vm true   # On EC2
```

### Context Switching
```bash
kubectl config use-context kind-avi-cluster     # switch context
kubectl get nodes --context kind-avi-cluster    # check nodes
```

---

## 📦 Core Concepts

| Resource | Description |
|---|---|
| **Cluster** | Group of nodes running containerized apps |
| **Node** | A machine (VM/physical) inside the cluster |
| **Namespace** | Virtual isolation boundary for resources |
| **Pod** | Smallest unit — wraps one or more containers |
| **ReplicaSet** | Ensures N pod replicas are always running |
| **Deployment** | Manages ReplicaSets + rolling updates + rollback |
| **DaemonSet** | Runs exactly one pod per node |
| **Job** | Runs a task once to completion |
| **CronJob** | Runs a Job on a schedule |
| **Service** | Stable network endpoint to reach pods |
| **PV** | PersistentVolume — actual storage on the node/cloud |
| **PVC** | PersistentVolumeClaim — pod's request for storage |

---

## 🔧 kubectl Cheatsheet

### Namespaces
```bash
kubectl get namespaces / kubectl get ns
kubectl create ns nginx
kubectl apply -f namespace.yml
```

### Pods
```bash
kubectl get pods                           # default namespace
kubectl get pods -n nginx                  # specific namespace
kubectl get pods -n kube-system            # system pods
kubectl get pods -A                        # all namespaces
kubectl get pods -n nginx -o wide          # extra details: Node IP

kubectl run nginx --image=nginx            # imperative (quick)
kubectl run nginx --image=nginx -n nginx
kubectl apply -f pod.yml                   # declarative (recommended)

kubectl describe pod nginx-pod -n nginx    # debug / events
kubectl exec -it nginx-pod -n nginx -- /bin/sh   # shell into pod
kubectl delete pod nginx -n nginx
```

### Deployments
```bash
kubectl apply -f deployment.yml
kubectl delete -f deployment.yml
kubectl get deployments -n nginx / kubectl get deploy -n nginx

kubectl scale deployment/nginx-deployment -n nginx --replicas=8
kubectl set image deployment/nginx-deployment nginx=nginx:1.27 -n nginx

kubectl rollout status deployment/nginx-deployment -n nginx    # watch update
kubectl rollout history deployment/nginx-deployment -n nginx   # version history
kubectl rollout undo deployment/nginx-deployment -n nginx      # rollback ↩️
```

### ReplicaSets
```bash
cp deployment.yml replicasets.yml          # use as base
kubectl apply -f replicasets.yml
kubectl get replicasets -n nginx / kubectl get rs -n nginx
```

### DaemonSets
```bash
kubectl apply -f daemonset.yml
kubectl get daemonsets -n nginx / kubectl get ds -n nginx
kubectl get pods -n nginx -o wide          # verify 1 pod per node
```

### Jobs
```bash
kubectl apply -f job.yml
kubectl get job -n nginx
kubectl logs pods/demo-job-2f8wl -n nginx  # see output
kubectl delete -f job.yml
```

### CronJobs
```bash
kubectl apply -f cronjob.yml
kubectl get cronjobs -n nginx / kubectl get cj -n nginx
kubectl get jobs -n nginx                  # jobs triggered by cronjob
kubectl describe cronjob minute-backup -n nginx
kubectl delete cronjob minute-backup -n nginx
```

### Storage
```bash
kubectl get pv                             # PersistentVolumes (cluster-wide)
kubectl get pvc                            # PersistentVolumeClaims
kubectl delete pvc/local-pvc
kubectl delete pv/local-pv
```

### Services
```bash
kubectl apply -f service.yml
kubectl get svc -n nginx
kubectl get all -n nginx                   # everything in namespace
kubectl describe svc nginx-service -n nginx
kubectl get endpoints -n nginx             # check pod-to-service binding

kubectl port-forward service/nginx-service -n nginx 80:80 --address=0.0.0.0
```

---

## 📄 Manifest Files

### 1. Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
```

### 2. Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

### 3. Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 4. ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicasets
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx-rep-pod   # ⚠️ must match template labels exactly
      app: nginx
  template:
    metadata:
      labels:
        name: nginx-rep-pod  # ✅ matches selector above
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 5. DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonsets
  namespace: nginx
spec:
  # ✅ No replicas — runs 1 pod per node automatically
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        name: nginx-dmn-pod
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 6. Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: demo-job
  namespace: nginx
spec:
  completions: 1      # run successfully 1 time
  parallelism: 1      # run 1 pod at a time
  template:
    metadata:
      name: demo-job-pod
      labels:
        app: batch-task
    spec:
      containers:
      - name: batch-container
        image: busybox:latest
        command: ["sh", "-c", "echo Hello Dosto! && sleep 10"]
      restartPolicy: Never   # ✅ Always/Never/OnFailure — Never for Jobs
```

### 7. CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: minute-backup
  namespace: nginx
spec:
  schedule: "*/2 * * * *"   # every 2 minutes
  jobTemplate:
    spec:
      template:
        metadata:
          name: minute-backup
          labels:
            app: minute-backup
        spec:
          containers:
          - name: backup-container
            image: busybox:latest
            command:
            - sh
            - -c
            - |
              cp /demo-data /backups ;
              echo "Backup Completed" ;
            volumeMounts:
            - name: data-volume
              mountPath: /demo-data
            - name: backup-volume
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: data-volume
            hostPath:
              path: /demo-data
              type: DirectoryOrCreate
          - name: backup-volume
            hostPath:
              path: /backups
              type: DirectoryOrCreate
```

### 8. PersistentVolume (PV)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  labels:
    app: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce           # one node can read/write
  persistentVolumeReclaimPolicy: Retain   # keep data after PVC deleted
  storageClassName: local-storage
  hostPath:
    path: /mnt/data           # path on the node
```

### 9. PersistentVolumeClaim (PVC)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi            # must match or be <= PV capacity
  storageClassName: local-storage   # must match PV storageClassName
```

### 10. Service — ClusterIP
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  selector:
    app: nginx          # routes to pods with this label
  ports:
    - protocol: TCP
      port: 80          # port clients use to reach Service
      targetPort: 80    # port container listens on
  type: ClusterIP       # internal only
```

---

## ⚖️ Workload Comparison

| Feature | Deployment | ReplicaSet | DaemonSet | Job | CronJob |
|---|---|---|---|---|---|
| **Replicas** | ✅ Manual | ✅ Manual | ❌ 1/node | ❌ N/A | ❌ N/A |
| **Rolling Update** | ✅ Yes | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **Rollback** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| **Scheduled** | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes |
| **Runs to completion** | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Use case** | Web apps | Low-level pods | Node agents | Batch tasks | Scheduled tasks |

---

## 💾 Storage — PV & PVC

```
PersistentVolume (PV)
  └── Actual storage (hostPath / cloud disk / NFS)
      └── cluster-wide, created by admin

PersistentVolumeClaim (PVC)
  └── Pod's request for storage
      └── namespace-scoped, created by developer

Binding:
  PVC requests 1Gi RWO  →  matches PV 1Gi RWO  →  Bound ✅
```

### Access Modes
| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | One node read+write |
| `ReadOnlyMany` | ROX | Many nodes read only |
| `ReadWriteMany` | RWX | Many nodes read+write |

### Reclaim Policies
| Policy | Behaviour after PVC deleted |
|---|---|
| `Retain` | PV kept, data preserved — manual cleanup |
| `Delete` | PV and data deleted automatically |
| `Recycle` | Data wiped, PV reused (deprecated) |

---

## 🌐 Networking — Services

### Traffic Flow
```
External User
     │
  LoadBalancer / NodePort
     │
  ClusterIP (stable internal IP)
     │
  kube-proxy routes
     │
  ┌──┴──┐
Pod-1  Pod-2   ← selected by label app: nginx
```

### Service Types
| Type | Accessible From | Use Case |
|---|---|---|
| `ClusterIP` | Inside cluster only | Internal microservices |
| `NodePort` | `<NodeIP>:<30000-32767>` | Dev/testing |
| `LoadBalancer` | External cloud IP | Production (AWS/GCP/Azure) |
| `ExternalName` | Maps to external DNS | Access outside services |

### Port-Forward (local testing)
```bash
kubectl port-forward service/nginx-service -n nginx 80:80 --address=0.0.0.0
# Now access via http://localhost:80
```

---

## 🕐 Cron Schedule Reference

```
"*/2  *  *  *  *"
  │   │  │  │  └── Day of week  (0–7)
  │   │  │  └───── Month        (1–12)
  │   │  └──────── Day of month (1–31)
  │   └─────────── Hour         (0–23)
  └─────────────── Minute       (0–59)
```

| Expression | Meaning |
|---|---|
| `*/2 * * * *` | Every 2 minutes |
| `0 * * * *` | Every hour |
| `0 0 * * *` | Daily at midnight |
| `0 9 * * 1` | Every Monday at 9am |
| `0 9 * * 1-5` | Weekdays at 9am |

---

## ⚡ Quick Tips

| Tip | Detail |
|---|---|
| **Declarative > Imperative** | `kubectl apply -f` is reproducible and version-controllable |
| **Always use `-n <ns>`** | Resources live in namespaces; default ≠ nginx |
| **`-o wide`** | Shows Pod IP + Node name |
| **`kubectl get all -n nginx`** | See everything in a namespace at once |
| **`selector` must match `labels`** | ReplicaSet/Deployment won't work if these differ |
| **DaemonSet has no `replicas`** | It auto-manages 1 pod per node |
| **`restartPolicy: Always`** | ❌ Not allowed in Job/CronJob — use `Never` or `OnFailure` |
| **PV is cluster-wide** | PVC is namespace-scoped |
| **PVC `storageClassName` must match PV** | Or binding will stay `Pending` |
| **`kubectl rollout undo`** | Your best friend after a bad deploy |
| **`kubectl get endpoints`** | Empty = selector labels don't match any pod |


# Taints & Tolerations — Kubernetes Short Notes

## What & Why

- **Taint** → applied on a **Node** — repels pods
- **Toleration** → applied on a **Pod** — allows it to schedule on tainted nodes
- Together they control **which pods can run on which nodes**

---

## Taint Syntax

```bash
kubectl taint node <node-name> key=value:effect
```

```bash
# Add taint
kubectl taint node avi-cluster-worker prod=true:NoSchedule

# Remove taint (trailing -)
kubectl taint node avi-cluster-worker prod=true:NoSchedule-
```

---

## 3 Taint Effects

| Effect | Behaviour |
|---|---|
| `NoSchedule` | Pod won't schedule on node unless it tolerates |
| `PreferNoSchedule` | Tries to avoid, but not guaranteed |
| `NoExecute` | Evicts running pods + blocks new ones |

---

## Toleration Syntax (in Pod spec)

```yaml
tolerations:
- key: "prod"
  operator: "Equal"    # Equal = match key+value
  value: "true"
  effect: "NoSchedule"
```

### Operator Types

| Operator | Meaning |
|---|---|
| `Equal` | key + value + effect must match |
| `Exists` | only key needs to match, no value needed |

```yaml
# Exists example — tolerates any value for that key
tolerations:
- key: "prod"
  operator: "Exists"
  effect: "NoSchedule"
```

---

## Tolerate Everything (not recommended)

```yaml
tolerations:
- operator: "Exists"   # tolerates ALL taints on any node
```

---

## Quick Flow

```
Node has Taint  →  Pod without toleration = BLOCKED
Node has Taint  →  Pod with matching toleration = ALLOWED
```

---

## Real World Use Cases

| Taint | Purpose |
|---|---|
| `env=prod:NoSchedule` | Only prod pods on prod nodes |
| `gpu=true:NoSchedule` | Only GPU workloads on GPU nodes |
| `team=backend:NoSchedule` | Isolate team workloads |
| `node.kubernetes.io/not-ready:NoExecute` | Kubernetes system taint |

---

## Important Notes

- Taints & tolerations don't **attract** pods to nodes — use **NodeSelector** or **NodeAffinity** for that
- Control-plane has a default taint — that's why regular pods don't run on it
- `NoExecute` with `tolerationSeconds` evicts pods after a timeout:

```yaml
tolerations:
- key: "prod"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
  tolerationSeconds: 300   # evict after 5 mins
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `key:value:effect` | Use `key=value:effect` |
| Typo in effect (`NoSchule`) | Must be exactly `NoSchedule` |
| Taint key mismatch (node vs pod) | Key + value + effect must match exactly |
| Wrong indentation in YAML | All toleration fields must align under `key` |

---

## List Taints on All Nodes

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Or detailed
kubectl describe nodes | grep -A5 Taints

# Specific node
kubectl describe node avi-cluster-worker | grep Taint
```


# Metrics Server & kubectl top — Kubernetes Short Notes

## What is Metrics Server?

Metrics Server is a **cluster-wide resource usage aggregator** — collects CPU and memory metrics from kubelets on each node and exposes them via the Kubernetes API.

---

## What It Enables

| Feature | Command |
|---|---|
| Pod CPU/memory usage | `kubectl top pod` |
| Node CPU/memory usage | `kubectl top node` |
| Horizontal Pod Autoscaler (HPA) | Requires metrics server |
| Vertical Pod Autoscaler (VPA) | Requires metrics server |

---

## Installation

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Common Issue on `kind` Clusters — TLS Error

```
x509: cannot validate certificate for node IP
```

### Fix — Add `--kubelet-insecure-tls` flag

```bash
kubectl patch deployment metrics-server \
  -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

---

## Common Issue — Pods Stuck in Pending

If nodes have taints, metrics-server pods can't schedule. Fix by adding tolerations:

```bash
kubectl patch deployment metrics-server \
  -n kube-system \
  --type=json \
  -p='[
    {"op":"add","path":"/spec/template/spec/tolerations","value":[
      {"key":"prod","operator":"Equal","value":"true","effect":"NoSchedule"},
      {"key":"node-role.kubernetes.io/control-plane","operator":"Exists","effect":"NoSchedule"}
    ]}
  ]'
```

---

## Verify Installation

```bash
kubectl get pods -n kube-system | grep metrics-server
# Should show Running, not Pending
```

---

## kubectl top Commands

```bash
kubectl top nodes                        # node level CPU + memory
kubectl top pods                         # pods in default namespace
kubectl top pods -n notes-app           # pods in specific namespace
kubectl top pods -A                      # ALL pods across all namespaces
kubectl top pods --sort-by=cpu -A       # sort by CPU usage
kubectl top pods --sort-by=memory -A    # sort by memory usage
```

---

## Understanding the Output

```
NAME                        CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
avi-cluster-control-plane   113m         1%       839Mi           10%
avi-cluster-worker          22m          0%       213Mi           2%
```

| Field | Meaning |
|---|---|
| `113m` | 113 millicores = 0.113 of 1 CPU core |
| `CPU(%)` | % of total CPU on that node |
| `839Mi` | Memory in Mebibytes |
| `MEMORY(%)` | % of total RAM on that node |

---

## kube-system Pod Roles

| Pod | Role |
|---|---|
| `kube-apiserver` | Handles all API requests — always heaviest |
| `etcd` | Database — stores all cluster state |
| `kube-controller-manager` | Manages deployments, replicasets |
| `kube-scheduler` | Decides which node pods go to |
| `coredns` | DNS resolution inside cluster |
| `kindnet` | Network plugin — one per node |
| `kube-proxy` | Network rules — one per node |
| `metrics-server` | Collects CPU/memory metrics |
| `local-path-provisioner` | Manages local storage (kind only) |

> `kube-apiserver` is always the heaviest — every `kubectl` command hits it.

---

## Quick Troubleshooting

```bash
# Metrics API not available?
kubectl get pods -n kube-system | grep metrics-server   # check if Running

# Pod stuck in Pending?
kubectl describe pod <metrics-server-pod> -n kube-system  # check Events

# No resources found?
kubectl top pods -A   # you may be in wrong namespace
```