# Kubernetes Storage

> **TL;DR:** PVCs are user-facing storage requests; PVs are the actual disks; StorageClasses dynamically provision PVs. StatefulSets give Pods stable identity + their own persistent storage.

## Volume Types — Container-Lifetime vs Pod-Lifetime vs Persistent

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - { name: cache, mountPath: /tmp }
        - { name: config, mountPath: /etc/config }
        - { name: data, mountPath: /data }
  volumes:
    - name: cache
      emptyDir:                    # destroyed when Pod dies
        sizeLimit: 1Gi
        medium: Memory             # tmpfs (RAM)
    - name: config
      configMap: { name: app-config }   # files projected from ConfigMap
    - name: data
      persistentVolumeClaim:
        claimName: app-data        # survives Pod restarts
```

| Volume | Lifetime | Use |
|--------|----------|-----|
| `emptyDir` | Pod | Scratch, caches, IPC between containers in a Pod |
| `emptyDir.medium: Memory` | Pod (RAM) | Sensitive temp data, fast scratch |
| `configMap` / `secret` | Pod (read-only) | Config & secret injection as files |
| `hostPath` | Node | Almost never — anti-pattern; ties Pod to a node |
| `persistentVolumeClaim` | Cluster (binds to PV) | DBs, uploads, anything stateful |
| `projected` | Pod | Combine multiple sources into one mount |

## The PV / PVC / StorageClass Trinity

```
StorageClass (template: "what backend, what params")
    |
    | dynamically provisions
    v
PersistentVolume (the actual disk: EBS, GCE PD, NFS share, Ceph RBD)
    |
    | bound to
    v
PersistentVolumeClaim (your app's request: 10Gi, RWO, ssd-class)
    |
    | mounted by
    v
Pod
```

### StorageClass (cluster-wide template)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete              # or Retain (keep PV/data after PVC delete)
volumeBindingMode: WaitForFirstConsumer   # wait until Pod is scheduled
allowVolumeExpansion: true
```

`WaitForFirstConsumer` is **important** — without it, the PV is provisioned in a random zone, and your Pod may end up in a different zone and unable to mount.

### PersistentVolumeClaim (request)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: app-data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

### Access Modes

| Mode | Meaning | Backends |
|------|---------|----------|
| `ReadWriteOnce` (RWO) | One node R/W | EBS, GCE PD — most block storage |
| `ReadOnlyMany` (ROX) | Many nodes RO | NFS, CephFS |
| `ReadWriteMany` (RWX) | Many nodes R/W | NFS, CephFS, EFS, Azure Files |
| `ReadWriteOncePod` (RWOP) | Exactly one Pod R/W (newer) | CSI drivers supporting it |

**Block storage (EBS, GCE PD) cannot be RWX.** If you need shared writes, you need a file storage class (EFS, NFS, CephFS, Azure Files).

## StatefulSet — Stateful Workloads Done Right

Difference from Deployment: stable network identity, stable storage, ordered operations.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka-headless        # MUST be a headless Service
  replicas: 3
  selector:
    matchLabels: { app: kafka }
  template:
    metadata:
      labels: { app: kafka }
    spec:
      containers:
        - name: kafka
          image: confluentinc/cp-kafka:7.6.0
          volumeMounts:
            - { name: data, mountPath: /var/lib/kafka }
  volumeClaimTemplates:              # PVC per Pod, auto-created
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi
---
apiVersion: v1
kind: Service
metadata: { name: kafka-headless }
spec:
  clusterIP: None
  selector: { app: kafka }
  ports: [{ port: 9092 }]
```

Result:
- Pods: `kafka-0`, `kafka-1`, `kafka-2` (predictable names)
- DNS: `kafka-0.kafka-headless.<ns>.svc.cluster.local`
- PVCs: `data-kafka-0`, `data-kafka-1`, `data-kafka-2` — each Pod owns its disk for life
- Scale up: creates `kafka-3` only after `kafka-2` is ready
- Scale down: deletes highest-ordinal first, **keeps the PVC** (so you can scale back without data loss)

**Critical gotcha:** Deleting a StatefulSet does **not** delete the PVCs (or the underlying PVs). This is intentional — protects against accidental data loss. Delete PVCs explicitly when retiring:

```bash
kubectl delete sts kafka
kubectl delete pvc -l app=kafka
```

## Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: db-snapshot-2026-05-20 }
spec:
  volumeSnapshotClassName: ebs-csi
  source:
    persistentVolumeClaimName: db-data
```

Restore from snapshot by referencing it in a new PVC:

```yaml
spec:
  dataSource:
    name: db-snapshot-2026-05-20
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 100Gi }
```

Useful for prod-to-staging restores, before-upgrade backups, point-in-time recovery.

## Expanding a PVC

If `allowVolumeExpansion: true` on the StorageClass:

```bash
kubectl edit pvc app-data
# change resources.requests.storage to a larger value
# Pod may need to be restarted depending on the CSI driver
```

Online expansion works for ext4, xfs on EBS/GCE PD. **You cannot shrink.**

## Reclaim Policies

- **Delete** (default for dynamic provisioning): PV + underlying disk deleted when PVC is deleted. Cost-efficient, easy to lose data.
- **Retain:** PV kept after PVC deletion; admin must manually clean up. Safer for prod DBs.

```yaml
reclaimPolicy: Retain
```

## Common Patterns

### Sidecar that processes shared scratch

```yaml
spec:
  containers:
    - name: app
      volumeMounts: [{ name: cache, mountPath: /shared }]
    - name: log-shipper
      volumeMounts: [{ name: cache, mountPath: /logs, readOnly: true }]
  volumes:
    - name: cache
      emptyDir: {}
```

### Projecting multiple sources

```yaml
volumes:
  - name: bundle
    projected:
      sources:
        - configMap: { name: app-config }
        - secret: { name: tls-cert }
        - downwardAPI:
            items:
              - path: namespace
                fieldRef: { fieldPath: metadata.namespace }
```

## Backup Strategy for Stateful Workloads

1. **App-aware backup** (`pg_dump`, `mongodump`) into object storage (S3/GCS). Most robust.
2. **CSI VolumeSnapshots** for point-in-time disk copies. Fast, but you still want app-level backups for cross-cluster restore.
3. **Velero** — open-source tool that backs up cluster manifests + PV snapshots together. Disaster recovery.

```bash
velero backup create nightly --include-namespaces=prod --ttl=720h
velero restore create --from-backup=nightly
```

## Interview Questions

**Q: PVC stuck in `Pending`. What do you check?**
A: (1) Is there a StorageClass? (`kubectl get sc`). (2) Does it support dynamic provisioning? (3) `kubectl describe pvc` — events tell you (e.g., "no available zone", "no matching PV"). (4) With `WaitForFirstConsumer`, it stays Pending until a Pod tries to mount it.

**Q: Deployment vs StatefulSet for a database — why StatefulSet?**
A: Stable network identity (each Pod has a DNS name persistent across restarts), stable storage (PVC follows the Pod, not random), ordered operations (you scale DBs one at a time, not all 3 at once).

**Q: Can two Pods share a PV?**
A: Only if the PV supports `ReadWriteMany` (NFS, EFS, CephFS, Azure Files). Block-storage backends (EBS, GCE PD) are RWO — only one node can mount at a time.

**Q: How do you back up a Postgres running in K8s?**
A: Two layers: (1) Logical: `pg_dump` cronjob → S3 (works across versions/clusters). (2) Physical: VolumeSnapshot of the PVC for fast restore. Use Velero to wrap both with manifests for full DR.

**Q: PV reclaim policy — Delete vs Retain?**
A: Delete (default) deletes the underlying disk when the PVC is gone — cost-safe but destructive. Retain keeps the disk; admin must clean up. Use Retain for anything you'd cry about losing.

**Q: Why does my StatefulSet still have PVCs after I deleted it?**
A: K8s intentionally orphans PVCs on StatefulSet delete to prevent data loss. Delete them manually: `kubectl delete pvc -l app=foo`.

## Common Pitfalls

- Using `hostPath` — Pod is now pinned to that node; if it dies, data is unreachable.
- Forgetting `volumeBindingMode: WaitForFirstConsumer` — PV in zone A, Pod scheduled to zone B, mount fails forever.
- Using ReadWriteOnce for a Deployment with >1 replica — only one Pod can mount; rest are stuck.
- Deleting a Deployment to "fix" a StatefulSet without cleaning up PVCs — orphan disks billed forever.
- Setting `emptyDir` with no `sizeLimit` — Pod can fill the node's disk.
- Not enabling `allowVolumeExpansion` — when you outgrow 10Gi, you're recreating + restoring.

## Related

- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [09-helm-and-kustomize.md](09-helm-and-kustomize.md)
- [24-databases-in-prod.md](24-databases-in-prod.md)
