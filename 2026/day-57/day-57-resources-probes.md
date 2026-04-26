# Day 57 – Resource Requests, Limits, and Probes

---

## Requests vs Limits

**Requests** — the guaranteed minimum resources a container is promised. The **scheduler** uses requests when deciding which node to place a pod on. A node will only accept a pod if it has enough unallocated capacity to satisfy all requests.

**Limits** — the maximum resources a container is allowed to consume. The **kubelet** enforces limits at runtime. Exceeding a CPU limit causes throttling. Exceeding a memory limit causes the container to be killed (OOMKilled).

```
Requests: "I need at least this much"   → used by scheduler for placement
Limits:   "I can use at most this much" → enforced by kubelet at runtime
```

| Scenario | CPU | Memory |
|---|---|---|
| Usage below limit | Normal execution | Normal execution |
| Usage at limit | Normal | Normal |
| Usage exceeds limit | **Throttled** (slowed down, not killed) | **OOMKilled** (process killed, container restarted) |

CPU is **compressible** — going over the limit just slows you down. Memory is **incompressible** — going over the limit kills the process immediately.

---

## Task 1: Resource Requests and Limits

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"       # 0.1 CPU core
        memory: "128Mi"   # 128 mebibytes
      limits:
        cpu: "250m"       # 0.25 CPU core
        memory: "256Mi"   # 256 mebibytes
```

```bash
kubectl apply -f resource-pod.yaml
kubectl describe pod resource-pod | grep -A 10 "Containers:"
# Limits:
#   cpu:     250m
#   memory:  256Mi
# Requests:
#   cpu:      100m
#   memory:   128Mi
# QoS Class: Burstable
```

### QoS Classes

| Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | requests == limits for ALL containers | Last to be evicted |
| **Burstable** | requests < limits, or only some containers have resources | Middle |
| **BestEffort** | No requests or limits set on any container | First to be evicted |

With `requests: 100m / 128Mi` and `limits: 250m / 256Mi`, the QoS class is `Burstable` — the pod is guaranteed its minimum resources but can burst up to the limit when capacity is available.

**CPU units:** `1` = 1 CPU core = `1000m` millicores. `100m` = 0.1 core = 10% of one CPU.

**Memory units:** `128Mi` = mebibytes (1 MiB = 1,048,576 bytes). `128M` = megabytes (1 MB = 1,000,000 bytes). Always use `Mi` or `Gi` for predictability.

---

## Task 2: OOMKilled — Exceeding Memory Limits

```yaml
# oom-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
  restartPolicy: Never
```

The container tries to allocate 200M but the limit is 100Mi. Kubernetes kills it instantly.

```bash
kubectl apply -f oom-pod.yaml
kubectl get pod oom-pod
# NAME      READY   STATUS      RESTARTS   AGE
# oom-pod   0/1     OOMKilled   0          3s

kubectl describe pod oom-pod | grep -A 5 "Last State:"
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137
#   Started:      ...
#   Finished:     ...
```

**Exit code 137 = 128 + 9 (SIGKILL).** The Linux kernel's OOM killer sent SIGKILL (signal 9) to the process. 128 + signal_number is the convention for killed-by-signal exit codes.

With `restartPolicy: Never`, the pod stays in `OOMKilled` state so you can inspect it. In a Deployment (default `restartPolicy: Always`), the container would restart immediately and you'd see `RESTARTS: 1, 2, 3...` climbing in `kubectl get pods`.

---

## Task 3: Pending Pod — Scheduler Cannot Place It

```yaml
# over-request-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: over-request-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: "100"       # 100 CPU cores — no node has this
        memory: "128Gi"  # 128 GiB — no single node has this
```

```bash
kubectl apply -f over-request-pod.yaml
kubectl get pod over-request-pod
# NAME               READY   STATUS    RESTARTS   AGE
# over-request-pod   0/1     Pending   0          30s

kubectl describe pod over-request-pod | grep -A 5 "Events:"
# Events:
#   Warning  FailedScheduling  default-scheduler
#   0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.
#   preemption: 0/1 nodes are available: 1 No preemption victims found
```

The scheduler is honest about exactly why placement failed. The pod stays `Pending` indefinitely — it does not crash, it just waits for a node that can satisfy its requests (which never comes in a local cluster).

```bash
kubectl delete pod over-request-pod
```

---

## Task 4: Liveness Probe

A **liveness probe** answers: "Is this container still alive and functioning?" If the probe fails repeatedly, Kubernetes restarts the container. Use it to detect deadlocks, infinite loops, or crashed processes that haven't exited.

```yaml
# liveness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: liveness
    image: busybox:latest
    command: ["sh", "-c", "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5    # Wait 5s before first probe
      periodSeconds: 5          # Probe every 5s
      failureThreshold: 3       # Kill after 3 consecutive failures
```

Timeline:
- `0s` — container starts, creates `/tmp/healthy`
- `5s` — first probe: `cat /tmp/healthy` succeeds ✅
- `30s` — container deletes `/tmp/healthy`
- `35s` — probe fails (file gone) ❌ (failure 1/3)
- `40s` — probe fails ❌ (failure 2/3)
- `45s` — probe fails ❌ (failure 3/3) → container restarted

```bash
kubectl apply -f liveness-pod.yaml
kubectl get pod liveness-pod -w
# NAME           READY   STATUS    RESTARTS   AGE
# liveness-pod   1/1     Running   0          10s
# liveness-pod   1/1     Running   1          55s    ← restarted!
# liveness-pod   1/1     Running   2          110s   ← restarted again (cycle repeats)

kubectl describe pod liveness-pod | grep -A 5 "Events:"
# Warning  Unhealthy  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
# Normal   Killing    Container liveness failed liveness probe, will be restarted
```

---

## Task 5: Readiness Probe

A **readiness probe** answers: "Is this container ready to receive traffic?" Failure removes the pod from Service endpoints but does **not** restart the container. Use it to delay traffic until the app is initialized, or to stop sending traffic when temporarily overloaded.

```yaml
# readiness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: readiness-test
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f readiness-pod.yaml
kubectl expose pod readiness-pod --port=80 --name=readiness-svc

kubectl get endpoints readiness-svc
# NAME            ENDPOINTS         AGE
# readiness-svc   10.244.0.12:80    10s   ← pod IP is listed

# Break the probe — delete the file nginx serves
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html

# Wait ~15s
kubectl get pod readiness-pod
# NAME            READY   STATUS    RESTARTS   AGE
# readiness-pod   0/1     Running   0          30s   ← 0/1 = not ready, but still Running

kubectl get endpoints readiness-svc
# NAME            ENDPOINTS   AGE
# readiness-svc   <none>      30s   ← removed from endpoints, no traffic sent
```

**Key distinction from liveness:** `RESTARTS` is still `0`. The container was NOT restarted — it's still running. It was just marked unready and removed from the Service's routing table. When the probe starts passing again, it's added back automatically.

---

## Task 6: Startup Probe

A **startup probe** gives slow-starting containers extra time to initialize. While the startup probe is running, liveness and readiness probes are disabled. Once startup succeeds, the other probes take over.

Without a startup probe, a slow-starting container gets killed by the liveness probe before it even finishes booting.

```yaml
# startup-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - name: slow-start
    image: busybox:latest
    command: ["sh", "-c", "sleep 20 && touch /tmp/started && sleep 3600"]
    startupProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 5
      failureThreshold: 12    # 5s × 12 = 60s budget for startup
    livenessProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 10
      failureThreshold: 3
```

**What `failureThreshold: 12` means for startup:** The startup probe gets `periodSeconds × failureThreshold = 5 × 12 = 60 seconds` total budget. During those 60 seconds, the liveness probe is paused. After the container creates `/tmp/started` (at ~20s), the startup probe succeeds, liveness takes over, and the pod becomes healthy.

**What would happen with `failureThreshold: 2`:** Budget = `5 × 2 = 10 seconds`. The container needs 20 seconds to start. The startup probe would fail at 10 seconds and Kubernetes would kill the container — it would never finish starting. The pod would CrashLoopBackOff continuously.

```bash
kubectl apply -f startup-pod.yaml
kubectl get pod startup-pod -w
# startup-pod   0/1   Running   0   5s    ← startup probe running, liveness paused
# startup-pod   0/1   Running   0   15s
# startup-pod   1/1   Running   0   25s   ← startup probe passed at ~20s, now 1/1 Ready
```

---

## Probe Types Summary

| Probe type | What it checks | Failure action | Use for |
|---|---|---|---|
| `exec` | Exit code of a command (0 = success) | — | File existence, custom health scripts |
| `httpGet` | HTTP status code (200-399 = success) | — | Web apps, REST APIs |
| `tcpSocket` | Can a TCP connection be established? | — | Databases, message queues |

| Probe | Failure action | Container restarted? | Use for |
|---|---|---|---|
| `livenessProbe` | Container killed and restarted | ✅ Yes | Detecting deadlocks, crashes |
| `readinessProbe` | Removed from Service endpoints | ❌ No | Traffic gating, initialization |
| `startupProbe` | Container killed if budget exceeded | ✅ Yes | Slow-starting applications |

---

## Probe Timing Fields

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15    # Wait this long before first probe
  periodSeconds: 10           # Probe every N seconds
  timeoutSeconds: 5           # Probe times out after N seconds
  successThreshold: 1         # N successes to become healthy (liveness: always 1)
  failureThreshold: 3         # N failures before action is taken
```

---

## Task 7: Clean Up

```bash
kubectl delete pod resource-pod oom-pod liveness-pod startup-pod readiness-pod
kubectl delete service readiness-svc
```

---

## Key Commands for Debugging

```bash
kubectl describe pod <n>       # Events section shows probe failures
kubectl get pod <n> -w         # Watch READY column and RESTARTS count
kubectl logs <n>               # Container stdout
kubectl logs <n> --previous    # Logs from the previous (crashed) container
kubectl top pod <n>            # Actual CPU/memory usage (needs Metrics Server)
```