# Day 10: StatefulSets and DaemonSets

## Overview

While Deployments are perfect for stateless applications, some workloads require special handling. **StatefulSets** manage stateful applications, and **DaemonSets** ensure a pod runs on every node.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKLOAD CONTROLLERS                          │
│                                                                  │
│   Deployment:                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  pod-abc123  pod-def456  pod-ghi789                     │   │
│   │  (interchangeable, random names)                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   StatefulSet:                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  mysql-0     mysql-1     mysql-2                        │   │
│   │  (ordered, stable names, persistent storage)            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   DaemonSet:                                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Node 1        Node 2        Node 3                     │   │
│   │  ┌──────┐      ┌──────┐      ┌──────┐                  │   │
│   │  │fluentd│     │fluentd│     │fluentd│                 │   │
│   │  └──────┘      └──────┘      └──────┘                  │   │
│   │  (one pod per node)                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## StatefulSets

### What is a StatefulSet?

A **StatefulSet** is used for applications that require:
- **Stable, unique network identifiers**
- **Stable, persistent storage**
- **Ordered, graceful deployment and scaling**
- **Ordered, graceful deletion and termination**

### StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| Pod names | Random (pod-abc123) | Ordered (app-0, app-1) |
| Pod identity | Interchangeable | Stable, persistent |
| Storage | Shared or none | Unique PVC per pod |
| Scaling order | Random | Ordered (0→1→2) |
| Deletion order | Random | Reverse order (2→1→0) |
| DNS | Through Service only | Individual pod DNS |

### StatefulSet Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     STATEFULSET                                  │
│                     Name: mysql                                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Headless Service (mysql)                    │   │
│   │              clusterIP: None                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│         ┌──────────────────┼──────────────────┐                 │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│   ┌───────────┐      ┌───────────┐      ┌───────────┐          │
│   │  mysql-0  │      │  mysql-1  │      │  mysql-2  │          │
│   │  (primary)│      │ (replica) │      │ (replica) │          │
│   └─────┬─────┘      └─────┬─────┘      └─────┬─────┘          │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│   ┌───────────┐      ┌───────────┐      ┌───────────┐          │
│   │  PVC-0    │      │  PVC-1    │      │  PVC-2    │          │
│   │  10Gi     │      │  10Gi     │      │  10Gi     │          │
│   └───────────┘      └───────────┘      └───────────┘          │
│                                                                  │
│   DNS Names:                                                    │
│   mysql-0.mysql.default.svc.cluster.local                       │
│   mysql-1.mysql.default.svc.cluster.local                       │
│   mysql-2.mysql.default.svc.cluster.local                       │
└─────────────────────────────────────────────────────────────────┘
```

### Basic StatefulSet Example

```yaml
# statefulset-example.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None              # Headless service required
  selector:
    app: nginx

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx           # Must match headless service
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:        # Automatic PVC creation
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

### MySQL StatefulSet Example

```yaml
# mysql-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
spec:
  ports:
  - port: 3306
  selector:
    app: mysql

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - -c
        - |
          # Generate server-id from pod ordinal index
          ordinal=$(hostname | grep -o '[0-9]*$')
          echo "[mysqld]" > /mnt/conf.d/server-id.cnf
          echo "server-id=$((100 + ordinal))" >> /mnt/conf.d/server-id.cnf
          
          # Primary (index 0) or replica
          if [ $ordinal -eq 0 ]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### Scaling StatefulSets

```bash
# Scale up (pods created in order: 0, 1, 2...)
kubectl scale statefulset web --replicas=5

# Scale down (pods deleted in reverse: 4, 3, 2...)
kubectl scale statefulset web --replicas=2
```

### Update Strategies

```yaml
spec:
  updateStrategy:
    type: RollingUpdate      # or OnDelete
    rollingUpdate:
      partition: 2           # Only update pods with index >= 2
```

| Strategy | Behavior |
|----------|----------|
| `RollingUpdate` | Update pods one at a time, in reverse order |
| `OnDelete` | Only update when pod is manually deleted |
| `partition` | Canary deployments - only update pods >= partition |

---

## DaemonSets

### What is a DaemonSet?

A **DaemonSet** ensures that all (or some) nodes run a copy of a pod. Use cases:
- **Log collection** (fluentd, filebeat)
- **Node monitoring** (node-exporter, datadog)
- **Cluster storage** (ceph, glusterd)
- **Network plugins** (calico, cilium)

### DaemonSet Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLUSTER                                    │
│                                                                  │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│   │     Node 1      │  │     Node 2      │  │     Node 3      │ │
│   │                 │  │                 │  │                 │ │
│   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│   │  │ fluentd   │  │  │  │ fluentd   │  │  │  │ fluentd   │  │ │
│   │  │ (daemon)  │  │  │  │ (daemon)  │  │  │  │ (daemon)  │  │ │
│   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│   │                 │  │                 │  │                 │ │
│   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│   │  │ App Pods  │  │  │  │ App Pods  │  │  │  │ App Pods  │  │ │
│   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│   │                 │  │                 │  │                 │ │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                  │
│   DaemonSet automatically adds pod when new node joins          │
└─────────────────────────────────────────────────────────────────┘
```

### Basic DaemonSet Example

```yaml
# daemonset-example.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
```

### Node Exporter DaemonSet

```yaml
# node-exporter-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        ports:
        - containerPort: 9100
          hostPort: 9100
        resources:
          limits:
            cpu: 200m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 50Mi
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

### Running DaemonSet on Specific Nodes

```yaml
# Use nodeSelector
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd
        
# Or node affinity
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
```

### Tolerations for System Nodes

```yaml
# Run on control plane nodes too
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
```

### DaemonSet Update Strategies

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # Update one node at a time
```

| Strategy | Behavior |
|----------|----------|
| `RollingUpdate` | Update pods gradually |
| `OnDelete` | Only update when pod is manually deleted |

---

## Comparison Summary

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPLOYMENT vs STATEFULSET vs DAEMONSET              │
│                                                                  │
│   ┌─────────────┬─────────────┬─────────────┬─────────────────┐ │
│   │   Feature   │ Deployment  │ StatefulSet │   DaemonSet     │ │
│   ├─────────────┼─────────────┼─────────────┼─────────────────┤ │
│   │ Pod Names   │ Random      │ Ordered     │ Node-based      │ │
│   │ Scaling     │ Any order   │ Sequential  │ Per node        │ │
│   │ Storage     │ Shared      │ Per-pod PVC │ HostPath        │ │
│   │ Network     │ Service     │ Headless    │ Host network    │ │
│   │ Use Case    │ Stateless   │ Databases   │ Node agents     │ │
│   └─────────────┴─────────────┴─────────────┴─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Commands Reference

```bash
# StatefulSets
kubectl get statefulsets
kubectl get sts              # Short form
kubectl describe sts mysql
kubectl scale sts mysql --replicas=5
kubectl delete sts mysql --cascade=orphan   # Keep pods

# DaemonSets
kubectl get daemonsets
kubectl get ds               # Short form
kubectl describe ds fluentd
kubectl rollout status ds/fluentd
kubectl rollout history ds/fluentd

# Check pod distribution
kubectl get pods -o wide

# View PVCs created by StatefulSet
kubectl get pvc -l app=mysql
```

---

## Best Practices

### StatefulSets

1. **Always use a Headless Service**
```yaml
spec:
  clusterIP: None
```

2. **Use volumeClaimTemplates for persistent storage**
```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
```

3. **Use partition for safe rollouts**
```yaml
updateStrategy:
  rollingUpdate:
    partition: 1
```

### DaemonSets

1. **Set resource limits**
```yaml
resources:
  limits:
    cpu: 200m
    memory: 200Mi
```

2. **Use tolerations for system nodes**
```yaml
tolerations:
- operator: Exists
```

3. **Use maxUnavailable for safe updates**
```yaml
updateStrategy:
  rollingUpdate:
    maxUnavailable: 1
```

---

## Summary

| Resource | Purpose | Examples |
|----------|---------|----------|
| **StatefulSet** | Ordered, stateful apps | Databases, Kafka, ZooKeeper |
| **DaemonSet** | One pod per node | Log collectors, monitoring agents |

### Key Takeaways

1. **StatefulSets** provide stable identity and storage
2. **DaemonSets** ensure every node runs a pod
3. **Use headless services** with StatefulSets
4. **StatefulSets scale and update in order**
5. **DaemonSets automatically add pods to new nodes**
6. **Use tolerations** to run DaemonSets on all nodes

---

## Next Steps

Secure your cluster with **[Day 11: RBAC and Security Basics](../day-11/README.md)**!
