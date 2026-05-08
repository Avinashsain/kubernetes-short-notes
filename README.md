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
minikube start --driver=docker          # Local
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
| **Deployment** | Manages replica sets and rolling updates |
| **Manifest/YML** | YAML config file to define K8s resources |

---

## 🔧 Essential kubectl Commands

### Context & Cluster
```bash
kubectl config use-context kind-avi-cluster   # Switch context
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
kubectl get pods                        # default namespace
kubectl get pods -n nginx               # specific namespace
kubectl get pods -A                     # all namespaces
kubectl get pods -n kube-system

kubectl run nginx --image=nginx         # imperative
kubectl run nginx --image=nginx -n nginx
kubectl apply -f pod.yml                # declarative

kubectl describe pod nginx-pod -n nginx
kubectl exec -it nginx-pod -n nginx -- /bin/sh
kubectl delete pod nginx -n nginx
```

### Deployments
```bash
kubectl apply -f deployment.yml
kubectl get pods -n nginx -o wide                                          # extra details (node, IP)
kubectl scale deployment/nginx-deployment -n nginx --replicas=8           # scale up/down
kubectl set image deployment/nginx-deployment nginx=nginx:1.27 -n nginx   # rolling image update
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

---

## ⚡ Quick Tips

- **Imperative** = `kubectl run / create` (fast, no file needed)
- **Declarative** = `kubectl apply -f file.yml` (reproducible, preferred)
- Always use `-n <namespace>` if not in default namespace
- `kubectl get pods -A` → see everything across all namespaces
- `kubectl describe` → debug pod issues (events, errors)
- `kubectl exec -it` → shell into a running container
- `kubectl get pods -o wide` → shows Pod IP and which Node it's running on
- `kubectl set image` → triggers a rolling update (zero-downtime image change)
