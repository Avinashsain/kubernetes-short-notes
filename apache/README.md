# Interactive Pods & watch Command — Kubernetes Short Notes

## kubectl run — Interactive Pod

```bash
kubectl run -it <pod-name> --image=busybox -n <namespace> -- sh
```

### Flags Explained

| Flag | Meaning |
|---|---|
| `run` | Creates a new pod |
| `-it` | Interactive terminal (`-i` stdin, `-t` tty) |
| `--image=busybox` | Lightweight Linux image |
| `-n <namespace>` | Target namespace |
| `-- sh` | Command to run inside container |
| `--rm` | Auto-delete pod on exit |

---

## BusyBox — Important Note

BusyBox does **not** have `bash` — only `sh`:

```bash
kubectl run -it load-generator --image=busybox -n apache -- bash  # ❌ bash not found
kubectl run -it load-generator --image=busybox -n apache -- sh    # ✅ correct
```

If no command is passed, busybox defaults to `sh` automatically.

---

## Auto-Delete Pod on Exit

```bash
kubectl run -it --rm load-generator --image=busybox -n apache -- sh
# Pod is deleted automatically when you exit
```

Without `--rm`, pod stays after exit and must be deleted manually:

```bash
kubectl delete pod load-generator -n apache
```

---

## Useful Commands Inside BusyBox

```sh
# Hit a service in a loop (load generation)
while true; do wget -q -O- http://<service>.<namespace>.svc.cluster.local; done

# Infinite CPU loop (trigger HPA)
while true; do echo "load"; done

# DNS check
nslookup apache-service.apache

# Ping service
ping apache-service.apache

# Check port
nc -zv apache-service 80

# Check env vars
env
```

---

## Exit the Pod

```sh
exit       # exits and stops the pod
Ctrl + D   # same as exit
```

---

## Security Warning

```
All commands and output will be recorded in container logs
including credentials and sensitive information
```

> Never paste secrets, passwords, or tokens inside an interactive pod — they get stored in container logs and visible via `kubectl logs`.

---

## watch Command

Runs a command repeatedly and shows live output — useful for monitoring pod status.

```bash
watch kubectl get pods -n apache          # refresh every 2s (default)
watch -n 1 kubectl get pods -n apache    # refresh every 1s
watch -n 5 kubectl get pods -n apache    # refresh every 5s
```

### Exit watch

```
Ctrl + C
```

---

## watch With Alias `k`

`watch` may not expand the `k` alias by default:

```bash
watch k get pods -n apache        # ❌ alias may not expand
watch kubectl get pods -n apache  # ✅ always works
```

### Fix — Enable Alias Expansion in watch

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias watch='watch '   # trailing space enables alias expansion
```

---

## Useful watch Combinations

```bash
watch kubectl get pods -n apache              # monitor pod status
watch kubectl get pods -n apache -o wide      # with node info
watch kubectl top pods -n apache              # live CPU/memory
watch kubectl get pods -A                     # all namespaces
watch -n 1 kubectl get hpa -n apache         # monitor HPA scaling
```

---

## Common Use Case — Load Testing for HPA

```bash
# Terminal 1 — generate load
kubectl run -it --rm load-generator --image=busybox -n apache -- sh
/ # while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done

# Terminal 2 — watch pods scale
watch kubectl get pods -n apache

# Terminal 3 — watch HPA
watch kubectl get hpa -n apache
```