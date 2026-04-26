# Day 58 – Metrics Server and Horizontal Pod Autoscaler (HPA)

---

## What the Metrics Server Is and Why HPA Needs It

The **Metrics Server** is a lightweight in-cluster component that collects real-time CPU and memory usage from kubelets across all nodes. It exposes this data through the Kubernetes Metrics API (`metrics.k8s.io`).

Without the Metrics Server:
- `kubectl top nodes` → error: `Metrics API not available`
- `kubectl top pods` → error
- HPA TARGETS column → `<unknown>` (HPA cannot calculate scaling decisions)

The HPA controller runs a control loop every 15 seconds. Each cycle it queries the Metrics API for actual pod CPU usage, compares it to the target percentage, and calculates how many replicas are needed. No Metrics Server = no data = no autoscaling.

**Metrics Server vs Prometheus:** The Metrics Server is for real-time, short-lived metrics used by HPA and `kubectl top`. It keeps only the current snapshot — no history. Prometheus stores metrics over time for dashboards, alerting, and long-term analysis. Both can coexist.

---

## Task 1: Install the Metrics Server

```bash
# Check if already running
kubectl get pods -n kube-system | grep metrics-server

# Minikube
minikube addons enable metrics-server

# kind / kubeadm — apply official manifest
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# On local clusters, kubelet TLS needs to be bypassed
# Edit the deployment to add the flag:
kubectl edit deployment metrics-server -n kube-system
# Under spec.containers[0].args, add:
# - --kubelet-insecure-tls

# Wait 60s for data collection, then:
kubectl top nodes
# NAME                          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# devops-cluster-control-plane  142m         7%     512Mi           25%

kubectl top pods -A
# NAMESPACE     NAME                                CPU(cores)   MEMORY(bytes)
# kube-system   coredns-xxx                         4m           12Mi
# kube-system   etcd-xxx                            22m          48Mi
# kube-system   kube-apiserver-xxx                  52m          240Mi
```

**`--kubelet-insecure-tls` is only for local development clusters.** In production, kubelets have properly signed TLS certificates and this flag is not needed.

---

## Task 2: Explore `kubectl top`

```bash
# Node resource usage
kubectl top nodes

# All pods, sorted by CPU
kubectl top pods -A --sort-by=cpu

# All pods, sorted by memory
kubectl top pods -A --sort-by=memory

# Pods in a specific namespace
kubectl top pods -n kube-system
```

**`kubectl top` vs `kubectl describe pod`:**

| Source | What it shows |
|---|---|
| `kubectl top pod` | **Actual real-time usage** — what the container is consuming right now |
| `kubectl describe pod` (Resources section) | **Configured requests and limits** — what you declared in the manifest |

A pod can have `requests.cpu: 100m` but actually use `450m` if no limit is set. `kubectl top` shows the `450m`. `kubectl describe` shows the `100m`. These are completely different numbers.

---

## Task 3: Deployment with CPU Requests

```yaml
# php-apache-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"    # HPA needs this to calculate % utilization
          limits:
            cpu: "500m"
```

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl expose deployment php-apache --port=80

kubectl top pod -l app=php-apache
# NAME                          CPU(cores)   MEMORY(bytes)
# php-apache-xxx-yyy            2m           10Mi   ← idle usage
```

**Why CPU requests are mandatory for HPA:** HPA calculates utilization as a percentage: `actualUsage / request × 100`. If `requests.cpu` is not set, the denominator is 0 — percentage is undefined. HPA shows `<unknown>` in TARGETS and makes no scaling decisions.

---

## Task 4: HPA (Imperative)

```bash
kubectl autoscale deployment php-apache \
  --cpu-percent=50 \
  --min=1 \
  --max=10

kubectl get hpa
# NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS
# php-apache   Deployment/php-apache   2%/50%          1         10        1

kubectl describe hpa php-apache
# Metrics:  ( current / target )
#   resource cpu on pods  (as a percentage of request):  2% (4m) / 50%
# Min replicas:  1
# Max replicas:  10
# Conditions:
#   ScalingActive  True   ValidMetricFound
#   AbleToScale    True   ReadyForNewScale
#   ScalingLimited True   TooFewReplicas
```

**How HPA calculates desired replicas:**

```
desiredReplicas = ceil(currentReplicas × (currentUtilization / targetUtilization))

Example at 80% actual, 50% target, 1 replica:
desiredReplicas = ceil(1 × (80 / 50)) = ceil(1.6) = 2

Example at 120% actual, 50% target, 2 replicas:
desiredReplicas = ceil(2 × (120 / 50)) = ceil(4.8) = 5
```

`TARGETS` showing `<unknown>` initially is normal — the Metrics Server needs ~30 seconds to collect a data point from the new pod.

---

## Task 5: Generate Load and Watch Autoscaling

```bash
# Start load generator in a separate terminal
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# Watch HPA react
kubectl get hpa php-apache --watch
# NAME         TARGETS     REPLICAS
# php-apache   2%/50%      1
# php-apache   87%/50%     1       ← CPU climbing
# php-apache   156%/50%    1       ← still 1 replica (HPA checks every 15s)
# php-apache   156%/50%    3       ← scales to 3
# php-apache   98%/50%     3       ← CPU stabilizing with 3 replicas
# php-apache   64%/50%     3
# php-apache   54%/50%     4       ← still above 50%, adds one more
# php-apache   48%/50%     4       ← stable at 4 replicas

# Stop load generator
kubectl delete pod load-generator

# Scale-down is SLOW (5-minute stabilization window prevents thrashing)
# Replicas gradually return to 1 over ~5-10 minutes
```

**Why scale-down is slow:** CPU spikes are common and brief (a batch job finishes, a user closes a tab). If HPA scaled down immediately after every spike, it would scale up and down continuously — killing pods that were about to receive traffic. The 5-minute stabilization window ensures scale-down only happens when load has genuinely subsided.

---

## Task 6: HPA from YAML — `autoscaling/v2`

```bash
kubectl delete hpa php-apache
```

```yaml
# php-apache-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately (no waiting)
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15               # Can double replicas every 15s
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 minutes before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60               # Remove at most 1 pod per minute
```

```bash
kubectl apply -f php-apache-hpa.yaml
kubectl describe hpa php-apache
# Behavior:
#   Scale Up:
#     Stabilization Window: 0 seconds
#     Select Policy: Max
#     Policies:
#       - Type: Percent  Value: 100  PeriodSeconds: 15
#   Scale Down:
#     Stabilization Window: 300 seconds
#     Select Policy: Max
#     Policies:
#       - Type: Pods  Value: 1  PeriodSeconds: 60
```

### `autoscaling/v1` vs `autoscaling/v2`

| Feature | `autoscaling/v1` | `autoscaling/v2` |
|---|---|---|
| CPU scaling | ✅ | ✅ |
| Memory scaling | ❌ | ✅ |
| Custom metrics | ❌ | ✅ (Prometheus, Datadog, etc.) |
| External metrics | ❌ | ✅ (queue depth, request rate) |
| `behavior` block | ❌ | ✅ (control scale speed) |
| Recommended | ❌ (legacy) | ✅ (use this) |

The `behavior` section is the main reason to use `autoscaling/v2`. It controls how fast HPA reacts — critical for cost efficiency (don't keep extra pods around too long) and reliability (don't scale down too fast during a brief lull).

---

## Task 7: Clean Up

```bash
kubectl delete hpa php-apache
kubectl delete service php-apache
kubectl delete deployment php-apache
kubectl delete pod load-generator --ignore-not-found
```

---

## Key Commands Reference

```bash
# Metrics
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# HPA
kubectl autoscale deployment <n> --cpu-percent=50 --min=1 --max=10
kubectl get hpa
kubectl describe hpa <n>
kubectl get hpa <n> --watch
kubectl delete hpa <n>
```