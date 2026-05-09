# ☸️ Kubernetes (kubectl) Complete Command Reference

> All essential kubectl commands in one place.  
> **Convention:** Replace `<name>`, `<namespace>`, `<image>` with your actual values.

---

## 📖 Table of Contents

1. [Cluster & Context](#1-cluster--context)
2. [Namespaces](#2-namespaces)
3. [Pods](#3-pods)
4. [Deployments](#4-deployments)
5. [ReplicaSets](#5-replicasets)
6. [DaemonSets](#6-daemonsets)
7. [Jobs](#7-jobs)
8. [CronJobs](#8-cronjobs)
9. [Services](#9-services)
10. [PersistentVolumes & Claims](#10-persistentvolumes--claims)
11. [ConfigMaps & Secrets](#11-configmaps--secrets)
12. [Logs & Debugging](#12-logs--debugging)
13. [Resource Management](#13-resource-management)
14. [Apply & Delete](#14-apply--delete)
15. [Output Formats](#15-output-formats)

---

## 1. Cluster & Context

```bash
# View cluster info
kubectl cluster-info
kubectl cluster-info --context kind-avi-cluster

# View all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context kind-avi-cluster
kubectl config use-context minikube

# View current context
kubectl config current-context

# View nodes
kubectl get nodes
kubectl get nodes -o wide                          # with IP, OS, kernel info
kubectl get nodes --context kind-avi-cluster

# Cluster version
kubectl version
kubectl version --short
```

---

## 2. Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create namespace
kubectl create ns <namespace>
kubectl apply -f namespace.yml

# Delete namespace (deletes ALL resources inside!)
kubectl delete ns <namespace>

# Set default namespace for session
kubectl config set-context --current --namespace=nginx
```

---

## 3. Pods

### List & Inspect
```bash
kubectl get pods                                   # default namespace
kubectl get pods -n <namespace>                    # specific namespace
kubectl get pods -A                                # all namespaces
kubectl get pods -n <namespace> -o wide            # Node IP, Pod IP
kubectl get pods -n <namespace> --show-labels      # show labels
kubectl get pods -n <namespace> -w                 # watch live updates

kubectl describe pod <pod-name> -n <namespace>     # full details + events
```

### Create & Run
```bash
# Imperative (quick)
kubectl run <pod-name> --image=nginx
kubectl run <pod-name> --image=nginx -n <namespace>
kubectl run <pod-name> --image=nginx --port=80 -n <namespace>

# Declarative (recommended)
kubectl apply -f pod.yml
```

### Execute & Access
```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec -it <pod-name> -n <namespace> -- ls /app
```

### Delete
```bash
kubectl delete pod <pod-name> -n <namespace>
kubectl delete pod <pod-name> -n <namespace> --force   # immediate
kubectl delete -f pod.yml
```

---

## 4. Deployments

### List & Inspect
```bash
kubectl get deployments -n <namespace>
kubectl get deploy -n <namespace>                  # short form
kubectl get deploy -A                              # all namespaces
kubectl describe deployment <name> -n <namespace>
```

### Create & Update
```bash
kubectl apply -f deployment.yml
kubectl create deployment <name> --image=nginx -n <namespace>
```

### Scale
```bash
kubectl scale deployment/<name> -n <namespace> --replicas=5
kubectl scale deployment/<name> -n <namespace> --replicas=0   # stop all pods
```

### Rolling Update & Rollback
```bash
# Update image
kubectl set image deployment/<name> <container>=<image>:<tag> -n <namespace>
# Example:
kubectl set image deployment/nginx-deployment nginx=nginx:1.27 -n nginx

# Watch rollout progress
kubectl rollout status deployment/<name> -n <namespace>

# View rollout history
kubectl rollout history deployment/<name> -n <namespace>

# Rollback to previous version
kubectl rollout undo deployment/<name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<name> -n <namespace> --to-revision=2

# Pause / Resume rollout
kubectl rollout pause deployment/<name> -n <namespace>
kubectl rollout resume deployment/<name> -n <namespace>
```

### Delete
```bash
kubectl delete deployment <name> -n <namespace>
kubectl delete -f deployment.yml
```

---

## 5. ReplicaSets

```bash
# List
kubectl get replicasets -n <namespace>
kubectl get rs -n <namespace>                      # short form

# Inspect
kubectl describe rs <name> -n <namespace>

# Scale
kubectl scale rs/<name> -n <namespace> --replicas=4

# Delete
kubectl delete rs <name> -n <namespace>
kubectl delete -f replicasets.yml
```

---

## 6. DaemonSets

```bash
# List
kubectl get daemonsets -n <namespace>
kubectl get ds -n <namespace>                      # short form

# Inspect
kubectl describe ds <name> -n <namespace>

# Verify 1 pod per node
kubectl get pods -n <namespace> -o wide

# Delete
kubectl delete ds <name> -n <namespace>
kubectl delete -f daemonset.yml
```

---

## 7. Jobs

```bash
# List
kubectl get jobs -n <namespace>

# Inspect
kubectl describe job <name> -n <namespace>

# View job pod logs
kubectl get pods -n <namespace>                    # find the pod name
kubectl logs <pod-name> -n <namespace>

# Delete
kubectl delete job <name> -n <namespace>
kubectl delete -f job.yml
```

---

## 8. CronJobs

```bash
# List
kubectl get cronjobs -n <namespace>
kubectl get cj -n <namespace>                      # short form

# Inspect
kubectl describe cronjob <name> -n <namespace>

# See jobs triggered by cronjob
kubectl get jobs -n <namespace>

# Manually trigger a cronjob immediately
kubectl create job <job-name> --from=cronjob/<cronjob-name> -n <namespace>

# Suspend / Resume a cronjob
kubectl patch cronjob <name> -n <namespace> -p '{"spec":{"suspend": true}}'
kubectl patch cronjob <name> -n <namespace> -p '{"spec":{"suspend": false}}'

# Delete
kubectl delete cronjob <name> -n <namespace>
kubectl delete -f cronjob.yml
```

---

## 9. Services

```bash
# List
kubectl get services -n <namespace>
kubectl get svc -n <namespace>                     # short form
kubectl get svc -A                                 # all namespaces

# Inspect
kubectl describe svc <name> -n <namespace>

# Check which pods the service routes to
kubectl get endpoints -n <namespace>

# Port-forward (local testing)
kubectl port-forward service/<name> -n <namespace> 8080:80
kubectl port-forward service/<name> -n <namespace> 80:80 --address=0.0.0.0

# Delete
kubectl delete svc <name> -n <namespace>
kubectl delete -f service.yml
```

---

## 10. PersistentVolumes & Claims

### PersistentVolume (PV) — cluster-wide
```bash
kubectl get pv
kubectl describe pv <name>
kubectl delete pv <name>
kubectl delete -f persistentvolume.yml
```

### PersistentVolumeClaim (PVC) — namespace-scoped
```bash
kubectl get pvc -n <namespace>
kubectl get pvc                                    # default namespace
kubectl describe pvc <name> -n <namespace>
kubectl delete pvc <name> -n <namespace>
kubectl delete pvc/<name>
kubectl delete -f persistentvolumeclaim.yml
```

### Check Binding Status
```bash
kubectl get pv                                     # STATUS: Bound / Available
kubectl get pvc -n <namespace>                     # STATUS: Bound / Pending
```

---

## 11. ConfigMaps & Secrets

### ConfigMaps
```bash
# Create
kubectl create configmap <name> --from-literal=key=value -n <namespace>
kubectl create configmap <name> --from-file=config.properties -n <namespace>
kubectl apply -f configmap.yml

# List & Inspect
kubectl get configmaps -n <namespace>
kubectl get cm -n <namespace>                      # short form
kubectl describe cm <name> -n <namespace>

# Delete
kubectl delete cm <name> -n <namespace>
```

### Secrets
```bash
# Create
kubectl create secret generic <name> --from-literal=password=secret123 -n <namespace>
kubectl apply -f secret.yml

# List & Inspect
kubectl get secrets -n <namespace>
kubectl describe secret <name> -n <namespace>

# Decode secret value
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.password}' | base64 --decode

# Delete
kubectl delete secret <name> -n <namespace>
```

---

## 12. Logs & Debugging

```bash
# Pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -f          # follow / stream live
kubectl logs <pod-name> -n <namespace> --tail=50   # last 50 lines
kubectl logs <pod-name> -n <namespace> --previous  # crashed pod logs

# Multi-container pod — specify container
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Events (great for debugging)
kubectl get events -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Top — resource usage
kubectl top nodes
kubectl top pods -n <namespace>

# Full pod details
kubectl describe pod <pod-name> -n <namespace>
```

---

## 13. Resource Management

```bash
# View all resources in a namespace
kubectl get all -n <namespace>

# View specific resource types
kubectl get pods,svc,deploy -n <namespace>

# Edit a resource live
kubectl edit deployment <name> -n <namespace>
kubectl edit svc <name> -n <namespace>

# Label a resource
kubectl label pod <pod-name> env=prod -n <namespace>

# Annotate a resource
kubectl annotate pod <pod-name> description="my pod" -n <namespace>

# Taint a node (prevent pods scheduling)
kubectl taint nodes <node-name> key=value:NoSchedule

# Remove taint
kubectl taint nodes <node-name> key=value:NoSchedule-
```

---

## 14. Apply & Delete

```bash
# Apply (create or update)
kubectl apply -f <file>.yml
kubectl apply -f .                                 # apply all YMLs in folder
kubectl apply -f https://example.com/manifest.yml  # from URL

# Delete
kubectl delete -f <file>.yml
kubectl delete -f .                                # delete all from folder

# Dry run (preview without applying)
kubectl apply -f <file>.yml --dry-run=client
kubectl apply -f <file>.yml --dry-run=server

# Force apply (ignore validation errors)
kubectl apply -f <file>.yml --validate=false
```

---

## 15. Output Formats

```bash
# Wide output (more columns)
kubectl get pods -n <namespace> -o wide

# YAML format
kubectl get pod <name> -n <namespace> -o yaml

# JSON format
kubectl get pod <name> -n <namespace> -o json

# Custom columns
kubectl get pods -n <namespace> -o custom-columns='NAME:.metadata.name,STATUS:.status.phase'

# JSONPath (extract specific field)
kubectl get pod <name> -n <namespace> -o jsonpath='{.status.podIP}'

# Sort by field
kubectl get pods -A --sort-by='.metadata.creationTimestamp'
```

---

## 🗂️ Short Forms Reference

| Full Name | Short |
|---|---|
| `namespaces` | `ns` |
| `pods` | `po` |
| `deployments` | `deploy` |
| `replicasets` | `rs` |
| `daemonsets` | `ds` |
| `services` | `svc` |
| `configmaps` | `cm` |
| `persistentvolumes` | `pv` |
| `persistentvolumeclaims` | `pvc` |
| `cronjobs` | `cj` |
| `nodes` | `no` |

---

## ⚡ Most Used Commands (Daily)

```bash
kubectl get all -n <namespace>                     # overview of everything
kubectl get pods -A                                # all pods cluster-wide
kubectl get pods -n <namespace> -o wide            # pod details with node
kubectl describe pod <name> -n <namespace>         # debug a pod
kubectl logs <pod-name> -n <namespace> -f          # stream logs
kubectl apply -f <file>.yml                        # deploy/update resource
kubectl delete -f <file>.yml                       # remove resource
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh   # shell into pod
kubectl rollout undo deployment/<name> -n <namespace>   # rollback deploy
kubectl port-forward svc/<name> -n <namespace> 8080:80  # local testing
```
