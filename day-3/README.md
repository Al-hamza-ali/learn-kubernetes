# Day 3: Pods, ReplicaSets, and Deployments

## Overview

In Kubernetes, workloads are managed through a hierarchy of resources. Understanding this hierarchy is crucial for deploying and managing applications effectively.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKLOAD HIERARCHY                           │
│                                                                 │
│                      ┌──────────────┐                           │
│                      │  Deployment  │  ◄── You define this      │
│                      └──────┬───────┘                           │
│                             │ manages                           │
│                             ▼                                   │
│                      ┌──────────────┐                           │
│                      │  ReplicaSet  │  ◄── K8s creates this     │
│                      └──────┬───────┘                           │
│                             │ manages                           │
│                             ▼                                   │
│              ┌──────────────┼──────────────┐                    │
│              │              │              │                    │
│         ┌────┴────┐    ┌────┴────┐    ┌────┴────┐               │
│         │   Pod   │    │   Pod   │    │   Pod   │               │
│         └────┬────┘    └────┬────┘    └────┬────┘               │
│              │              │              │                    │
│         ┌────┴────┐    ┌────┴────┐    ┌────┴────┐               │
│         │Container│    │Container│    │Container│               │
│         └─────────┘    └─────────┘    └─────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Pods

### What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster.

Key characteristics:
- A Pod can contain **one or more containers**
- Containers in a Pod share the same **network namespace** (same IP address)
- Containers in a Pod share the same **storage volumes**
- Pods are **ephemeral** - they can be created, destroyed, and replaced

### Pod Anatomy

```
┌──────────────────────────────────────────────────────────────────┐
│                            POD                                   │
│                        IP: 10.1.0.5                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Shared Network Namespace                 │ │
│  │                                                             │ │
│  │  ┌─────────────────┐         ┌─────────────────┐            │ │
│  │  │   Container 1   │         │   Container 2   │            │ │
│  │  │   (main app)    │ ◄─────► │   (sidecar)     │            │ │
│  │  │                 │localhost│                 │            │ │
│  │  │   Port: 8080    │         │   Port: 9090    │            │ │
│  │  └────────┬────────┘         └────────┬────────┘            │ │
│  │           │                           │                     │ │
│  └───────────┼───────────────────────────┼─────────────────────┘ │
│              │                           │                       │
│  ┌───────────┴───────────────────────────┴──────────────────────┐│
│  │                    Shared Volumes                            ││
│  │  ┌─────────────────┐         ┌─────────────────┐             ││
│  │  │   Volume 1      │         │   Volume 2      │             ││
│  │  │   (logs)        │         │   (config)      │             ││
│  │  └─────────────────┘         └─────────────────┘             ││
│  └──────────────────────────────────────────────────────────────┘│
└───────────────────────────────────────────────────────────────-──┘
```

### Pod Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      POD LIFECYCLE                              │
│                                                                 │
│   Pending ────► Running ────► Succeeded                         │
│      │             │              │                             │
│      │             │              └────► Completed (Job)        │
│      │             │                                            │
│      │             └────► Failed                                │
│      │                                                          │
│      └────► Failed (ImagePullBackOff, etc.)                     │
│                                                                 │
│   States:                                                       │
│   • Pending: Accepted but not yet running                       │
│   • Running: At least one container is running                  │
│   • Succeeded: All containers terminated successfully           │
│   • Failed: All containers terminated, at least one failed      │
│   • Unknown: State cannot be determined                         │
└─────────────────────────────────────────────────────────────────┘
```

### Example 1: Simple Pod

```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: development
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

#### Commands to manage this Pod:

```bash
# Create the pod
kubectl apply -f simple-pod.yaml

# View pod status
kubectl get pods

# View detailed information
kubectl describe pod nginx-pod

# View pod logs
kubectl logs nginx-pod

# Execute command inside pod
kubectl exec -it nginx-pod -- /bin/bash

# Delete the pod
kubectl delete pod nginx-pod
```

### Example 2: Multi-Container Pod (Sidecar Pattern)

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
  labels:
    app: web
spec:
  containers:
  # Main application container
  - name: web-app
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  # Sidecar container for log shipping
  - name: log-shipper
    image: busybox:1.36
    command: ['sh', '-c', 'tail -F /var/log/nginx/access.log']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
      readOnly: true

  volumes:
  - name: shared-logs
    emptyDir: {}
```

### Example 3: Pod with Resource Limits

```yaml
# pod-with-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"      # Minimum memory guaranteed
        cpu: "250m"         # 250 millicores (0.25 CPU)
      limits:
        memory: "128Mi"     # Maximum memory allowed
        cpu: "500m"         # Maximum CPU allowed
    ports:
    - containerPort: 80
```

### Example 4: Pod with Health Checks

```yaml
# pod-with-probes.yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-check-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    
    # Startup probe: Check if app has started
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    
    # Liveness probe: Is the container alive?
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    # Readiness probe: Is container ready for traffic?
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 3
```

### Example 5: Pod with Environment Variables and ConfigMap

```yaml
# pod-with-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    # Direct value
    - name: APP_ENV
      value: "production"
    
    # From ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    
    # From Secret
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db_password
    
    # Pod information
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

### When to Use Naked Pods?

| Scenario | Recommendation |
|----------|----------------|
| Testing/debugging | ✅ Use naked pods |
| One-off tasks | ✅ Consider Jobs instead |
| Production workloads | ❌ Use Deployments |
| Stateful applications | ❌ Use StatefulSets |
| Node-level daemons | ❌ Use DaemonSets |

---

## 2. ReplicaSets

### What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of pod replicas are running at any given time. It's the mechanism that provides self-healing and scaling capabilities.

Key characteristics:
- Maintains a **stable set of replica Pods**
- Guarantees **availability** of a specified number of identical Pods
- Uses **label selectors** to identify Pods it manages
- Creates/deletes Pods to match the desired count

### ReplicaSet Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        REPLICASET                                │
│                    Name: nginx-rs                                │
│                    Replicas: 3                                   │
│                                                                  │
│    Selector: app=nginx                                          │
│         │                                                        │
│         │ watches & manages pods with matching labels           │
│         │                                                        │
│         ▼                                                        │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │                    MANAGED PODS                          │  │
│    │                                                          │  │
│    │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │  │
│    │   │   Pod 1     │  │   Pod 2     │  │   Pod 3     │     │  │
│    │   │ app=nginx   │  │ app=nginx   │  │ app=nginx   │     │  │
│    │   │ Running ✓   │  │ Running ✓   │  │ Running ✓   │     │  │
│    │   └─────────────┘  └─────────────┘  └─────────────┘     │  │
│    │                                                          │  │
│    └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│    If a pod fails, ReplicaSet creates a new one automatically  │
└─────────────────────────────────────────────────────────────────┘
```

### ReplicaSet Self-Healing

```
┌─────────────────────────────────────────────────────────────────┐
│                   SELF-HEALING IN ACTION                         │
│                                                                  │
│   Desired: 3 replicas                                           │
│                                                                  │
│   STEP 1: Normal state                                          │
│   ┌─────┐  ┌─────┐  ┌─────┐                                    │
│   │Pod 1│  │Pod 2│  │Pod 3│     Current: 3 ✓                   │
│   └─────┘  └─────┘  └─────┘                                    │
│                                                                  │
│   STEP 2: Pod 2 crashes                                         │
│   ┌─────┐  ┌─────┐  ┌─────┐                                    │
│   │Pod 1│  │  ✗  │  │Pod 3│     Current: 2 ✗                   │
│   └─────┘  └─────┘  └─────┘                                    │
│                                                                  │
│   STEP 3: ReplicaSet detects and creates Pod 4                 │
│   ┌─────┐  ┌─────┐  ┌─────┐                                    │
│   │Pod 1│  │Pod 4│  │Pod 3│     Current: 3 ✓                   │
│   └─────┘  └─────┘  └─────┘                                    │
│                                                                  │
│   Time: Seconds (automatic, no human intervention)              │
└─────────────────────────────────────────────────────────────────┘
```

### Example 1: Basic ReplicaSet

```yaml
# nginx-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

#### ReplicaSet YAML Breakdown:

```yaml
spec:
  replicas: 3              # Number of desired pods
  
  selector:                # How ReplicaSet finds pods to manage
    matchLabels:
      app: nginx           # Must match template.metadata.labels
  
  template:                # Pod template (same as Pod spec)
    metadata:
      labels:
        app: nginx         # MUST match selector.matchLabels
    spec:
      containers:          # Container definitions
      - name: nginx
        image: nginx:1.25
```

### Commands to Manage ReplicaSets

```bash
# Create ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# View ReplicaSets
kubectl get replicasets
kubectl get rs          # Short form

# View with more details
kubectl get rs -o wide

# Describe ReplicaSet
kubectl describe rs nginx-rs

# Scale ReplicaSet
kubectl scale rs nginx-rs --replicas=5

# View pods created by ReplicaSet
kubectl get pods -l app=nginx

# Delete ReplicaSet (and its pods)
kubectl delete rs nginx-rs

# Delete ReplicaSet but keep pods
kubectl delete rs nginx-rs --cascade=orphan
```

### Example 2: ReplicaSet with matchExpressions

```yaml
# advanced-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: advanced-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
    - key: version
      operator: NotIn
      values:
      - deprecated
  template:
    metadata:
      labels:
        app: myapp
        environment: production
        version: v1.0
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

#### Selector Operators:

| Operator | Description | Example |
|----------|-------------|---------|
| `In` | Label value is in the set | `environment In [prod, staging]` |
| `NotIn` | Label value is not in the set | `version NotIn [deprecated]` |
| `Exists` | Label key exists | `app Exists` |
| `DoesNotExist` | Label key doesn't exist | `temp DoesNotExist` |

### Why Not Use ReplicaSets Directly?

While ReplicaSets are powerful, you should typically use **Deployments** instead because:

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| Replication | ✅ | ✅ |
| Self-healing | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback | ❌ | ✅ |
| Version history | ❌ | ✅ |
| Pause/Resume | ❌ | ✅ |

---

## 3. Deployments

### What is a Deployment?

A **Deployment** provides declarative updates for Pods and ReplicaSets. It's the recommended way to deploy stateless applications.

Key capabilities:
- **Rolling updates**: Gradually update pods with zero downtime
- **Rollback**: Revert to a previous version instantly
- **Scaling**: Adjust the number of replicas
- **Pause/Resume**: Pause updates to make multiple changes
- **Version history**: Maintain revision history

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT                                │
│                    Name: nginx-deployment                        │
│                    Replicas: 3                                   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Current ReplicaSet                     │   │
│   │                   (nginx-rs-abc123)                      │   │
│   │                                                          │   │
│   │   ┌─────────┐   ┌─────────┐   ┌─────────┐              │   │
│   │   │  Pod 1  │   │  Pod 2  │   │  Pod 3  │              │   │
│   │   │  v1.25  │   │  v1.25  │   │  v1.25  │              │   │
│   │   └─────────┘   └─────────┘   └─────────┘              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Revision History (for rollback)             │   │
│   │                                                          │   │
│   │  Revision 1: nginx-rs-xyz789 (v1.24) - 0 replicas       │   │
│   │  Revision 2: nginx-rs-abc123 (v1.25) - 3 replicas ◄──   │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Example 1: Basic Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
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
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Commands to Manage Deployments

```bash
# Create deployment
kubectl apply -f nginx-deployment.yaml

# View deployments
kubectl get deployments
kubectl get deploy       # Short form

# View deployment details
kubectl describe deployment nginx-deployment

# View rollout status
kubectl rollout status deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment

# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pause rollout
kubectl rollout pause deployment/nginx-deployment

# Resume rollout
kubectl rollout resume deployment/nginx-deployment

# Delete deployment
kubectl delete deployment nginx-deployment
```

### Example 2: Deployment with Rolling Update Strategy

```yaml
# deployment-rolling-update.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  annotations:
    kubernetes.io/change-cause: "Initial deployment v1.0"
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
  
  # Rolling Update Strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired count during update
      maxUnavailable: 1  # Max pods that can be unavailable during update
  
  # Minimum time pod should be ready before considered available
  minReadySeconds: 10
  
  # Number of old ReplicaSets to retain for rollback
  revisionHistoryLimit: 10
  
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

### Rolling Update Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│              ROLLING UPDATE: v1.0 → v2.0                        │
│              (maxSurge=1, maxUnavailable=1)                     │
│              Replicas: 4                                         │
│                                                                  │
│   STEP 1: Initial state (all v1.0)                              │
│   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                           │
│   │v1.0 │  │v1.0 │  │v1.0 │  │v1.0 │    Running: 4             │
│   └─────┘  └─────┘  └─────┘  └─────┘                           │
│                                                                  │
│   STEP 2: Create 1 new pod (maxSurge=1)                         │
│   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                  │
│   │v1.0 │  │v1.0 │  │v1.0 │  │v1.0 │  │v2.0 │  Running: 5      │
│   └─────┘  └─────┘  └─────┘  └─────┘  └─────┘  (starting)      │
│                                                                  │
│   STEP 3: v2.0 ready, terminate 1 v1.0                          │
│   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                  │
│   │ ✗   │  │v1.0 │  │v1.0 │  │v1.0 │  │v2.0 │  Running: 4      │
│   └─────┘  └─────┘  └─────┘  └─────┘  └─────┘                   │
│                                                                  │
│   STEP 4-7: Repeat until all pods are v2.0                      │
│   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                           │
│   │v2.0 │  │v2.0 │  │v2.0 │  │v2.0 │    Running: 4 ✓           │
│   └─────┘  └─────┘  └─────┘  └─────┘                           │
│                                                                  │
│   Total time: ~2-3 minutes for 4 replicas                       │
│   Downtime: ZERO                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Example 3: Deployment with Recreate Strategy

```yaml
# deployment-recreate.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-recreate
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  
  # Recreate Strategy - kills all pods before creating new ones
  strategy:
    type: Recreate
  
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

### Recreate vs Rolling Update

```
┌─────────────────────────────────────────────────────────────────┐
│                RECREATE vs ROLLING UPDATE                        │
│                                                                  │
│   RECREATE STRATEGY                                              │
│   ─────────────────                                              │
│   Step 1: Kill all v1 pods                                      │
│   ┌─────┐  ┌─────┐  ┌─────┐                                    │
│   │  ✗  │  │  ✗  │  │  ✗  │     Downtime starts ⚠️             │
│   └─────┘  └─────┘  └─────┘                                    │
│                                                                  │
│   Step 2: Create all v2 pods                                    │
│   ┌─────┐  ┌─────┐  ┌─────┐                                    │
│   │v2.0 │  │v2.0 │  │v2.0 │     Downtime ends ✓                │
│   └─────┘  └─────┘  └─────┘                                    │
│                                                                  │
│   Use when:                                                      │
│   • App doesn't support running multiple versions               │
│   • Database schema changes require single version              │
│   • Development/testing environments                            │
│                                                                  │
│   ─────────────────────────────────────────────────────────     │
│                                                                  │
│   ROLLING UPDATE STRATEGY                                        │
│   ───────────────────────                                        │
│   Gradual replacement (shown above)                             │
│                                                                  │
│   Use when:                                                      │
│   • Zero downtime is required                                   │
│   • App supports multiple versions running together             │
│   • Production environments                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Example 4: Complete Production Deployment

```yaml
# production-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
  labels:
    app: webapp
    tier: frontend
  annotations:
    kubernetes.io/change-cause: "Deploy webapp v2.1.0"
spec:
  replicas: 5
  selector:
    matchLabels:
      app: webapp
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  
  minReadySeconds: 30
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 600  # Fail if no progress in 10 min
  
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
        version: v2.1.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      # Graceful termination
      terminationGracePeriodSeconds: 60
      
      # Pod distribution across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: webapp
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: webapp
        image: myregistry/webapp:v2.1.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 8080
        
        # Environment variables
        env:
        - name: APP_ENV
          value: production
        - name: LOG_LEVEL
          value: info
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: database-url
        
        # Resource management
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        # Health checks
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        
        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: tmp
          mountPath: /tmp
      
      volumes:
      - name: config
        configMap:
          name: webapp-config
      - name: tmp
        emptyDir: {}
      
      # Image pull secrets for private registry
      imagePullSecrets:
      - name: registry-credentials
```

### Rollback Example

```bash
# Check deployment history
kubectl rollout history deployment/webapp

# Output:
# REVISION  CHANGE-CAUSE
# 1         Deploy webapp v2.0.0
# 2         Deploy webapp v2.1.0

# View specific revision details
kubectl rollout history deployment/webapp --revision=1

# Rollback to previous version
kubectl rollout undo deployment/webapp

# Rollback to specific revision
kubectl rollout undo deployment/webapp --to-revision=1

# Check rollback status
kubectl rollout status deployment/webapp
```

---

## Comparison: Pod vs ReplicaSet vs Deployment

| Feature | Pod | ReplicaSet | Deployment |
|---------|-----|------------|------------|
| Basic unit | ✅ | Manages Pods | Manages ReplicaSets |
| Self-healing | ❌ | ✅ | ✅ |
| Scaling | ❌ | ✅ | ✅ |
| Rolling updates | ❌ | ❌ | ✅ |
| Rollback | ❌ | ❌ | ✅ |
| Version history | ❌ | ❌ | ✅ |
| Use case | Testing | Rarely used directly | Production apps |

---

## Hands-On Exercise

### Step 1: Create a Deployment

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.24 --replicas=3

# Or using YAML
kubectl apply -f nginx-deployment.yaml
```

### Step 2: Verify Resources

```bash
# Check deployment
kubectl get deployment nginx

# Check ReplicaSet (created automatically)
kubectl get rs

# Check Pods (created by ReplicaSet)
kubectl get pods -l app=nginx

# Watch all resources
kubectl get deploy,rs,pods -l app=nginx
```

### Step 3: Perform Rolling Update

```bash
# Update nginx version
kubectl set image deployment/nginx nginx=nginx:1.25

# Watch the rollout
kubectl rollout status deployment/nginx

# Check new ReplicaSet
kubectl get rs
```

### Step 4: Test Rollback

```bash
# Deploy a broken version
kubectl set image deployment/nginx nginx=nginx:broken

# Watch it fail
kubectl rollout status deployment/nginx

# Rollback
kubectl rollout undo deployment/nginx

# Verify
kubectl get pods -l app=nginx
```

### Step 5: Scale

```bash
# Scale up
kubectl scale deployment/nginx --replicas=5

# Scale down
kubectl scale deployment/nginx --replicas=2

# Enable autoscaling
kubectl autoscale deployment/nginx --min=2 --max=10 --cpu-percent=80
```

---

## Best Practices

### 1. Always Use Deployments for Stateless Apps
```yaml
# ✅ Good: Use Deployment
kind: Deployment

# ❌ Avoid: Naked pods or ReplicaSets directly
kind: Pod
kind: ReplicaSet
```

### 2. Define Resource Requests and Limits
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

### 3. Always Configure Health Checks
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

### 4. Use Meaningful Labels
```yaml
labels:
  app: myapp
  version: v1.0.0
  environment: production
  team: backend
```

### 5. Set Appropriate Update Strategy
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

### 6. Use Annotations for Change Tracking
```bash
kubectl annotate deployment/myapp kubernetes.io/change-cause="Deployed v2.0"
```

---

## Summary

| Resource | Purpose | When to Use |
|----------|---------|-------------|
| **Pod** | Smallest deployable unit | Testing, debugging, one-off tasks |
| **ReplicaSet** | Maintains desired pod count | Rarely used directly |
| **Deployment** | Declarative updates for pods | Production stateless applications |

### Key Takeaways:
1. **Pods** are the basic building blocks but ephemeral
2. **ReplicaSets** provide self-healing and scaling
3. **Deployments** add rolling updates, rollbacks, and version history
4. Always use **Deployments** for production workloads
5. Configure **health checks** for reliable deployments
6. Use **rolling updates** for zero-downtime deployments

---

## Next Steps

Your pods are running, but how do clients access them? Head to **[Day 4: Services](../day-4/README.md)** to learn about Kubernetes networking!

---