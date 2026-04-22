# Day 51 – Kubernetes Manifests and Your First Pods

---

## The Four Required Fields of a Kubernetes Manifest

Every Kubernetes resource is defined in YAML with four mandatory top-level fields:

**`apiVersion`** — Which API group and version to use. For core resources like Pods, ConfigMaps, and Services it is `v1`. For Deployments it is `apps/v1`. The version tells the API server which schema to validate against.

**`kind`** — The resource type. Today it is `Pod`. Other kinds you will use soon: `Deployment`, `Service`, `ConfigMap`, `Secret`, `Namespace`.

**`metadata`** — The identity of the resource. `name` is required and must be unique within a namespace. `labels` are optional key-value pairs used to organize resources and connect them to selectors (e.g., a Service finding its Pods).

**`spec`** — The desired state. For a Pod, this describes which containers to run, which images to pull, which ports to expose, environment variables, volume mounts, etc. Kubernetes works continuously to make the actual state match what you describe here.

---

## Task 1: Nginx Pod

### `nginx-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl get pods -o wide

# Detailed info (events section is most useful for debugging)
kubectl describe pod nginx-pod

# Container logs
kubectl logs nginx-pod

# Shell inside the container
kubectl exec -it nginx-pod -- /bin/bash
# Inside:
curl localhost:80   # Nginx welcome page
exit
```

**What `-o wide` adds:** The node the pod landed on, its pod IP, and the nominated node/readiness gates. The pod IP is an internal cluster IP — only reachable from within the cluster.

---

## Task 2: BusyBox Pod

### `busybox-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  labels:
    app: busybox
    environment: dev
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo Hello from BusyBox && sleep 3600"]
```

```bash
kubectl apply -f busybox-pod.yaml
kubectl get pods
kubectl logs busybox-pod
# Output: Hello from BusyBox
```

**Why `command` is required here:** BusyBox is a minimal shell toolkit — it has no default long-running process. Without the `sleep 3600`, the container would execute `sh -c "echo Hello from BusyBox"`, print the message, and exit immediately. Kubernetes would see the container exit and restart it, creating a `CrashLoopBackOff`. The `sleep 3600` keeps the container alive for an hour so you can interact with it.

---

## Task 3: Third Pod (Multi-label)

### `app-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    app: my-web-app
    environment: staging
    team: backend
    version: "1.0"
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    env:
    - name: APP_ENV
      value: "staging"
```

```bash
kubectl apply -f app-pod.yaml
kubectl get pods --show-labels

# Filter by individual labels
kubectl get pods -l app=my-web-app
kubectl get pods -l environment=staging
kubectl get pods -l team=backend

# Add a label imperatively
kubectl label pod nginx-pod environment=production
kubectl get pods --show-labels

# Remove a label (trailing dash means delete)
kubectl label pod nginx-pod environment-
```

---

## Task 3: Imperative vs Declarative

### Imperative — `kubectl run`

```bash
kubectl run redis-pod --image=redis:latest
kubectl get pods
```

Creates the pod immediately using flags. No YAML file involved. Fast, but hard to version-control and reproduce exactly.

```bash
# Extract what Kubernetes created
kubectl get pod redis-pod -o yaml
```

The output includes dozens of auto-generated fields: `uid`, `resourceVersion`, `creationTimestamp`, `status`, `managedFields`. These are Kubernetes internals — you never write them by hand.

### Declarative — `kubectl apply -f`

```bash
kubectl apply -f nginx-pod.yaml
```

Defines the desired state in a file, checked into git. Kubernetes reconciles actual state to match. Re-running `kubectl apply` on an unchanged file is a no-op.

### Dry-run scaffold trick

```bash
kubectl run test-pod --image=nginx --dry-run=client -o yaml > scaffolded-pod.yaml
```

Generates valid YAML without creating anything. Use this to quickly bootstrap a manifest, then strip out the fields you don't need and customize the rest.

**Comparison — hand-written vs dry-run output:**

| Field | Hand-written | `--dry-run` output |
|---|---|---|
| `apiVersion` | ✅ present | ✅ present |
| `kind` | ✅ present | ✅ present |
| `metadata.name` | ✅ present | ✅ present |
| `metadata.creationTimestamp` | ❌ omitted | `null` (added by kubectl) |
| `spec.dnsPolicy` | ❌ omitted (uses default) | `ClusterFirst` (explicit) |
| `spec.restartPolicy` | ❌ omitted (uses default) | `Always` (explicit) |
| `status` | ❌ omitted | `{}` (empty placeholder) |

The hand-written file is shorter and cleaner. The dry-run output is more explicit about defaults.

---

## Task 4: Validate Before Applying

```bash
# Client-side validation (checks YAML structure only)
kubectl apply -f nginx-pod.yaml --dry-run=client

# Server-side validation (checks against live cluster API schema)
kubectl apply -f nginx-pod.yaml --dry-run=server
```

### What happens when `image` is missing

Edit `nginx-pod.yaml` and remove the `image` field, then run:

```bash
kubectl apply -f nginx-pod.yaml --dry-run=client
```

Error output:
```
error: error validating "nginx-pod.yaml": error validating data:
ValidationError(Pod.spec.containers[0]): missing required field "image"
in io.k8s.api.core.v1.Container
```

Kubernetes enforces the schema strictly — `image` is a required field on every container definition. The dry-run catches this before anything is created.

**`--dry-run=server` vs `--dry-run=client`:**
- `client` validates only the YAML structure against the built-in schema in `kubectl`. Works offline.
- `server` sends the manifest to the API server which applies admission controllers and webhook validations. Catches more issues but requires a live cluster.

---

## Task 5: Labels and Filtering

```bash
# List with labels
kubectl get pods --show-labels

# NAME         READY   STATUS    RESTARTS   AGE   LABELS
# nginx-pod    1/1     Running   0          5m    app=nginx
# busybox-pod  1/1     Running   0          3m    app=busybox,environment=dev
# app-pod      1/1     Running   0          1m    app=my-web-app,environment=staging,team=backend,version=1.0

# Equality-based selector
kubectl get pods -l environment=dev

# Set-based selector (in, notin, exists)
kubectl get pods -l 'environment in (dev, staging)'
kubectl get pods -l '!team'    # pods WITHOUT the team label
```

Labels are pure metadata — Kubernetes itself ignores their values. They gain power through **selectors**: Services use them to find Pods to route traffic to, Deployments use them to own Pods, `kubectl` uses them to filter output.

---

## Task 6: Clean Up

```bash
kubectl delete pod nginx-pod
kubectl delete pod busybox-pod
kubectl delete pod redis-pod

# Or via file
kubectl delete -f app-pod.yaml

kubectl get pods
# No resources found in default namespace.
```

### What happens when you delete a standalone Pod

It is gone forever. There is no controller watching it. Kubernetes does not recreate it. This is why standalone Pods are only appropriate for one-off debugging tasks. In production, every workload runs inside a **Deployment** (Day 52) which ensures a controller always maintains the desired number of running replicas.

---

## Key Commands Reference

```bash
kubectl apply -f <file>              # Create or update
kubectl get pods                     # List pods
kubectl get pods -o wide             # List with node + IP
kubectl get pods --show-labels       # List with labels
kubectl get pods -l <selector>       # Filter by label
kubectl describe pod <name>          # Full info + events
kubectl logs <name>                  # Container stdout/stderr
kubectl exec -it <name> -- /bin/sh   # Shell inside container
kubectl label pod <name> key=value   # Add label
kubectl label pod <name> key-        # Remove label
kubectl delete pod <name>            # Delete pod
kubectl delete -f <file>             # Delete by manifest
kubectl run <name> --image=<img> --dry-run=client -o yaml  # Scaffold YAML
```