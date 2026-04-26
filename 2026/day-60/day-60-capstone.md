# Day 60 – Capstone: Deploy WordPress + MySQL on Kubernetes

---

## Architecture

```
┌─────────────────────────────────── namespace: capstone ───────────────────────────────────┐
│                                                                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                              SECRETS & CONFIG                                       │   │
│  │  Secret: mysql-secret                    ConfigMap: wordpress-config               │   │
│  │  • MYSQL_ROOT_PASSWORD                   • WORDPRESS_DB_HOST (mysql-0.mysql...)    │   │
│  │  • MYSQL_DATABASE=wordpress              • WORDPRESS_DB_NAME=wordpress             │   │
│  │  • MYSQL_USER=wpuser                                                               │   │
│  │  • MYSQL_PASSWORD=wp-secret-pw                                                     │   │
│  └────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────┐   ┌─────────────────────────────────────────────┐   │
│  │       MySQL (StatefulSet)        │   │       WordPress (Deployment)                  │   │
│  │                                  │   │                                               │   │
│  │  mysql-0 (primary)               │   │  wp-pod-1  wp-pod-2                          │   │
│  │  ├── /var/lib/mysql              │   │  ├── readiness probe: GET /wp-login.php      │   │
│  │  │   └── PVC: mysql-data-mysql-0 │   │  ├── liveness probe: GET /wp-login.php       │   │
│  │  ├── envFrom: mysql-secret       │   │  ├── envFrom: wordpress-config               │   │
│  │  ├── cpu req: 250m / lim: 500m   │   │  ├── env: DB_USER/PASS from mysql-secret     │   │
│  │  └── mem req: 512Mi / lim: 1Gi  │   │  ├── cpu req: 200m / lim: 500m              │   │
│  │                                  │   │  └── mem req: 256Mi / lim: 512Mi            │   │
│  │  Service: mysql (Headless)       │   │                                               │   │
│  │  clusterIP: None, port: 3306     │   │  HPA: min 2 / max 10 / cpu target: 50%      │   │
│  │  mysql-0.mysql.capstone...       │   │                                               │   │
│  │                                  │   │  Service: wordpress (NodePort :30080)        │   │
│  └─────────────────────────────────┘   └─────────────────────────────────────────────┘   │
│                                                 │                                           │
│                                                 ▼                                           │
│                                         [Browser / User]                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Task 1: Create the Namespace

```bash
kubectl create namespace capstone
kubectl config set-context --current --namespace=capstone

# Verify
kubectl config view --minify | grep namespace
# namespace: capstone
```

---

## Task 2: Deploy MySQL

### Secret

```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: capstone
type: Opaque
stringData:           # stringData accepts plaintext and base64-encodes automatically
  MYSQL_ROOT_PASSWORD: "root-secure-pw-2026"
  MYSQL_DATABASE: "wordpress"
  MYSQL_USER: "wpuser"
  MYSQL_PASSWORD: "wp-secure-pw-2026"
```

### Headless Service

```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: capstone
spec:
  clusterIP: None       # Headless — enables per-pod DNS
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

### StatefulSet

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: capstone
spec:
  serviceName: "mysql"
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        envFrom:
        - secretRef:
            name: mysql-secret
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 5
        readinessProbe:
          exec:
            command: ["mysql", "-u", "root", "-p$(MYSQL_ROOT_PASSWORD)", "-e", "SELECT 1"]
          initialDelaySeconds: 20
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml

kubectl get pods -n capstone -w
# mysql-0   0/1   Pending → ContainerCreating → Running
# (MySQL takes ~30s to initialize)

# Verify MySQL is working
kubectl exec -it mysql-0 -n capstone -- mysql -u wpuser -pwp-secure-pw-2026 -e "SHOW DATABASES;"
# +--------------------+
# | Database           |
# +--------------------+
# | information_schema |
# | wordpress          |   ← created automatically by MYSQL_DATABASE env var
# +--------------------+
```

---

## Task 3: Deploy WordPress

### ConfigMap

```yaml
# wordpress-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: capstone
data:
  WORDPRESS_DB_HOST: "mysql-0.mysql.capstone.svc.cluster.local:3306"
  WORDPRESS_DB_NAME: "wordpress"
```

`mysql-0.mysql.capstone.svc.cluster.local` follows the StatefulSet per-pod DNS pattern: `<pod>.<headless-service>.<namespace>.svc.cluster.local`. This is why a StatefulSet with a Headless Service is required for MySQL — WordPress needs to connect to a specific, stable address.

### Deployment

```yaml
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: capstone
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: wordpress-config
        env:
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /wp-login.php
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 6
        livenessProbe:
          httpGet:
            path: /wp-login.php
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 15
          failureThreshold: 3
```

```bash
kubectl apply -f wordpress-config.yaml
kubectl apply -f wordpress-deployment.yaml

kubectl get pods -n capstone
# NAME                         READY   STATUS    RESTARTS
# mysql-0                      1/1     Running   0
# wordpress-xxx-aaa            1/1     Running   0
# wordpress-xxx-bbb            1/1     Running   0
```

---

## Task 4: Expose WordPress

```yaml
# wordpress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: capstone
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f wordpress-service.yaml

# kind
kubectl port-forward svc/wordpress 8080:80 -n capstone
# Access: http://localhost:8080 → WordPress setup wizard

# Minikube
minikube service wordpress -n capstone
```

Completing the setup wizard creates the WordPress database tables in MySQL. Create a test blog post.

---

## Task 5: Self-Healing and Persistence Tests

### Test 1: WordPress pod deletion

```bash
# Delete one WordPress pod
kubectl delete pod wordpress-xxx-aaa -n capstone

# Watch Deployment recreate it
kubectl get pods -n capstone -w
# wordpress-xxx-aaa   Terminating
# wordpress-xxx-ccc   ContainerCreating   ← new pod already starting
# wordpress-xxx-ccc   1/1 Running

# Refresh browser — site still works, blog post still there
```

**Why it works:** The Deployment controller detects only 1/2 desired replicas and creates a replacement. WordPress is stateless — all data is in MySQL. The new pod connects to the same MySQL instance.

### Test 2: MySQL pod deletion

```bash
kubectl delete pod mysql-0 -n capstone

kubectl get pods -n capstone -w
# mysql-0   Terminating
# mysql-0   Pending → ContainerCreating → Running

# StatefulSet recreates mysql-0 and binds it to the same PVC (mysql-data-mysql-0)
# Wait ~30s for MySQL to initialize

# Access WordPress — blog post still there
```

**Why data survives:** `mysql-0` is recreated with the same name and reconnects to `mysql-data-mysql-0` — the same PVC with the same data. The StatefulSet guarantees identity (name) and storage persistence across restarts.

---

## Task 6: HPA for WordPress

```yaml
# wordpress-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
  namespace: capstone
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

```bash
kubectl apply -f wordpress-hpa.yaml

kubectl get hpa -n capstone
# NAME            REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
# wordpress-hpa   Deployment/wordpress   5%/50%    2         10        2

kubectl get all -n capstone
# pod/mysql-0
# pod/wordpress-xxx-aaa
# pod/wordpress-xxx-bbb
# deployment.apps/wordpress
# statefulset.apps/mysql
# service/mysql
# service/wordpress
# horizontalpodautoscaler.apps/wordpress-hpa
# persistentvolumeclaim/mysql-data-mysql-0
```

---

## Task 7 (Bonus): Helm Comparison

```bash
# Install WordPress via Helm in a separate namespace
kubectl create namespace helm-test
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install wp-helm bitnami/wordpress -n helm-test

kubectl get all -n helm-test | wc -l
# ~15 resources created automatically
# (Deployment, StatefulSet for MariaDB, Services, Secrets, PVCs, ServiceAccount, etc.)
```

| Approach | Manual YAML | Helm Chart |
|---|---|---|
| Resources created | ~8 files, explicit control | Automated, lots of defaults |
| Customization | Full — every field | Via `values.yaml` overrides |
| Understanding | You know exactly what runs | Abstracted — check `helm get manifest` |
| Repeatable | Only if you track files in git | Built-in — `helm install` is deterministic |
| Production-ready defaults | You configure everything | Probes, security context, PDBs pre-configured |

```bash
helm uninstall wp-helm -n helm-test
kubectl delete namespace helm-test
```

---

## Task 8: Clean Up and Reflect

```bash
kubectl get all -n capstone   # Final state — count all resources

# Twelve concepts in one deployment:
# 1. Namespace              (Task 1)
# 2. Secret                 (Task 2)
# 3. Headless Service       (Task 2)
# 4. StatefulSet + PVC      (Task 2, Day 55-56)
# 5. ConfigMap              (Task 3)
# 6. Deployment             (Task 3, Day 52)
# 7. Resource limits        (Tasks 2-3, Day 57)
# 8. Liveness probe         (Tasks 2-3, Day 57)
# 9. Readiness probe        (Tasks 2-3, Day 57)
# 10. NodePort Service      (Task 4, Day 53)
# 11. HPA                   (Task 6, Day 58)
# 12. Helm comparison       (Task 7, Day 59)

kubectl delete namespace capstone
# All resources inside the namespace are deleted — pods, services, deployments, HPAs, PVCs

kubectl config set-context --current --namespace=default
```

**Yes — deleting the namespace removes everything inside it**, including PVCs. This is why namespace deletion requires careful consideration in production. The PVC containing the MySQL database is gone. Always back up databases before namespace operations in production.

---

## Concept-to-Day Mapping

| Concept used | Day learned |
|---|---|
| Namespace | Day 52 |
| Secret + ConfigMap | Day 54 |
| StatefulSet + Headless Service | Day 56 |
| Persistent Volume Claim | Day 55 |
| Deployment | Day 52 |
| Resource Requests + Limits | Day 57 |
| Liveness + Readiness Probes | Day 57 |
| NodePort Service | Day 53 |
| HPA (autoscaling/v2) | Day 58 |
| Helm comparison | Day 59 |

---

## What I'd Add for Production

1. **Ingress + TLS** — Replace the NodePort with an Ingress controller (nginx-ingress or Traefik) and cert-manager for automatic HTTPS certificates. No more `:30080` ports.
2. **MySQL replication** — Add `mysql-1` as a read replica. WordPress can route reads to the replica and writes to `mysql-0`, improving performance and adding redundancy.
3. **Pod Disruption Budget (PDB)** — Ensure at least 1 WordPress pod is always running during node maintenance: `minAvailable: 1`.
4. **Network Policies** — Restrict MySQL to only accept connections from WordPress pods. Block everything else.
5. **Backup CronJob** — A Kubernetes CronJob that dumps MySQL data to S3 every night.
6. **Secrets management** — Replace Kubernetes Secrets (base64, in etcd) with HashiCorp Vault or AWS Secrets Manager + External Secrets Operator.
7. **Monitoring** — Add Prometheus + Grafana (via Helm) with dashboards for WordPress response time, MySQL query latency, and pod CPU/memory.

---

## Hardest Parts and What Clicked

**Hardest:** Getting the MySQL DNS name exactly right for `WORDPRESS_DB_HOST`. The format `mysql-0.mysql.capstone.svc.cluster.local:3306` uses all four layers — pod name, service name, namespace, cluster domain, and port. Any typo causes WordPress to fail to connect with a generic database error.

**What clicked:** The relationship between StatefulSet, Headless Service, PVC, and stable DNS all snapped together when I deleted `mysql-0` and watched it come back with the same name, same DNS, and the same data. That moment proved every concept from Days 53–56 is working together correctly.