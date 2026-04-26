# Day 59 – Helm — Kubernetes Package Manager

---

## What Helm Is — The Three Core Concepts

**Helm** is the package manager for Kubernetes. Instead of writing and managing dozens of individual YAML files for every application, you use a single Helm **chart** that packages all the manifests together, makes them configurable through a values file, and handles upgrades and rollbacks as a unit.

**Chart** — a directory of files that describe a related set of Kubernetes resources. Like a `.deb` or `.rpm` package, but for Kubernetes. Contains manifest templates, default values, and metadata.

**Release** — a specific installation of a chart in your cluster. Installing the same chart twice creates two independent releases, each with its own name, resources, and configuration. Like two `nginx` packages installed with different configs.

**Repository** — a collection of charts hosted at a URL, like a package registry. Bitnami, Artifact Hub, and official project repos host thousands of charts.

---

## Task 1: Install Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm

# Verify
helm version
# version.BuildInfo{Version:"v3.15.x", ...}

helm env
# HELM_BIN=helm
# HELM_CACHE_HOME=~/.cache/helm
# HELM_CONFIG_HOME=~/.config/helm
# HELM_DATA_HOME=~/.local/share/helm
```

---

## Task 2: Add Repository and Search

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm search repo nginx
# NAME                    CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx           18.x.x          1.27.x        NGINX Open Source web server

helm search repo bitnami | wc -l
# 150+  ← Bitnami maintains a huge chart catalog
```

---

## Task 3: Install a Chart

```bash
helm install my-nginx bitnami/nginx

# Output tells you exactly what was deployed:
# NAME: my-nginx
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# NOTES: Get the application URL by running...

kubectl get all
# deployment.apps/my-nginx          → manages pods
# service/my-nginx                  → LoadBalancer exposing port 80
# replicaset.apps/my-nginx-xxx      → owned by deployment

helm list
# NAME        NAMESPACE   REVISION    STATUS      CHART           APP VERSION
# my-nginx    default     1           deployed    nginx-18.x.x    1.27.x

helm status my-nginx
# Full status + the post-install NOTES

helm get manifest my-nginx
# Prints all the Kubernetes YAML that was applied — Deployment, Service, ConfigMap, etc.
```

One `helm install` command replaced writing a Deployment, Service, and supporting resources by hand. The chart also handled sensible defaults for probes, resource requests, and labels.

---

## Task 4: Customize with Values

```bash
# See all configurable options
helm show values bitnami/nginx | head -60
# replicaCount: 1
# image:
#   registry: docker.io
#   repository: bitnami/nginx
#   tag: ...
# service:
#   type: LoadBalancer
#   port: 80
# resources: {}
# ...

# Install with inline overrides
helm install my-nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort
```

### `custom-values.yaml`

```yaml
# custom-values.yaml
replicaCount: 3

service:
  type: NodePort
  nodePorts:
    http: "30081"

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"

readinessProbe:
  initialDelaySeconds: 10
  periodSeconds: 5

livenessProbe:
  initialDelaySeconds: 30
  periodSeconds: 10
```

**What each section does:**

- `replicaCount` — sets how many nginx pods to run; overrides the chart default of 1
- `service.type: NodePort` — changes from LoadBalancer (cloud) to NodePort (local cluster accessible)
- `resources` — sets CPU/memory requests and limits for each pod (chart default is empty = BestEffort QoS)
- `readinessProbe` and `livenessProbe` — tune timing for local environments where startup might be slower

```bash
helm install my-nginx-file bitnami/nginx -f custom-values.yaml

# Verify overrides were applied
helm get values my-nginx-file
# replicaCount: 3
# service:
#   type: NodePort
#   nodePorts:
#     http: "30081"
# resources:
#   requests:
#     cpu: 100m
#     memory: 128Mi
# ...
```

**`--set` vs `-f values.yaml`:** Use `--set` for quick one-off overrides. Use `-f` for anything more than 2-3 values — it's readable, version-controllable, and reproducible.

---

## Task 5: Upgrade and Rollback

```bash
# Upgrade — scale to 5 replicas
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Check revisions
helm history my-nginx
# REVISION   STATUS      CHART           DESCRIPTION
# 1          superseded  nginx-18.x.x    Install complete
# 2          deployed    nginx-18.x.x    Upgrade complete

# Rollback to revision 1
helm rollback my-nginx 1

helm history my-nginx
# REVISION   STATUS      CHART          DESCRIPTION
# 1          superseded  nginx-18.x.x   Install complete
# 2          superseded  nginx-18.x.x   Upgrade complete
# 3          deployed    nginx-18.x.x   Rollback to 1    ← new revision created
```

Rollback creates a **new revision (3)** — it doesn't overwrite revision 2. This is intentional: every state of the release is auditable. You can see exactly what happened and when.

After rollback, verify replicas returned to 1:
```bash
kubectl get deployment my-nginx
# READY: 1/1
```

---

## Task 6: Create Your Own Chart

```bash
helm create my-app
# Creates: my-app/
#   Chart.yaml          # Chart metadata
#   values.yaml         # Default values
#   charts/             # Sub-chart dependencies
#   templates/          # Kubernetes manifest templates
#     deployment.yaml
#     service.yaml
#     hpa.yaml
#     ingress.yaml
#     serviceaccount.yaml
#     _helpers.tpl        # Named templates (reusable snippets)
#     NOTES.txt           # Post-install instructions
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my web application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### values.yaml (edited)

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

### Go Template syntax in `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}         # Named template from _helpers.tpl
  labels:
    {{- include "my-app.labels" . | nindent 4 }}  # Multiline label block, indented 4 spaces
spec:
  replicas: {{ .Values.replicaCount }}             # From values.yaml
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}                   # From Chart.yaml
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      {{- if .Values.autoscaling.enabled }}
      # Conditionally rendered block
      {{- end }}
```

**Template constructs:**
- `{{ .Values.key }}` — inserts a value from `values.yaml`
- `{{ .Chart.Name }}` — accesses Chart.yaml metadata
- `{{ .Release.Name }}` — the release name given at `helm install`
- `{{- ... -}}` — trims whitespace before/after
- `| nindent 4` — pipe to nindent for proper YAML indentation
- `{{- if .Values.feature.enabled }} ... {{- end }}` — conditional rendering

```bash
# Validate chart structure
helm lint my-app
# ==> Linting my-app/
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed

# Preview rendered YAML without installing
helm template my-release ./my-app

# Install
helm install my-release ./my-app
kubectl get pods
# 3 pods running (replicaCount: 3 from values.yaml)

# Upgrade — scale to 5
helm upgrade my-release ./my-app --set replicaCount=5
kubectl get pods
# 5 pods running
```

---

## Task 7: Clean Up

```bash
helm uninstall my-nginx
helm uninstall my-nginx-custom
helm uninstall my-nginx-file
helm uninstall my-release

# With history preserved (for auditing)
helm uninstall my-nginx --keep-history
helm history my-nginx     # Still visible even after uninstall

helm list
# Empty — zero releases

rm -rf my-app custom-values.yaml
```

---

## When to Use Helm vs Raw YAML

| Situation | Use |
|---|---|
| Deploying a well-known OSS tool (Prometheus, nginx, cert-manager) | Helm chart from public repo |
| Your own app with a few manifests | Raw YAML in git |
| Your own app shared across multiple teams/environments | Custom Helm chart |
| One-off debugging pod | `kubectl run` (imperative) |
| Production deployment with upgrade history and rollback | Helm |

---

## Key Commands Reference

```bash
helm repo add <name> <url>
helm repo update
helm search repo <keyword>
helm show values <chart>
helm install <release> <chart> [--set k=v] [-f values.yaml]
helm list
helm status <release>
helm get manifest <release>
helm get values <release>
helm upgrade <release> <chart> [--set k=v]
helm history <release>
helm rollback <release> [revision]
helm uninstall <release> [--keep-history]
helm create <chart-name>
helm lint <chart-dir>
helm template <release> <chart-dir>
```