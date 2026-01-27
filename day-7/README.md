# Day 7: Volumes and Persistent Storage

## Overview

Containers are ephemeral — when they restart, all data is lost. Kubernetes **Volumes** solve this by providing persistent storage that survives container restarts and can be shared between containers.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE STORAGE PROBLEM                           │
│                                                                  │
│   Without Volumes:                                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Container starts → Writes data → Container crashes     │   │
│   │                                         ↓                │   │
│   │                              Container restarts          │   │
│   │                                         ↓                │   │
│   │                              💀 DATA IS GONE!            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   With Volumes:                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Container starts → Writes to Volume → Container crashes│   │
│   │                              ↓                           │   │
│   │                     Container restarts                   │   │
│   │                              ↓                           │   │
│   │                     ✅ Data is still there!              │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Volume Types

Kubernetes supports many volume types:

| Category | Volume Types |
|----------|--------------|
| **Ephemeral** | emptyDir |
| **Local** | hostPath, local |
| **Network** | nfs, iscsi, fc |
| **Cloud** | awsElasticBlockStore, gcePersistentDisk, azureDisk |
| **Distributed** | cephfs, glusterfs |
| **Special** | configMap, secret, downwardAPI, projected |

---

## Ephemeral Volumes

### emptyDir

An **emptyDir** volume is created when a Pod is assigned to a node and exists as long as the Pod runs. Perfect for:
- Scratch space
- Sharing data between containers
- Caching

```yaml
# emptydir-example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-data-pod
spec:
  containers:
  # Container 1: Writes data
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from writer" > /data/message; sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  # Container 2: Reads data
  - name: reader
    image: busybox
    command: ['sh', '-c', 'sleep 5; cat /data/message; sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

#### emptyDir with Memory Medium

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory      # Uses RAM instead of disk
    sizeLimit: 100Mi    # Limit size
```

---

## Host Volumes

### hostPath

Mounts a file or directory from the host node's filesystem. Use with caution — creates node dependencies.

```yaml
# hostpath-example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: host-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: host-data
    hostPath:
      path: /data/web
      type: DirectoryOrCreate
```

#### hostPath Types

| Type | Behavior |
|------|----------|
| `""` | No checks |
| `DirectoryOrCreate` | Create directory if not exists |
| `Directory` | Directory must exist |
| `FileOrCreate` | Create file if not exists |
| `File` | File must exist |
| `Socket` | Unix socket must exist |

⚠️ **Warning**: hostPath is a security risk and should be avoided in production.

---

## Persistent Volumes (PV) and Persistent Volume Claims (PVC)

For production storage, Kubernetes uses a two-part abstraction:

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERSISTENT STORAGE FLOW                       │
│                                                                  │
│   ┌───────────────┐    binds    ┌───────────────┐               │
│   │     PVC       │◄───────────►│      PV       │               │
│   │  (Request)    │             │   (Storage)   │               │
│   │               │             │               │               │
│   │  "I need      │             │  "I provide   │               │
│   │   10Gi RWO"   │             │   100Gi RWO"  │               │
│   └───────┬───────┘             └───────┬───────┘               │
│           │                             │                        │
│           │ used by                     │ backed by             │
│           ▼                             ▼                        │
│   ┌───────────────┐             ┌───────────────┐               │
│   │      Pod      │             │    Storage    │               │
│   │               │             │   Backend     │               │
│   │  volumeMounts │             │  (AWS EBS,    │               │
│   │               │             │   NFS, etc)   │               │
│   └───────────────┘             └───────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

### Persistent Volume (PV)

A **PV** is a piece of storage provisioned by an administrator or dynamically by a StorageClass.

```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### Persistent Volume Claim (PVC)

A **PVC** is a request for storage by a user.

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

### Using PVC in a Pod

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| ReadWriteOnce | RWO | Single node read-write |
| ReadOnlyMany | ROX | Multiple nodes read-only |
| ReadWriteMany | RWX | Multiple nodes read-write |
| ReadWriteOncePod | RWOP | Single pod read-write (K8s 1.22+) |

---

## Reclaim Policies

What happens when PVC is deleted:

| Policy | Behavior |
|--------|----------|
| **Retain** | PV remains, data preserved, manual cleanup needed |
| **Delete** | PV and underlying storage deleted |
| **Recycle** | (Deprecated) Basic scrub and reuse |

---

## Storage Classes

**StorageClass** enables dynamic provisioning — PVs are created automatically when PVCs request them.

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Using StorageClass

```yaml
# pvc-with-storageclass.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd    # References StorageClass
```

### Common Cloud StorageClasses

#### AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
```

#### Google Cloud Persistent Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gce-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
```

#### Azure Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
```

---

## Complete Example: Database with Persistent Storage

```yaml
# mysql-persistent.yaml

# StorageClass (if not using default)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Retain
allowVolumeExpansion: true

---
# PVC for MySQL data
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: database-storage

---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
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
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-data

---
# MySQL Secret
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: "MySecurePassword123"

---
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None    # Headless service
```

---

## Volume Snapshots

Create point-in-time copies of PVCs:

```yaml
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mysql-data
```

### Restore from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  dataSource:
    name: mysql-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## Commands Reference

```bash
# Persistent Volumes
kubectl get pv
kubectl describe pv my-pv
kubectl delete pv my-pv

# Persistent Volume Claims
kubectl get pvc
kubectl describe pvc my-pvc
kubectl delete pvc my-pvc

# Storage Classes
kubectl get storageclass
kubectl get sc            # Short form
kubectl describe sc fast-ssd

# Check PV/PVC binding
kubectl get pv,pvc

# Expand PVC (if allowed)
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"30Gi"}}}}'
```

---

## Best Practices

### 1. Use StorageClasses for Dynamic Provisioning
```yaml
# Let Kubernetes create PVs automatically
storageClassName: standard
```

### 2. Choose Appropriate Access Modes
```yaml
# Single-writer databases: RWO
# Shared content: RWX (if supported)
accessModes:
  - ReadWriteOnce
```

### 3. Set Reclaim Policy Based on Data Importance
```yaml
# Production data: Retain
# Test data: Delete
reclaimPolicy: Retain
```

### 4. Request Appropriate Size
```yaml
# Don't over-provision
resources:
  requests:
    storage: 10Gi
```

### 5. Use Volume Snapshots for Backups

---

## Summary

| Concept | Description |
|---------|-------------|
| **Volume** | Storage attached to a Pod |
| **emptyDir** | Temporary storage, deleted with Pod |
| **hostPath** | Node filesystem mount (use with caution) |
| **PersistentVolume (PV)** | Cluster-wide storage resource |
| **PersistentVolumeClaim (PVC)** | Request for storage |
| **StorageClass** | Dynamic provisioning template |

### Key Takeaways

1. **Containers are ephemeral** — use volumes for persistent data
2. **emptyDir** is great for temp files and container sharing
3. **Avoid hostPath** in production
4. **Use PVC/PV** for production persistent storage
5. **StorageClasses** enable automatic provisioning
6. **Choose the right access mode** for your use case

---

## Next Steps

Learn how to organize and limit resources with **[Day 8: Namespaces and Resource Management](../day-8/README.md)**!
