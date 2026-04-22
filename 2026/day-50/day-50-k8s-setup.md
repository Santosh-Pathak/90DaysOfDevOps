# Day 50 – Kubernetes Architecture and Cluster Setup

---

## Task 1: The Kubernetes Story (From Memory, Then Verified)

### Why Kubernetes was created

Docker solved the "how do I package and run one container?" problem. But in production you might need 100 containers of a service, spread across 10 servers, with automatic restarts when one crashes, rolling updates that don't cause downtime, and load balancing across all running copies.

Docker alone has no answer for this. You would need to manually track which server has capacity, SSH in to restart crashed containers, and update every server one by one. That's not automation — that's just slower manual ops.

Kubernetes (K8s) solves **container orchestration**: given a description of what you want running (e.g., "5 copies of this image, always"), it makes it happen and keeps it that way — automatically scheduling, restarting, scaling, and updating containers across a fleet of machines.

### Who created it and what inspired it

Kubernetes was created by Google and open-sourced in 2014. It was directly inspired by Google's internal system called **Borg** (and its successor, Omega), which Google had been using for over a decade to run billions of containers across its global data centers. Kubernetes brought those ideas to the open-source world.

### What the name means

"Kubernetes" comes from Greek (κυβερνήτης) meaning **helmsman** or **pilot** — the person who steers a ship. The logo is a ship's helm. The abbreviation **K8s** counts the 8 letters between K and s.

---

## Task 2: Kubernetes Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    CONTROL PLANE (Master)                  │
│                                                            │
│  ┌─────────────────┐    ┌────────────────────────────┐   │
│  │   API Server     │    │           etcd              │   │
│  │  (kube-apiserver)│    │  (distributed key-value DB) │   │
│  │                  │◄──►│  stores ALL cluster state   │   │
│  │ • Front door to  │    │  replicated for HA          │   │
│  │   the cluster    │    └────────────────────────────┘   │
│  │ • Every command  │                                      │
│  │   goes through   │    ┌────────────────────────────┐   │
│  │   the API server │    │       Scheduler             │   │
│  │ • Validates and  │◄──►│  (kube-scheduler)           │   │
│  │   persists to    │    │  • Watches for unscheduled  │   │
│  │   etcd           │    │    pods                     │   │
│  └─────────────────┘    │  • Picks the best node      │   │
│           ▲              │    (CPU, memory, affinity)  │   │
│           │              └────────────────────────────┘   │
│           │              ┌────────────────────────────┐   │
│           └─────────────►│   Controller Manager        │   │
│                          │  (kube-controller-manager)  │   │
│                          │  • Runs control loops       │   │
│                          │  • ReplicaSet controller:   │   │
│                          │    "always 3 pods running"  │   │
│                          │  • Node controller          │   │
│                          │  • Deployment controller    │   │
│                          └────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
                              │  (API calls)
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   WORKER NODE 1  │  │   WORKER NODE 2  │  │   WORKER NODE 3  │
│                  │  │                  │  │                  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │  kubelet   │  │  │  │  kubelet   │  │  │  │  kubelet   │  │
│  │ • Agent    │  │  │  │            │  │  │  │            │  │
│  │ • Talks to │  │  │  │            │  │  │  │            │  │
│  │   API svr  │  │  │  │            │  │  │  │            │  │
│  │ • Ensures  │  │  │  │            │  │  │  │            │  │
│  │   pods run │  │  │  │            │  │  │  │            │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │ kube-proxy │  │  │  │ kube-proxy │  │  │  │ kube-proxy │  │
│  │ • iptables │  │  │  │            │  │  │  │            │  │
│  │   / IPVS   │  │  │  │            │  │  │  │            │  │
│  │   rules    │  │  │  │            │  │  │  │            │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │ containerd │  │  │  │ containerd │  │  │  │ containerd │  │
│  │ (runtime)  │  │  │  │            │  │  │  │            │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │  Pod  Pod  │  │  │  │  Pod  Pod  │  │  │  │    Pod     │  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### What happens when you run `kubectl apply -f pod.yaml`

1. **kubectl** reads `~/.kube/config` to find the API server endpoint and auth credentials.
2. **kubectl** sends an HTTP POST to the **API Server** with the pod manifest.
3. **API Server** authenticates the request, validates the YAML schema, and **writes the pod object to etcd** with status `Pending`.
4. The **Scheduler** is watching for pods with no `nodeName` assigned. It sees the new pending pod, calculates which node has enough CPU/memory and matches any affinity rules, and writes the chosen node name back to the pod object via the API server.
5. The **kubelet** on the chosen worker node watches for pods assigned to its node. It sees the new pod, calls **containerd** (or the configured runtime) to pull the image and start the container.
6. kubelet reports pod status back to the API server (Running, Ready, etc.), which is stored in etcd.

### What happens if the API server goes down

- **Existing workloads keep running** — pods that are already running on worker nodes continue to run. kubelet and kube-proxy don't stop existing containers.
- **No new operations are possible** — you can't create, update, or delete resources. `kubectl` commands fail. The scheduler and controller manager also lose their connection to etcd and stop making decisions.
- **etcd and worker nodes are unaffected** — they wait for the API server to come back.
- In a production cluster with multiple control plane replicas, a load balancer in front of the API servers ensures HA.

### What happens if a worker node goes down

- The **node controller** (inside controller-manager) detects that the node's heartbeat has stopped. After a timeout (~5 minutes by default), it marks the node `NotReady`.
- Pods on that node are marked for eviction. The **ReplicaSet controller** sees that the desired replica count is no longer met and schedules new pods on healthy nodes.
- If the node had `PersistentVolumeClaims`, the volumes may need to be detached before they can be reattached to new pods.

---

## Task 3: Install kubectl

```bash
# Linux (amd64)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

```
Client Version: v1.30.x
Kustomize Version: v5.x.x
```

---

## Task 4: Cluster Setup

### Tool chosen: `kind` (Kubernetes in Docker)

**Why kind over minikube:**
- kind creates clusters as Docker containers — no VM overhead, faster startup (~30 seconds vs 2+ minutes).
- It runs perfectly in CI environments (GitHub Actions, etc.) where Docker is available but VMs are not.
- Multi-node clusters are easy to create with a config file.
- Minikube is excellent too, but kind's Docker-native approach matches how I've been working throughout this challenge.

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name devops-cluster

# Output:
# Creating cluster "devops-cluster" ...
#  ✓ Ensuring node image (kindest/node:v1.30.x) 🖼
#  ✓ Preparing nodes 📦
#  ✓ Writing configuration 📜
#  ✓ Starting control-plane 🕹️
#  ✓ Installing CNI 🔌
#  ✓ Installing StorageClass 💾
# Set kubectl context to "kind-devops-cluster"
# You can now use your cluster with: kubectl cluster-info --context kind-devops-cluster
```

---

## Task 5: Explore the Cluster

### `kubectl get nodes`
```
NAME                          STATUS   ROLES           AGE   VERSION
devops-cluster-control-plane  Ready    control-plane   2m    v1.30.x
```

### `kubectl get pods -n kube-system`

| Pod Name | Component | What it does |
|---|---|---|
| `etcd-devops-cluster-control-plane` | etcd | The database — stores all cluster state (nodes, pods, configmaps, secrets). Every kubectl get/apply reads from and writes to here. |
| `kube-apiserver-devops-cluster-control-plane` | API Server | The front door — all `kubectl` commands and internal component communication goes through this REST API. |
| `kube-scheduler-devops-cluster-control-plane` | Scheduler | Watches for unscheduled pods and assigns them to the most suitable node based on resource availability and constraints. |
| `kube-controller-manager-devops-cluster-control-plane` | Controller Manager | Runs all control loops. ReplicaSet controller keeps replicas at the desired count. Node controller monitors node health. |
| `coredns-<hash>` | CoreDNS | Cluster DNS. Every Service gets a DNS name (e.g., `my-service.default.svc.cluster.local`). CoreDNS resolves these names to ClusterIPs. |
| `kindnet-<hash>` | CNI (kindnet) | Container Network Interface plugin — sets up pod networking so pods across nodes can communicate. (kind uses kindnet; other clusters use Calico, Flannel, Cilium, etc.) |
| `kube-proxy-<hash>` | kube-proxy | Implements Service networking using iptables/IPVS rules. Routes traffic to the right pod when you hit a Service IP. |

Every component from the architecture diagram (Task 2) is visible here as a running pod. This is the "Kubernetes eats its own cooking" principle — even the control plane runs as pods managed by Kubernetes.

---

## Task 6: Cluster Lifecycle & kubeconfig

```bash
# Delete cluster
kind delete cluster --name devops-cluster

# Recreate
kind create cluster --name devops-cluster

# Check current context
kubectl config current-context
# kind-devops-cluster

# List all contexts
kubectl config get-contexts
# CURRENT   NAME                    CLUSTER                 AUTHINFO
# *         kind-devops-cluster     kind-devops-cluster     kind-devops-cluster

# View full kubeconfig
kubectl config view
```

### What is kubeconfig?

`kubeconfig` is a YAML file that tells `kubectl` how to connect to one or more Kubernetes clusters. It stores:

- **Clusters** — the API server URL and the CA certificate to verify TLS
- **Users** — credentials (client certificate, token, or exec plugin) to authenticate with the API server
- **Contexts** — named combinations of a cluster + a user + a namespace. Switching contexts switches which cluster kubectl talks to.

**Location:** `~/.kube/config` by default. The `KUBECONFIG` environment variable can point to a different file or a colon-separated list of files.

`kind create cluster` automatically adds a new context to `~/.kube/config` and sets it as the current context. `kind delete cluster` removes it.

---

## Key Commands Cheat Sheet

```bash
# Cluster info
kubectl cluster-info
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <name>

# Nodes
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>

# Namespaces
kubectl get namespaces

# Pods (all namespaces)
kubectl get pods -A
kubectl get pods -n kube-system

# kind-specific
kind get clusters
kind create cluster --name <name>
kind delete cluster --name <name>
```

---

## What's Next

With the cluster running, the next steps in the Kubernetes journey are:

- **Pods** — the smallest deployable unit; run containers
- **Deployments** — manage ReplicaSets for rolling updates and rollbacks
- **Services** — expose pods via stable IPs and DNS
- **ConfigMaps & Secrets** — inject configuration without rebuilding images
- **Ingress** — HTTP routing from outside the cluster to Services
- **Helm** — the package manager for Kubernetes