# Day 53 – Kubernetes Services

---

## The Problem Services Solve

Every Pod in Kubernetes gets its own IP address. This sounds convenient until you realize two things:

1. **Pod IPs are ephemeral.** When a Pod restarts, crashes and is replaced by a Deployment, or is rescheduled to a different node, it gets a brand new IP. Any client that cached the old IP is now broken.
2. **Deployments run multiple Pods.** If your service has 3 replicas, which of the 3 IPs should a client connect to? And how does traffic get distributed?

A **Service** solves both problems by providing:
- A **stable virtual IP (ClusterIP)** that never changes, even as Pods come and go
- A **stable DNS name** that resolves to that IP
- **Built-in load balancing** — traffic is distributed across all healthy Pods matching the Service's selector

```
[Client]
    │
    ▼
[Service — stable IP + DNS]
    │
    ├──► Pod 1  (10.244.0.5)
    ├──► Pod 2  (10.244.0.6)
    └──► Pod 3  (10.244.0.7)
```

If Pod 2 crashes and a new Pod 4 appears at 10.244.0.9, the Service automatically updates its routing table (Endpoints) to include Pod 4 and exclude the dead Pod 2. The client never notices.

---

## Task 1: Deploy the Application

### `app-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -o wide

# NAME                       READY   STATUS    IP
# web-app-xxx-aaa            1/1     Running   10.244.0.5
# web-app-xxx-bbb            1/1     Running   10.244.0.6
# web-app-xxx-ccc            1/1     Running   10.244.0.7
```

These three IPs are what change when Pods restart. The Service will abstract over them.

---

## Task 2: ClusterIP Service (Internal Access Only)

### `clusterip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

**Field breakdown:**
- `selector.app: web-app` — matches Pods with this label; these are the Pods that receive traffic
- `port: 80` — the port the Service itself listens on (what clients connect to)
- `targetPort: 80` — the port on each Pod to forward traffic to (these don't have to match)
- `type: ClusterIP` — the default; only reachable from inside the cluster

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services

# NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# kubernetes           ClusterIP   10.96.0.1      <none>        443/TCP   2d
# web-app-clusterip    ClusterIP   10.96.144.22   <none>        80/TCP    10s
```

The `CLUSTER-IP` (`10.96.144.22`) is stable — it won't change even if all 3 Pods restart.

### Test from inside the cluster

```bash
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
# Inside the pod:
wget -qO- http://web-app-clusterip
# Returns: Nginx welcome page HTML
exit
```

The `--rm` flag deletes the temporary pod immediately when you exit. The `--restart=Never` prevents Kubernetes from restarting it.

---

## Task 3: Kubernetes DNS for Service Discovery

Every Service gets an automatic DNS entry from CoreDNS:

```
<service-name>.<namespace>.svc.cluster.local
```

```bash
kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh
# Inside:
nslookup web-app-clusterip
# Server:    10.96.0.10         ← CoreDNS ClusterIP
# Address 1: 10.96.0.10
# Name:      web-app-clusterip.default.svc.cluster.local
# Address 1: 10.96.144.22      ← Service ClusterIP (matches kubectl get services)

# Short name (same namespace)
wget -qO- http://web-app-clusterip

# Full DNS name (works across namespaces)
wget -qO- http://web-app-clusterip.default.svc.cluster.local
exit
```

**How DNS resolution works:**
- `web-app-clusterip` → CoreDNS expands to `web-app-clusterip.default.svc.cluster.local` and returns the ClusterIP
- Traffic hits the ClusterIP, which is a virtual IP implemented by `kube-proxy` using iptables/IPVS rules on every node
- `kube-proxy` NATs the connection to one of the actual Pod IPs

**Cross-namespace access:** Use the full DNS name. A pod in namespace `frontend` reaching a service in namespace `backend` would use: `http://my-service.backend.svc.cluster.local`

---

## Task 4: NodePort Service (External Access via Node)

### `nodeport-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

- `nodePort: 30080` — opens port 30080 on every node in the cluster (valid range: 30000-32767)
- Traffic path: `<any-node-IP>:30080` → Service ClusterIP → Pod:80

```bash
kubectl apply -f nodeport-service.yaml
kubectl get services

# NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# web-app-nodeport     NodePort    10.96.201.55   <none>        80:30080/TCP   10s
```

The `80:30080/TCP` format means: the Service listens on port 80 (ClusterIP access), and port 30080 is exposed on every node.

### Accessing NodePort

```bash
# kind: get node IP
kubectl get nodes -o wide
# Then: curl <node-internal-ip>:30080

# Alternatively on kind, port-forward for local testing
kubectl port-forward service/web-app-nodeport 8080:80
# Then: curl http://localhost:8080

# Minikube
minikube service web-app-nodeport --url
```

---

## Task 5: LoadBalancer Service (Cloud External Access)

### `loadbalancer-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get services

# NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# web-app-loadbalancer   LoadBalancer   10.96.89.33    <pending>     80:31456/TCP   30s
```

**Why `<pending>` on a local cluster:** A `LoadBalancer` Service works by calling the cloud provider's API (AWS ELB, GCP Cloud Load Balancing, Azure Load Balancer) to provision a real external load balancer. On kind, minikube, or Docker Desktop, there is no cloud provider to call, so the EXTERNAL-IP stays `<pending>` indefinitely.

On a real cloud cluster, `<pending>` resolves to a public IP or hostname within ~30 seconds. That IP routes to all your nodes on the NodePort, which routes to the Service, which load-balances to Pods.

```bash
# On Minikube — simulates the cloud provider
minikube tunnel
# Then re-check: EXTERNAL-IP becomes 127.0.0.1
kubectl get services
```

---

## Task 6: Service Types Side by Side

```bash
kubectl get services -o wide

# NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        SELECTOR
# web-app-clusterip      ClusterIP      10.96.144.22   <none>        80/TCP         app=web-app
# web-app-nodeport       NodePort       10.96.201.55   <none>        80:30080/TCP   app=web-app
# web-app-loadbalancer   LoadBalancer   10.96.89.33    <pending>     80:31456/TCP   app=web-app
```

### Each type builds on the previous one

```
LoadBalancer
   └─ creates a NodePort
         └─ creates a ClusterIP
```

```bash
kubectl describe service web-app-loadbalancer

# Type:                     LoadBalancer
# IP:                       10.96.89.33        ← ClusterIP
# Port:                     <unset>  80/TCP
# NodePort:                 <unset>  31456/TCP  ← NodePort
# LoadBalancer Ingress:     <pending>            ← External IP
# Endpoints:                10.244.0.5:80,10.244.0.6:80,10.244.0.7:80
```

Yes — the LoadBalancer service also has a ClusterIP and a NodePort. You can reach it internally via the ClusterIP or externally via the NodePort, regardless of whether the cloud load balancer is provisioned.

### Endpoints — the routing table

```bash
kubectl get endpoints web-app-clusterip
# NAME                ENDPOINTS                                    AGE
# web-app-clusterip   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80   5m
```

Endpoints are automatically maintained by the Endpoints controller. When a Pod becomes unready or is deleted, its IP is removed. When a new Pod passes its readiness check, its IP is added. The Service's selector drives this continuously.

| Service Type | Reachable From | EXTERNAL-IP | Typical Use |
|---|---|---|---|
| ClusterIP | Inside cluster only | `<none>` | Microservice-to-microservice traffic |
| NodePort | Outside via `<NodeIP>:<nodePort>` | `<none>` | Dev/test, bare-metal, simple setups |
| LoadBalancer | Outside via cloud LB IP | Assigned by cloud | Production in AWS/GCP/Azure |

---

## Task 7: Clean Up

```bash
kubectl delete -f app-deployment.yaml
kubectl delete -f clusterip-service.yaml
kubectl delete -f nodeport-service.yaml
kubectl delete -f loadbalancer-service.yaml

kubectl get pods
# No resources found in default namespace.

kubectl get services
# NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   2d
```

Only the built-in `kubernetes` Service (the API server endpoint) remains.

---

## Key Commands Reference

```bash
kubectl apply -f service.yaml
kubectl get services
kubectl get services -o wide
kubectl describe service <name>
kubectl get endpoints <service-name>
kubectl run test --image=busybox --rm -it --restart=Never -- sh
kubectl port-forward service/<name> <local-port>:<service-port>
```