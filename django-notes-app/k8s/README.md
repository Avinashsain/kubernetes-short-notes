# Port Forward, Scale & HPA Testing — Kubernetes Short Notes

## kubectl port-forward

Forwards a local port to a pod/service — useful for testing without Ingress or LoadBalancer.

```bash
kubectl port-forward service/<service-name> -n <namespace> <local-port>:<service-port>
```

### Examples

```bash
kubectl port-forward service/nginx-service -n nginx 80:80 --address=0.0.0.0
kubectl port-forward service/notes-app-service -n notes-app 8000:8000 --address=0.0.0.0
kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 80:80 --address=0.0.0.0
kubectl port-forward service/apache-service -n apache 80:80 --address=0.0.0.0
```

### Flags

| Flag | Meaning |
|---|---|
| `--address=0.0.0.0` | Expose on all interfaces — accessible outside localhost |
| `--address=127.0.0.1` | Default — localhost only |

### Exit port-forward
```
Ctrl + C
```

---

## kubectl scale

Manually scale a deployment up or down:

```bash
kubectl scale deployment <deployment-name> -n <namespace> --replicas=<count>
```

```bash
# Scale apache to 3 replicas
kubectl scale deployment apache-deployment -n apache --replicas=3

# Scale down to 1
kubectl scale deployment apache-deployment -n apache --replicas=1
```

---

## kubectl delete pods

```bash
kubectl delete pod load-generator -n apache      # delete specific pod
kubectl delete pods -n apache                    # delete ALL pods in namespace
kubectl delete pods --all -n apache             # explicit all
```

> Pods managed by a Deployment will be recreated automatically after deletion.

---

## HPA — Horizontal Pod Autoscaler

Automatically scales pods up/down based on CPU or memory usage.

```bash
# Check HPA status
kubectl get hpa -n notes-app

# Detailed HPA info
kubectl describe hpa -n notes-app

# Watch HPA live
watch kubectl get hpa -n notes-app

# Watch pods scale live
watch kubectl get pods -n notes-app
```

### HPA Output Explained

```
NAME      REFERENCE              TARGETS       MINPODS  MAXPODS  REPLICAS
notes-hpa notes-app-deployment  cpu: 80%/50%  1        5        3
```

| Field | Meaning |
|---|---|
| `TARGETS` | `current/threshold` — if current > threshold, scale up |
| `MINPODS` | Minimum replicas always running |
| `MAXPODS` | Maximum replicas HPA can scale to |
| `REPLICAS` | Current running replicas |

---

## HPA Scale Down Cooldown

HPA scale down has a **default cooldown of 5 minutes** — pods won't scale down immediately after load stops. That's why repeated `kubectl get pods` checks are needed.

```bash
# Keep checking until replicas drop back to min
watch -n 5 kubectl get hpa -n notes-app
```

---

## Load Testing Workflow for HPA

```bash
# Terminal 1 — generate load
kubectl run -it --rm load-generator --image=busybox -n apache -- sh
/ # while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done

# Terminal 2 — watch pods scale up
watch kubectl get pods -n notes-app

# Terminal 3 — watch HPA metrics
watch kubectl get hpa -n notes-app
```

---

## Full Session Summary

| Command | Purpose |
|---|---|
| `port-forward` | Expose services locally for testing |
| `scale --replicas=3` | Manually scale apache to 3 pods |
| `delete pod load-generator` | Cleanup load generator pod |
| `delete pods -n apache` | Delete all apache pods |
| `get hpa` | Check autoscaler status |
| `watch get pods` | Monitor scaling in real time |

---

## Quick Cheatsheet

```bash
# Port forward
kubectl port-forward service/<svc> -n <ns> <local>:<remote> --address=0.0.0.0

# Scale
kubectl scale deployment <name> -n <ns> --replicas=<n>

# HPA
kubectl get hpa -n <ns>
kubectl describe hpa -n <ns>

# Watch
watch kubectl get pods -n <ns>
watch kubectl get hpa -n <ns>
```
