# Day 56 – Kubernetes StatefulSets

---

## What StatefulSets Are and When to Use Them

A **StatefulSet** is a Kubernetes workload controller designed for applications that need **stable identity** — databases, message queues, distributed caches, anything where each instance has a role and needs to be individually addressable.

The key insight: Deployments treat pods as interchangeable (any pod = any other pod). StatefulSets treat pods as individuals — each has a fixed name, its own DNS entry, and its own persistent storage that follows it through restarts.

**Use StatefulSet for:** MySQL, PostgreSQL, MongoDB, Kafka, Zookeeper, Redis Cluster, Elasticsearch.

**Use Deployment for:** Nginx, APIs, microservices, anything where all replicas are identical and interchangeable.

---

## Task 1: The Problem with Deployments for Stateful Apps

```bash
kubectl create deployment demo --image=nginx --replicas=3
kubectl get pods

# NAME                    READY   STATUS    AGE
# demo-7d86db8dff-2kxjp   1/1     Running   10s
# demo-7d86db8dff-9qmbt   1/1     Running   10s
# demo-7d86db8dff-xr4mn   1/1     Running   10s
```

Delete one and watch the replacement:
```bash
kubectl delete pod demo-7d86db8dff-2kxjp
kubectl get pods
# demo-7d86db8dff-lw9pk   ← new pod, new name, new IP
```

**Why random names are a problem for databases:**

In a database cluster (e.g., MySQL primary + replicas), each node has a role:
- The primary accepts writes
- Each replica replicates from the primary
- Other services connect to the primary by name

If the primary pod disappears and reappears with a random new name and IP, every client, every replica, and every monitoring system breaks. There's no stable address to reconnect to. A database cluster needs predictable names like `mysql-0` (primary) and `mysql-1`, `mysql-2` (replicas) so topology is stable across restarts.

```bash
kubectl delete deployment demo
```

---

## Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random: `app-7d86-abc` | Stable, ordered: `app-0`, `app-1`, `app-2` |
| Startup order | All at once | Sequential: pod-0 → pod-1 → pod-2 |
| Scale-down order | Random | Reverse: pod-2 → pod-1 → pod-0 |
| Storage | All pods share one PVC (or none) | Each pod gets its own PVC |
| Network identity | No stable hostname | Per-pod DNS: `pod-0.svc.ns.svc.cluster.local` |
| Pod replacement | New pod, new name | New pod, **same name**, reconnects to same PVC |
| Best for | Stateless: web servers, APIs | Stateful: databases, queues, caches |

---

## Task 2: Headless Service

A normal Service has a ClusterIP — it load-balances across all matching Pods. A **Headless Service** (`clusterIP: None`) skips the load balancer entirely. Instead, DNS returns the individual IP of each Pod, and individual pod DNS names become resolvable.

StatefulSets require a Headless Service to enable per-pod DNS.

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None        # ← This makes it headless
  selector:
    app: web
  ports:
  - port: 80
    name: http
```

```bash
kubectl apply -f headless-service.yaml
kubectl get services

# NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# web          ClusterIP   None         <none>        80/TCP    5s
```

`CLUSTER-IP` shows `None` — confirmed headless. DNS queries for `web.default.svc.cluster.local` now return individual pod IPs instead of a single virtual IP.

---

## Task 3: StatefulSet Manifest

```yaml
# web-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"       # Must match the Headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:    # Each pod gets its own PVC from this template
  - metadata:
      name: web-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
```

```bash
kubectl apply -f web-statefulset.yaml

# Watch ordered creation
kubectl get pods -l app=web -w
# NAME    READY   STATUS              AGE
# web-0   0/1     ContainerCreating   2s
# web-0   1/1     Running             5s     ← web-0 Ready first
# web-1   0/1     ContainerCreating   6s
# web-1   1/1     Running             9s     ← then web-1
# web-2   0/1     ContainerCreating   10s
# web-2   1/1     Running             13s    ← then web-2
```

```bash
kubectl get pvc
# NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES
# web-data-web-0  Bound    pvc-aaa...     100Mi      RWO
# web-data-web-1  Bound    pvc-bbb...     100Mi      RWO
# web-data-web-2  Bound    pvc-ccc...     100Mi      RWO
```

**Pod names:** `web-0`, `web-1`, `web-2` — stable, predictable, ordered.

**PVC names:** `<template-name>-<statefulset-name>-<ordinal>` → `web-data-web-0`, `web-data-web-1`, `web-data-web-2`. Each pod exclusively owns its PVC. No sharing.

---

## Task 4: Stable Network Identity

Each StatefulSet pod gets a DNS name:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh

# Inside:
nslookup web-0.web.default.svc.cluster.local
# Server:    10.96.0.10
# Address 1: 10.244.0.12   ← pod IP of web-0

nslookup web-1.web.default.svc.cluster.local
# Address 1: 10.244.0.13   ← pod IP of web-1

nslookup web-2.web.default.svc.cluster.local
# Address 1: 10.244.0.14   ← pod IP of web-2
exit
```

```bash
kubectl get pods -o wide
# NAME    READY   IP
# web-0   1/1     10.244.0.12   ← matches nslookup
# web-1   1/1     10.244.0.13   ← matches
# web-2   1/1     10.244.0.14   ← matches
```

The DNS names match the pod IPs exactly. Even when `web-0` is deleted and recreated, `web-0.web.default.svc.cluster.local` resolves to the new pod's IP. The DNS name is the stable identity — the underlying IP can change.

---

## Task 5: Data Persists Across Pod Deletion

```bash
# Write unique data to each pod
kubectl exec web-0 -- sh -c "echo 'Data written by web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data written by web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data written by web-2' > /usr/share/nginx/html/index.html"

# Verify
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Data written by web-0

# Delete web-0
kubectl delete pod web-0

# StatefulSet recreates it — watch
kubectl get pods -l app=web -w
# web-0   0/1   Terminating
# web-0   0/1   Pending
# web-0   0/1   ContainerCreating
# web-0   1/1   Running

# Check data in the new web-0
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Data written by web-0   ← identical! Pod reconnected to same PVC
```

The new `web-0` pod bound to `web-data-web-0` — the same PVC as before. The PVC was never deleted, so all data survived. This is the entire point of `volumeClaimTemplates`.

---

## Task 6: Ordered Scaling

```bash
# Scale up — pods create in order
kubectl scale statefulset web --replicas=5
kubectl get pods -l app=web -w
# web-3 creates after web-2 is Ready, web-4 creates after web-3 is Ready

kubectl get pvc
# 5 PVCs: web-data-web-0 through web-data-web-4

# Scale down — pods terminate in reverse order
kubectl scale statefulset web --replicas=3
kubectl get pods -l app=web -w
# web-4 terminates first, then web-3

# Check PVCs after scale-down
kubectl get pvc
# Still 5 PVCs! web-data-web-3 and web-data-web-4 are RETAINED
```

**Why PVCs are kept on scale-down:** If Kubernetes deleted PVCs on scale-down, scaling back up would lose all data. By keeping them, scaling back to 5 replicas reconnects `web-3` to `web-data-web-3` and `web-4` to `web-data-web-4` with all data intact.

This is a safety feature. You must manually delete the extra PVCs if you truly want the storage released.

---

## Task 7: Clean Up

```bash
kubectl delete statefulset web
kubectl delete service web

# Check PVCs — they survive StatefulSet deletion
kubectl get pvc
# web-data-web-0   Bound
# web-data-web-1   Bound
# web-data-web-2   Bound
# web-data-web-3   Bound   ← all still here
# web-data-web-4   Bound
```

PVCs are NOT auto-deleted when a StatefulSet is deleted. This is intentional — accidental StatefulSet deletion should not wipe your database. Delete them manually:

```bash
kubectl delete pvc web-data-web-0 web-data-web-1 web-data-web-2 web-data-web-3 web-data-web-4
# Or: kubectl delete pvc -l app=web
```

---

## Key Commands Reference

```bash
kubectl get sts                              # List StatefulSets (short name)
kubectl get sts web -o yaml                  # Full StatefulSet definition
kubectl describe sts web
kubectl scale statefulset web --replicas=N
kubectl delete statefulset web               # Does NOT delete PVCs
kubectl delete pvc -l app=web               # Delete PVCs separately
```