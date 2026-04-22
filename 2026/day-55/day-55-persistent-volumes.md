# Day 55 – Persistent Volumes (PV) and Persistent Volume Claims (PVC)

---

## Why Containers Need Persistent Storage

Containers are ephemeral by design. When a Pod is deleted, restarted, or rescheduled to a different node, its filesystem is wiped. This is fine for stateless workloads (Nginx, APIs) but catastrophic for stateful ones:

- A database Pod loses all its data on restart
- A logging agent loses its position in the log file
- A file upload service loses every uploaded file

Kubernetes solves this with **PersistentVolumes** — storage that exists independently of any Pod and survives Pod deletion, restart, or rescheduling.

---

## Task 1: Proving the Problem — `emptyDir`

```yaml
# ephemeral-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-pod
spec:
  containers:
  - name: writer
    image: busybox:latest
    command: ["sh", "-c", "echo 'written at '$(date) >> /data/message.txt && cat /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: temp-storage
      mountPath: /data
  volumes:
  - name: temp-storage
    emptyDir: {}
```

```bash
kubectl apply -f ephemeral-pod.yaml
kubectl exec ephemeral-pod -- cat /data/message.txt
# written at Wed Apr 23 10:00:00 UTC 2026

kubectl delete pod ephemeral-pod
kubectl apply -f ephemeral-pod.yaml

kubectl exec ephemeral-pod -- cat /data/message.txt
# written at Wed Apr 23 10:05:00 UTC 2026   ← only ONE line, old data is gone
```

**`emptyDir`** creates a temporary directory on the node that lives as long as the Pod lives. When the Pod is deleted, the directory is wiped. The timestamp is different — proof that data from the previous Pod is gone.

---

## Task 2: PersistentVolume (Static Provisioning)

### `pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/k8s-pv-data
```

**Field breakdown:**

`capacity.storage: 1Gi` — how much storage this PV offers. A PVC requesting more than this will not bind to it.

`accessModes: ReadWriteOnce` — the PV can be mounted read-write by a single node at a time.

| Access Mode | Abbreviation | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by one node only |
| `ReadOnlyMany` | ROX | Read-only by many nodes simultaneously |
| `ReadWriteMany` | RWX | Read-write by many nodes simultaneously |

`persistentVolumeReclaimPolicy: Retain` — when the PVC is deleted, keep the PV and its data (status becomes `Released`). The alternative is `Delete` — automatically delete the PV and the underlying storage.

`hostPath: /tmp/k8s-pv-data` — uses a directory on the node's filesystem. Only appropriate for local development/testing — if the Pod moves to a different node, it won't find the data there.

```bash
kubectl apply -f pv.yaml
kubectl get pv

# NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      AGE
# local-pv   1Gi        RWO            Retain           Available   10s
```

`Available` means the PV exists but no PVC has claimed it yet.

---

## Task 3: PersistentVolumeClaim

### `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

The PVC requests `500Mi` — the PV offers `1Gi`. Kubernetes matches them because the PV's capacity is sufficient and the access modes overlap.

```bash
kubectl apply -f pvc.yaml

kubectl get pvc
# NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   AGE
# local-pvc   Bound    local-pv   1Gi        RWO            5s

kubectl get pv
# NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               AGE
# local-pv   1Gi        RWO            Retain           Bound    default/local-pvc   2m
```

Both show `Bound`. The `VOLUME` column in `kubectl get pvc` shows which PV was bound (`local-pv`). The `CLAIM` column in `kubectl get pv` shows which PVC claimed it (`default/local-pvc`).

**Matching rules:** Kubernetes binds a PVC to a PV if:
1. The PV's capacity is >= the PVC's request
2. The access modes overlap
3. The `storageClassName` matches (or both are empty)
4. Any `selector` on the PVC matches the PV's labels

---

## Task 4: Pod Using PVC — Data That Survives

### `pvc-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-writer
spec:
  containers:
  - name: writer
    image: busybox:latest
    command: ["sh", "-c", "echo 'Pod 1 wrote at '$(date) >> /data/message.txt && cat /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: local-pvc
```

```bash
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-writer -- cat /data/message.txt
# Pod 1 wrote at Wed Apr 23 10:00:00 UTC 2026

kubectl delete pod pvc-writer
```

Edit `pvc-pod.yaml` — change the pod name to `pvc-writer-2` and the message to `Pod 2 wrote`, then:

```bash
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-writer-2 -- cat /data/message.txt
# Pod 1 wrote at Wed Apr 23 10:00:00 UTC 2026
# Pod 2 wrote at Wed Apr 23 10:05:00 UTC 2026
```

Both lines are there. The second Pod found the data written by the first Pod. The PV outlived the Pod that wrote the data.

---

## Task 5: StorageClasses — Understanding Dynamic Provisioning

```bash
kubectl get storageclass

# NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
# standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   2d
```

```bash
kubectl describe storageclass standard
# Provisioner:           rancher.io/local-path
# Parameters:            <none>
# AllowVolumeExpansion:  <unset>
# MountOptions:          <none>
# ReclaimPolicy:         Delete
# VolumeBindingMode:     WaitForFirstConsumer
```

**Key fields:**

`Provisioner` — the plugin that creates the actual storage. On cloud clusters this is `kubernetes.io/aws-ebs`, `pd.csi.storage.gke.io`, etc. On kind/minikube it's a local provisioner.

`ReclaimPolicy: Delete` — when the PVC is deleted, the PV and the underlying storage are deleted automatically. This is the dynamic provisioning default.

`VolumeBindingMode: WaitForFirstConsumer` — the PV is not created until a Pod actually requests it. This ensures the PV is created on the same node where the Pod will run.

---

## Task 6: Dynamic Provisioning

With a StorageClass, developers only write PVCs — the StorageClass creates PVs automatically.

### `dynamic-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: standard
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Mi
```

```bash
kubectl apply -f dynamic-pvc.yaml
kubectl get pvc
# dynamic-pvc status is Pending (WaitForFirstConsumer — needs a Pod)

# Create a pod using the PVC
kubectl run dynamic-pod --image=busybox --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"dynamic-pvc"}}],"containers":[{"name":"dynamic-pod","image":"busybox","command":["sh","-c","echo dynamic >> /data/file && sleep 3600"],"volumeMounts":[{"mountPath":"/data","name":"v"}]}]}}' \
  -- sh

kubectl get pvc
# NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES
# local-pvc     Bound    local-pv                                   1Gi        RWO
# dynamic-pvc   Bound    pvc-a1b2c3d4-...                          200Mi      RWO

kubectl get pv
# NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# local-pv               1Gi        RWO            Retain           Bound
# pvc-a1b2c3d4-...       200Mi      RWO            Delete           Bound
```

Two PVs now: the manually created `local-pv` (Retain) and the automatically provisioned `pvc-a1b2c3d4-...` (Delete). The second one was created by the StorageClass the moment a Pod requested the PVC.

---

## Task 7: Clean Up and Reclaim Policy Comparison

```bash
# 1. Delete pods first
kubectl delete pod pvc-writer-2 dynamic-pod

# 2. Delete PVCs
kubectl delete pvc dynamic-pvc
kubectl delete pvc local-pvc
```

```bash
kubectl get pv
# NAME                 CAPACITY   RECLAIM POLICY   STATUS
# local-pv             1Gi        Retain           Released    ← still here!
# pvc-a1b2c3d4-...   (gone)      Delete           (deleted)   ← auto-removed
```

**`Retain` policy (local-pv):** When the PVC was deleted, the PV moved to `Released` status. The data in `/tmp/k8s-pv-data` is still there on the node. You must manually delete the PV and clean up the storage:

```bash
kubectl delete pv local-pv
# And if needed: rm -rf /tmp/k8s-pv-data  (on the node)
```

**`Delete` policy (dynamic-pvc):** When the PVC was deleted, Kubernetes automatically deleted the PV AND the underlying storage. Nothing to clean up manually.

| Reclaim Policy | When PVC is deleted | Data | Use for |
|---|---|---|---|
| `Retain` | PV becomes `Released` | Preserved | Databases where accidental deletion is catastrophic |
| `Delete` | PV and storage deleted | Gone | Ephemeral workloads, test environments |
| `Recycle` | PV wiped (`rm -rf`) then `Available` | Wiped | Deprecated — don't use |

---

## PV Lifecycle Summary

```
         kubectl apply -f pv.yaml
               │
               ▼
          [Available]  ←── no PVC bound yet
               │
               │  kubectl apply -f pvc.yaml (matching request)
               ▼
           [Bound]  ←── PVC is using this PV
               │
               │  kubectl delete pvc ...
               ▼
  ┌────────────┴────────────┐
  │ Retain policy           │ Delete policy
  ▼                         ▼
[Released]              (PV deleted)
  │
  │  manual kubectl delete pv
  ▼
(PV deleted)
```

---

## Static vs Dynamic Provisioning

| | Static | Dynamic |
|---|---|---|
| Who creates PVs? | Cluster admin manually | StorageClass + provisioner automatically |
| Developer workflow | Create PVC → hope a matching PV exists | Create PVC → PV appears automatically |
| Flexibility | Low — admin must anticipate sizes | High — PV sized to match PVC request |
| Best for | On-premises, specific hardware, special configs | Cloud environments, self-service teams |

---

## Key Commands Reference

```bash
# PV
kubectl apply -f pv.yaml
kubectl get pv
kubectl describe pv <n>
kubectl delete pv <n>

# PVC
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl describe pvc <n>
kubectl delete pvc <n>

# StorageClass
kubectl get storageclass
kubectl describe storageclass <n>

# Check binding
kubectl get pv,pvc
```