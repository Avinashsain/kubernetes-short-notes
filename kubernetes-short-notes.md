# Kubernetes In One Shot — Short Notes

---

## 1. Core Concepts

### Monolithic vs Microservices
| Monolithic | Microservices |
|---|---|
| Single deployable unit | Multiple small independent services |
| Hard to scale individually | Each service scales independently |
| One failure can break all | Failure isolated to one service |
| Simple to develop initially | Complex but flexible |

### Kubernetes Architecture
```
Control Plane                   Worker Nodes
├── API Server                  ├── kubelet
├── etcd (database)             ├── kube-proxy
├── Scheduler                   └── Container Runtime
└── Controller Manager              (containerd/docker)
```

### kubectl Basics
```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl logs <pod-name> -n <namespace>
kubectl exec -it <pod-name> -- sh
```

### Pods
- Smallest deployable unit in Kubernetes
- Contains one or more containers
- Shares network and storage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
```

### Namespaces
```bash
kubectl get namespaces
kubectl create namespace myapp
kubectl get pods -n myapp
```

### Labels, Selectors & Annotations
```yaml
metadata:
  labels:
    app: nginx        # used for selection
    env: prod
  annotations:
    description: "web server"   # metadata only, not for selection
```

---

## 2. Workloads

### Deployments
- Manages ReplicaSets — ensures desired number of pods
- Supports rolling updates and rollbacks

```bash
kubectl create deployment nginx --image=nginx:1.27
kubectl scale deployment nginx --replicas=3
kubectl rollout status deployment nginx
kubectl rollout undo deployment nginx      # rollback
```

### StatefulSets
- For stateful apps (databases) — pods get stable identity
- Pods named: `mysql-0`, `mysql-1`, `mysql-2`
- Each pod gets its own PersistentVolume

### DaemonSets
- Runs **one pod per node** automatically
- Used for: log collectors, monitoring agents, network plugins

### ReplicaSets
- Ensures N copies of a pod are always running
- Usually managed by Deployments, not directly

### Jobs
- Runs a pod to **completion** (one-time tasks)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["echo", "done"]
      restartPolicy: Never
```

### CronJobs
- Runs Jobs on a **schedule**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "0 * * * *"   # every hour
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: worker
            image: busybox
            command: ["echo", "scheduled"]
          restartPolicy: Never
```

---

## 3. Networking

### Services
| Type | Use Case |
|---|---|
| `ClusterIP` | Internal only (default) |
| `NodePort` | Expose on node IP + port |
| `LoadBalancer` | Cloud load balancer |
| `ExternalName` | DNS alias to external service |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Ingress
- HTTP/HTTPS routing to services based on host/path
- Requires an Ingress Controller (nginx, traefik)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### Network Policies
- Controls traffic flow between pods (firewall rules)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
```

---

## 4. Storage

### Persistent Volumes (PV)
- Cluster-level storage resource provisioned by admin

### Persistent Volume Claims (PVC)
- Pod's request for storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

### StorageClasses
- Dynamic provisioning of PVs

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: docker.io/hostpath
```

### ConfigMaps
- Store non-sensitive config as key-value pairs

```bash
kubectl create configmap app-config --from-literal=ENV=production
```

### Secrets
- Store sensitive data (base64 encoded)

```bash
kubectl create secret generic db-secret \
  --from-literal=PASSWORD=mysecret
```

---

## 5. Scaling and Scheduling

### HPA — Horizontal Pod Autoscaler
- Scales pods based on CPU/memory

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: notes-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: notes-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### VPA — Vertical Pod Autoscaler
- Adjusts CPU/memory **requests and limits** automatically

### Node Affinity
- Attract pods to specific nodes

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: env
          operator: In
          values: ["prod"]
```

### Taints & Tolerations
```bash
# Taint a node
kubectl taint node worker-1 env=prod:NoSchedule

# Toleration in pod
tolerations:
- key: "env"
  operator: "Equal"
  value: "prod"
  effect: "NoSchedule"
```

### Resource Quotas & Limits
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"
```

### Probes
```yaml
livenessProbe:           # restart if unhealthy
  httpGet:
    path: /
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:          # remove from service if not ready
  httpGet:
    path: /
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10
```

---

## 6. Cluster Administration

### RBAC — Role Based Access Control
```yaml
# Role
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

# RoleBinding
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
subjects:
- kind: User
  name: avinash
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Cluster Upgrade
```bash
# Upgrade kubeadm
apt-get update && apt-get install -y kubeadm=1.28.0-00

# Upgrade control plane
kubeadm upgrade apply v1.28.0

# Upgrade kubelet
apt-get install -y kubelet=1.28.0-00
systemctl restart kubelet
```

### CRDs — Custom Resource Definitions
- Extend Kubernetes with custom resources

```bash
kubectl get crds
kubectl describe crd <crd-name>
```

---

## 7. Monitoring and Logging

### Metrics Server
```bash
# Install
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Use
kubectl top nodes
kubectl top pods -A
kubectl top pods --sort-by=cpu -A
```

### Logging
```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> --previous       # crashed container
kubectl logs -f <pod-name>               # follow live logs
kubectl logs <pod-name> -c <container>   # specific container
```

### Monitoring Tools
| Tool | Purpose |
|---|---|
| Prometheus | Metrics collection |
| Grafana | Metrics visualization |
| Loki | Log aggregation |
| Alertmanager | Alerts |

---

## 8. Advanced Features

### Helm
- Package manager for Kubernetes

```bash
helm install my-app ./chart
helm upgrade my-app ./chart
helm rollback my-app 1
helm list
helm uninstall my-app
```

### Operators
- Automate complex stateful app management
- Extends Kubernetes with custom controllers

### Service Mesh (Istio/Linkerd)
- mTLS, traffic management, observability between services

### Kubernetes API
```bash
kubectl api-resources         # list all resources
kubectl api-versions          # list API versions
kubectl explain pod.spec      # explain fields
```

---

## 9. Security

### Pod Security Standards (PSS)
| Level | Description |
|---|---|
| `Privileged` | No restrictions |
| `Baseline` | Minimal restrictions |
| `Restricted` | Heavily restricted |

### Image Scanning
- Scan images for vulnerabilities before deploying
- Tools: Trivy, Snyk, Grype

```bash
trivy image nginx:latest
```

### Secrets Encryption
```bash
# Never store secrets in plain text
kubectl create secret generic db-secret \
  --from-literal=PASSWORD=mysecret

# Use sealed-secrets or external vaults (HashiCorp Vault, AWS Secrets Manager)
```

---

## 10. Cloud-Native Kubernetes

### Managed Services
| Provider | Service |
|---|---|
| AWS | EKS (Elastic Kubernetes Service) |
| Azure | AKS (Azure Kubernetes Service) |
| GCP | GKE (Google Kubernetes Engine) |

```bash
# EKS
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Connect
kubectl get nodes
```

### Cluster Autoscaler
- Automatically adds/removes **nodes** based on pending pods
- Different from HPA (which scales pods, not nodes)

### Spot/Preemptible Nodes
- Cheaper cloud nodes that can be terminated anytime
- Use with tolerations and node affinity for cost savings

---

## Quick Reference Cheatsheet

```bash
# Pods
kubectl get pods -A
kubectl describe pod <name> -n <ns>
kubectl logs <name> -n <ns>
kubectl exec -it <name> -- sh

# Deployments
kubectl get deployments -A
kubectl scale deployment <name> --replicas=3
kubectl rollout undo deployment <name>

# Services
kubectl get svc -A
kubectl port-forward service/<svc> 8080:80

# Nodes
kubectl get nodes
kubectl describe node <name>
kubectl taint node <name> key=value:NoSchedule
kubectl top nodes

# Namespaces
kubectl get ns
kubectl create ns <name>

# Apply/Delete
kubectl apply -f file.yaml
kubectl delete -f file.yaml
```