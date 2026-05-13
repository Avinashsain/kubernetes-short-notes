# Kubernetes RBAC — Short Notes

---

## Core Concepts

| Resource | Purpose |
|---|---|
| `ServiceAccount` | Identity for a pod/process inside the cluster |
| `Role` | Defines **what** actions are allowed (namespace-scoped) |
| `RoleBinding` | Binds a Role to a ServiceAccount/User (namespace-scoped) |
| `ClusterRole` | Same as Role but **cluster-wide** |
| `ClusterRoleBinding` | Same as RoleBinding but **cluster-wide** |

---

## 1. ServiceAccount

Identity assigned to pods. Every namespace has a `default` SA automatically.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apache-user
  namespace: apache
```

---

## 2. Role

Defines permissions **within a namespace**. Always split rules by `apiGroups`.

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: apache-manager
  namespace: apache
rules:
- apiGroups: [""]               # core group
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

- apiGroups: ["apps"]           # apps group
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

- apiGroups: ["autoscaling"]    # autoscaling group
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

---

## 3. RoleBinding

Connects a Role to a ServiceAccount.

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: apache-manager-binding
  namespace: apache
roleRef:
  kind: Role
  name: apache-manager
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: apache-user
  namespace: apache
  apiGroup: ""                  # always "" for ServiceAccount
```

---

## 4. ClusterRole

Same as Role but applies **cluster-wide**. Use for cluster-scoped resources like nodes, namespaces, PVs — or to share permissions across all namespaces.

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-reader               # no namespace field — cluster-wide
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces", "persistentvolumes"]
  verbs: ["get", "list", "watch"]

- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

---

## 5. ClusterRoleBinding

Binds a ClusterRole to a ServiceAccount/User **cluster-wide**.

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: apache-user-node-reader   # no namespace field — cluster-wide
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: apache-user
  namespace: apache               # namespace required for ServiceAccount
  apiGroup: ""                    # always "" for ServiceAccount
```

---

## Role vs ClusterRole — When to Use What

| Scenario | Use |
|---|---|
| Grant access within a single namespace | `Role` + `RoleBinding` |
| Grant access to cluster-scoped resources (nodes, PVs) | `ClusterRole` + `ClusterRoleBinding` |
| Reuse same permissions across all namespaces | `ClusterRole` + `RoleBinding` (per namespace) |
| Grant cluster-admin level access | `ClusterRole` + `ClusterRoleBinding` |

> **Tip:** A `ClusterRole` can be bound with a `RoleBinding` to restrict it to a single namespace — useful for reusing common permission sets.

---

## `kubectl auth can-i` Commands

```bash
# Basic check
kubectl auth can-i <verb> <resource> -n <namespace> --as=<identity>

# Test as a ServiceAccount (full qualified name required)
kubectl auth can-i get pods -n apache \
  --as=system:serviceaccount:apache:apache-user

# List ALL permissions for a SA
kubectl auth can-i --list -n apache \
  --as=system:serviceaccount:apache:apache-user

# Test cluster-scoped resources (no -n flag)
kubectl auth can-i get nodes \
  --as=system:serviceaccount:apache:apache-user

# Check your own current permissions
kubectl auth whoami
kubectl auth can-i --list -n apache
```

---

## `apiGroups` Quick Reference

| Resource | `apiGroups` |
|---|---|
| `pods`, `services`, `configmaps`, `secrets` | `""` |
| `deployments`, `replicasets`, `daemonsets` | `"apps"` |
| `horizontalpodautoscalers` | `"autoscaling"` |
| `verticalpodautoscalers` | `"autoscaling.k8s.io"` |
| `jobs`, `cronjobs` | `"batch"` |
| `nodes`, `namespaces`, `persistentvolumes` | `""` (cluster-scoped) |

---

## Common Verbs

| Verb | Action |
|---|---|
| `get` | Read a single resource |
| `list` | Read all resources of a type |
| `watch` | Stream changes |
| `create` | Create new resource |
| `update` | Full replace |
| `patch` | Partial update |
| `delete` | Delete resource |

---

## Key Rules to Remember

- `ServiceAccount` `apiGroup` in subjects is always `""`
- `roleRef.apiGroup` is always `rbac.authorization.k8s.io`
- **Never mix** `""` and `"apps"` in the same `apiGroups` rule
- `Role` + `RoleBinding` = namespace only
- `ClusterRole` + `ClusterRoleBinding` = cluster-wide
- Nodes, Namespaces, PVs are **cluster-scoped** — Role cannot grant access to them