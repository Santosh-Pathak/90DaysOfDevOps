# Day 54 – Kubernetes ConfigMaps and Secrets

---

## What ConfigMaps and Secrets Are

**ConfigMap** — A Kubernetes object that stores non-sensitive configuration as plain key-value pairs or file contents. Use it for: environment names, feature flags, ports, config file contents, URLs.

**Secret** — A Kubernetes object for sensitive data: passwords, API keys, TLS certificates, tokens. Stored as base64-encoded values in etcd. The encoding is not encryption — it's just a transport format. Real security comes from RBAC (controlling who can `get` the secret) and encryption at rest (configuring the API server to encrypt etcd).

**When to use which:**

| Use ConfigMap | Use Secret |
|---|---|
| `APP_ENV=production` | `DB_PASSWORD=s3cret` |
| `LOG_LEVEL=info` | `API_KEY=sk-...` |
| Nginx config files | TLS certificates |
| Feature flag values | Docker registry credentials |
| Port numbers | SSH private keys |

---

## Task 1: ConfigMap from Literals

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```

```bash
kubectl describe configmap app-config
# Name:         app-config
# Namespace:    default
# Data
# ====
# APP_DEBUG:  false
# APP_ENV:    production
# APP_PORT:   8080

kubectl get configmap app-config -o yaml
# apiVersion: v1
# data:
#   APP_DEBUG: "false"
#   APP_ENV: production
#   APP_PORT: "8080"
# kind: ConfigMap
# ...
```

All three key-value pairs are stored as plain text. No encoding, no encryption. Anyone with `kubectl get configmap` access can read them.

---

## Task 2: ConfigMap from a File

### Custom Nginx config

```nginx
# nginx-health.conf
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /health {
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

```bash
kubectl create configmap nginx-config --from-file=default.conf=nginx-health.conf
```

The key (`default.conf`) becomes the filename when the ConfigMap is mounted as a volume. The value is the entire file contents.

```bash
kubectl get configmap nginx-config -o yaml
# data:
#   default.conf: |
#     server {
#         listen 80;
#         location /health {
#             return 200 "healthy\n";
#         ...
```

---

## Task 3: Using ConfigMaps in Pods

### Method A: `envFrom` — inject all keys as environment variables

```yaml
# configmap-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo APP_ENV=$APP_ENV APP_PORT=$APP_PORT APP_DEBUG=$APP_DEBUG && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
  restartPolicy: Never
```

```bash
kubectl apply -f configmap-env-pod.yaml
kubectl logs configmap-env-pod
# APP_ENV=production APP_PORT=8080 APP_DEBUG=false
```

### Method B: Volume mount — inject file contents

```yaml
# configmap-volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config
```

```bash
kubectl apply -f configmap-volume-pod.yaml
kubectl exec configmap-volume-pod -- curl -s http://localhost/health
# healthy
```

The ConfigMap key `default.conf` becomes a file at `/etc/nginx/conf.d/default.conf` — exactly where Nginx looks for site configuration.

### env vs envFrom vs volume mount

| Method | ConfigMap key | Pod sees | Best for |
|---|---|---|---|
| `envFrom.configMapRef` | All keys | All as env vars | Injecting many settings at once |
| `env[].valueFrom.configMapKeyRef` | Specific key | One env var | Selective injection |
| Volume mount | File content key | File in container | Full config files (Nginx, Prometheus, etc.) |

---

## Task 4: Creating a Secret

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD='s3cureP@ssw0rd'
```

```bash
kubectl get secret db-credentials -o yaml
# apiVersion: v1
# data:
#   DB_PASSWORD: czNjdXJlUEBzc3cwcmQ=
#   DB_USER: YWRtaW4=
# kind: Secret
# type: Opaque
```

### Decoding

```bash
echo 'czNjdXJlUEBzc3cwcmQ=' | base64 --decode
# s3cureP@ssw0rd

echo 'YWRtaW4=' | base64 --decode
# admin
```

### Why base64 is encoding, not encryption

Base64 is a reversible encoding — anyone who can read the Secret YAML can decode it in seconds with a standard tool. It is not a cipher, it does not use a key, and it provides zero confidentiality.

The real security properties of Secrets come from:

1. **RBAC** — Secrets are separate objects from ConfigMaps, so you can grant a developer `get ConfigMap` permission without granting them `get Secret` permission.
2. **Reduced exposure surface** — Secret values are not shown in `kubectl describe` output (they appear as `<secret>`), reducing accidental log exposure.
3. **`tmpfs` storage on nodes** — Secret volumes are mounted in memory on worker nodes, not written to disk.
4. **Encryption at rest** — When configured, the API server encrypts Secret data in etcd using AES-CBC or AES-GCM. This requires explicit setup (EncryptionConfiguration).

---

## Task 5: Using Secrets in Pods

```yaml
# secret-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sh", "-c", "echo DB_USER=$DB_USER && ls /etc/db-credentials && cat /etc/db-credentials/DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
    volumeMounts:
    - name: db-secret-vol
      mountPath: /etc/db-credentials
      readOnly: true
  volumes:
  - name: db-secret-vol
    secret:
      secretName: db-credentials
  restartPolicy: Never
```

```bash
kubectl apply -f secret-pod.yaml
kubectl logs secret-pod
# DB_USER=admin
# DB_PASSWORD
# DB_USER
# s3cureP@ssw0rd
```

**Volume-mounted Secret values are plaintext.** Kubernetes decodes the base64 before writing to the pod's tmpfs volume. Files at `/etc/db-credentials/DB_USER` and `/etc/db-credentials/DB_PASSWORD` contain the raw plaintext values.

---

## Task 6: Live ConfigMap Update Propagation

```yaml
# live-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: live-config
data:
  message: "hello"
```

```yaml
# live-watcher-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-watcher
spec:
  containers:
  - name: watcher
    image: busybox:latest
    command: ["sh", "-c", "while true; do echo $(date): $(cat /config/message); sleep 5; done"]
    volumeMounts:
    - name: live-config-vol
      mountPath: /config
  volumes:
  - name: live-config-vol
    configMap:
      name: live-config
```

```bash
kubectl apply -f live-config.yaml
kubectl apply -f live-watcher-pod.yaml
kubectl logs -f live-watcher
# Wed Apr 23 10:00:00 UTC 2026: hello
# Wed Apr 23 10:00:05 UTC 2026: hello

# In another terminal — update the ConfigMap
kubectl patch configmap live-config --type merge -p '{"data":{"message":"world"}}'

# Back in the logs (after ~30-60 seconds for kubelet sync)
# Wed Apr 23 10:01:10 UTC 2026: world
```

### Volume mounts update; environment variables do not

| Method | Updates automatically? | Requires pod restart? |
|---|---|---|
| Volume mount | ✅ Yes, within ~60s (kubelet sync period) | No |
| Environment variable (`envFrom` / `env`) | ❌ No | Yes — must recreate pod |

This is why config files (Nginx config, Prometheus rules, feature flags) are better served via volume mounts. Static settings like `APP_ENV` that don't change at runtime can safely use environment variables.

---

## Task 7: Clean Up

```bash
kubectl delete pod configmap-env-pod configmap-volume-pod secret-pod live-watcher
kubectl delete configmap app-config nginx-config live-config
kubectl delete secret db-credentials
```

---

## Summary

| | ConfigMap | Secret |
|---|---|---|
| Data type | Plain text | base64-encoded |
| Intended for | Non-sensitive config | Passwords, keys, certs |
| Readable by `kubectl describe` | ✅ Full values shown | ❌ Values redacted |
| Encryption at rest | ❌ By default | ✅ Optional (requires config) |
| RBAC separation | ✅ Separate object | ✅ Separate object |
| Volume mount behavior | Auto-updates in ~60s | Auto-updates in ~60s |
| Env var behavior | Static at pod start | Static at pod start |

---

## Key Commands Reference

```bash
# Create
kubectl create configmap <n> --from-literal=K=V --from-file=key=file
kubectl create secret generic <n> --from-literal=K=V --from-file=key=file

# Inspect
kubectl get configmap <n> -o yaml
kubectl get secret <n> -o yaml
kubectl describe configmap <n>
kubectl describe secret <n>   # values are redacted

# Decode a secret value
kubectl get secret <n> -o jsonpath='{.data.KEY}' | base64 --decode

# Update
kubectl patch configmap <n> --type merge -p '{"data":{"key":"value"}}'

# Delete
kubectl delete configmap <n>
kubectl delete secret <n>
```