# Kubernetes Short Notes ☸️

---

## 🛠️ Tools Setup

### KIND (Kubernetes IN Docker)
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
minikube start --driver=docker            # Local
minikube start --driver=docker --vm true  # On EC2
```

---

## 📦 Core Concepts

| Term | Description |
|------|-------------|
| **Cluster** | Group of nodes running containerized apps |
| **Node** | A machine (VM/physical) in the cluster |
| **Namespace** | Virtual cluster for resource isolation |
| **Pod** | Smallest deployable unit; wraps containers |
| **ReplicaSet** | Ensures N copies of a pod are always running |
| **Deployment** | Manages ReplicaSets + rolling updates |
| **DaemonSet** | Runs one pod per node automatically |
| **Manifest/YML** | YAML config file to define K8s resources |

---

## 🔧 Essential kubectl Commands

### Context & Cluster
```bash
kubectl config use-context kind-avi-cluster    # Switch context
kubectl cluster-info --context kind-avi-cluster
kubectl get nodes
kubectl get nodes --context kind-avi-cluster
```

### Namespaces
```bash
kubectl get namespaces / kubectl get ns
kubectl create ns nginx
kubectl apply -f namespace.yml
```

### Pods
```bash
kubectl get pods                         # default namespace
kubectl get pods -n nginx                # specific namespace
kubectl get pods -n kube-system
kubectl get pods -A                      # all namespaces
kubectl get pods -n nginx -o wide        # extra details (Node, IP)

kubectl run nginx --image=nginx          # imperative
kubectl run nginx --image=nginx -n nginx
kubectl apply -f pod.yml                 # declarative

kubectl describe pod nginx-pod -n nginx
kubectl exec -it nginx-pod -n nginx -- /bin/sh
kubectl delete pod nginx -n nginx
```

### Deployments
```bash
kubectl apply -f deployment.yml
kubectl delete -f deployment.yml
kubectl scale deployment/nginx-deployment -n nginx --replicas=8
kubectl set image deployment/nginx-deployment nginx=nginx:1.27 -n nginx
```

### ReplicaSets
```bash
cp deployment.yml replicasets.yml        # copy as base
kubectl apply -f replicasets.yml
kubectl get replicasets -n nginx
```

### DaemonSets
```bash
kubectl apply -f daemonts.yml
kubectl get pods -n nginx
kubectl get pods -n nginx -o wide        # check 1 pod per node
```

---

## 📄 Manifest Files

### Namespace
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: nginx
```

### Pod
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

### Deployment
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

### ReplicaSet
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
      name: nginx-rep-pod   # must match template labels
      app: nginx
  template:
    metadata:
      labels:
        name: nginx-rep-pod  # ✅ must match selector
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonsets
  namespace: nginx
spec:
  # ✅ No replicas field — runs 1 pod per node automatically
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

---

## ⚖️ Workload Comparison

| Feature | Deployment | ReplicaSet | DaemonSet |
|---|---|---|---|
| Replicas control | ✅ Manual | ✅ Manual | ❌ Auto (1/node) |
| Rolling updates | ✅ Yes | ❌ No | ✅ Yes |
| Runs on every node | ❌ No | ❌ No | ✅ Yes |
| Use case | App workloads | Low-level pods | Agents, logging |

---

## ⚡ Quick Tips

- **Imperative** = `kubectl run / create` — fast, no file needed
- **Declarative** = `kubectl apply -f file.yml` — reproducible, preferred
- Always use `-n <namespace>` if not in the default namespace
- `kubectl get pods -A` → see everything across all namespaces
- `kubectl get pods -o wide` → shows Pod IP and which Node it's on
- `kubectl describe` → debug pod issues (events, errors)
- `kubectl exec -it` → shell into a running container
- `kubectl set image` → triggers a zero-downtime rolling update
- `selector.matchLabels` must exactly match `template.metadata.labels`
