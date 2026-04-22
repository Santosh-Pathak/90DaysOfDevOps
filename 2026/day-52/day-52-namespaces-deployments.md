# Day 52 – Kubernetes Namespaces and Deployments

---

## What Namespaces Are and Why You Use Them

A **namespace** is a virtual cluster inside your physical cluster. Resources inside different namespaces are isolated from each other by default — a Pod in `dev` cannot accidentally conflict with a Pod named the same thing in `staging`, and RBAC policies can grant a team access only to their namespace, not the entire cluster.

**When to use namespaces:**
- **Environment separation** — `dev`, `staging`, `production` on the same cluster
- **Team isolation** — team-a and team-b each get their own space
- **Resource quotas** — limit how much CPU/memory a namespace can consume
- **Access control** — grant a developer write access to `dev` but read-only access to `production`

**When NOT to use namespaces:** separating truly isolated workloads that should never share the cluster (use separate clusters for that).

---

## Task 1: Built-in Namespaces

```bash
kubectl get namespaces

# NAME              STATUS   AGE
# default           Active   1d
# kube-node-lease   Active   1d
# kube-public       Active   1d
# kube-system       Active   1d
```

| Namespace | Purpose |
|---|---|
| `default` | Where resources land when no namespace is specified |
| `kube-system` | Kubernetes control plane components (API server, scheduler, CoreDNS, etc.) |
| `kube-public` | Publicly readable — used for cluster-info ConfigMap |
| `kube-node-lease` | Node heartbeat Lease objects — the controller-manager uses these to detect node failures faster |

```bash
kubectl get pods -n kube-system
# Shows etcd, kube-apiserver, kube-scheduler, kube-controller-manager, coredns, kube-proxy...
```

The `kube-system` namespace is where the cluster keeps itself alive. Do not create resources here; do not delete these pods.

---

## Task 2: Create Custom Namespaces

### Imperative

```bash
kubectl create namespace dev
kubectl create namespace staging
```

### Declarative

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces
```

### Running pods in specific namespaces

```bash
kubectl run nginx-dev --image=nginx:latest -n dev
kubectl run nginx-staging --image=nginx:latest -n staging

# Only shows default namespace pods
kubectl get pods

# Shows ALL pods across all namespaces
kubectl get pods -A
# NAMESPACE   NAME             READY   STATUS    RESTARTS   AGE
# dev         nginx-dev        1/1     Running   0          30s
# staging     nginx-staging    1/1     Running   0          20s
```

`kubectl get pods` is namespace-scoped and defaults to `default`. `-A` (or `--all-namespaces`) is the escape hatch.

---

## Task 3: The Deployment Manifest

### `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.24
        ports:
        - containerPort: 80
```

### What each section does

**`apiVersion: apps/v1`** — Deployments are in the `apps` API group, not the core `v1` group. The API server uses this to find the right schema and controller.

**`spec.replicas: 3`** — The Deployment controller continuously watches the cluster. If fewer than 3 matching Pods exist, it creates more. If more than 3 exist, it terminates the extras.

**`spec.selector.matchLabels`** — This is how the Deployment *claims ownership* of Pods. Any Pod with the label `app: nginx` in this namespace is considered managed by this Deployment.

**`spec.template`** — The blueprint for creating Pods. The `metadata.labels` here MUST match `selector.matchLabels` — if they don't match, the Deployment controller can't find its own pods and Kubernetes returns a validation error.

**`spec.template.spec`** — This is identical to a standalone Pod's `spec`. Containers, images, ports, env vars, volumes — all defined here.

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments -n dev
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           30s

kubectl get pods -n dev
# NAME                               READY   STATUS    RESTARTS   AGE
# nginx-deployment-7c79c4bf97-4xk9p  1/1     Running   0          30s
# nginx-deployment-7c79c4bf97-j2mnd  1/1     Running   0          30s
# nginx-deployment-7c79c4bf97-vqt8r  1/1     Running   0          30s
```

**Column meanings:**
- `READY` — Pods currently running / desired replicas
- `UP-TO-DATE` — Pods updated to the latest template version
- `AVAILABLE` — Pods passing their readiness checks (ready to receive traffic)

---

## Task 4: Self-Healing in Action

```bash
# Note one pod name
kubectl get pods -n dev

# Delete it
kubectl delete pod nginx-deployment-7c79c4bf97-4xk9p -n dev

# Immediately watch
kubectl get pods -n dev
# The deleted pod is Terminating, a NEW pod is already Creating/Running
```

The replacement pod has a **different name** — the hash suffix is generated fresh. The Deployment controller doesn't resurrect the exact pod; it creates a brand new one from the template.

**Why this matters:** A standalone Pod (`kubectl apply -f pod.yaml`) that gets deleted stays deleted. No one recreates it. A Deployment-managed pod that gets deleted is replaced within seconds by the ReplicaSet controller. This is the fundamental reason you never run production workloads as bare Pods.

---

## Task 5: Scaling

### Imperative scaling

```bash
kubectl scale deployment nginx-deployment --replicas=5 -n dev
kubectl get pods -n dev
# 5 pods running

kubectl scale deployment nginx-deployment --replicas=2 -n dev
kubectl get pods -n dev
# 3 pods terminate (Kubernetes picks which ones), 2 remain
```

### Declarative scaling

Edit `nginx-deployment.yaml`: change `replicas: 3` to `replicas: 4`, then:

```bash
kubectl apply -f nginx-deployment.yaml
```

**When scaling down:** Kubernetes terminates pods gracefully — it sends `SIGTERM`, waits for the termination grace period (default 30 seconds), then sends `SIGKILL`. Pods are selected for termination based on criteria like: which are unscheduled, which have the fewest restarts, which were started most recently.

---

## Task 6: Rolling Updates and Rollbacks

### Trigger a rolling update

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev

# Watch it happen
kubectl rollout status deployment/nginx-deployment -n dev
# Waiting for deployment "nginx-deployment" rollout to finish:
#   1 out of 3 new replicas have been updated...
#   2 out of 3 new replicas have been updated...
#   3 out of 3 new replicas have been updated...
# deployment "nginx-deployment" successfully rolled out
```

**How rolling updates work (zero downtime):** Kubernetes creates new pods with the updated image one at a time (controlled by `maxSurge` and `maxUnavailable` settings, both default to 25%). Only after a new pod passes its readiness check does Kubernetes terminate an old one. At no point are all pods down simultaneously.

### Behind the scenes — ReplicaSets

```bash
kubectl get replicasets -n dev
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-7c79c4bf97   0         0         0       10m   (old — nginx:1.24)
# nginx-deployment-5d4f9b8c6e   3         3         3       2m    (new — nginx:1.25)
```

A Deployment never modifies a ReplicaSet — it creates a new one for each new template and scales the old one to 0. This is what enables rollbacks.

### View rollout history

```bash
kubectl rollout history deployment/nginx-deployment -n dev
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

Add a `--record` flag or set `kubernetes.io/change-cause` annotation to capture a human-readable reason in the history.

### Roll back

```bash
kubectl rollout undo deployment/nginx-deployment -n dev
kubectl rollout status deployment/nginx-deployment -n dev

# Verify the image
kubectl describe deployment nginx-deployment -n dev | grep Image
# Image: nginx:1.24
```

After rollback, Kubernetes scales the old ReplicaSet back up to 3 and scales the new one back to 0. The revision history is preserved.

---

## Task 7: Clean Up

```bash
kubectl delete deployment nginx-deployment -n dev
kubectl delete pod nginx-dev -n dev
kubectl delete pod nginx-staging -n staging
kubectl delete namespace dev staging production

kubectl get namespaces
kubectl get pods -A
```

**Deleting a namespace is destructive** — it cascades deletes to every resource inside it (Pods, Deployments, Services, ConfigMaps, Secrets). In production, namespace deletion requires careful coordination and should be gated behind RBAC.

---

## Deployment vs Standalone Pod

| | Standalone Pod | Deployment |
|---|---|---|
| Self-healing | ❌ Never recreated | ✅ Controller recreates immediately |
| Scaling | ❌ Fixed at 1 | ✅ Scale with one command |
| Rolling updates | ❌ Delete & recreate = downtime | ✅ Zero-downtime by default |
| Rollback | ❌ Not possible | ✅ `kubectl rollout undo` |
| Use in production | ❌ Never | ✅ Always |
| Use for debugging | ✅ Appropriate | ❌ Overkill |

---

## Key Commands Reference

```bash
# Namespaces
kubectl create namespace <n>
kubectl get namespaces
kubectl delete namespace <n>          # Deletes everything inside!

# Deployments
kubectl apply -f deployment.yaml
kubectl get deployments -n <ns>
kubectl describe deployment <n> -n <ns>
kubectl scale deployment <n> --replicas=N -n <ns>
kubectl set image deployment/<n> <container>=<image>:<tag> -n <ns>
kubectl rollout status deployment/<n> -n <ns>
kubectl rollout history deployment/<n> -n <ns>
kubectl rollout undo deployment/<n> -n <ns>

# ReplicaSets (managed automatically — inspect, don't touch)
kubectl get replicasets -n <ns>
```