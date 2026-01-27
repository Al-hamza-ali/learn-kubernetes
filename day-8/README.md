# Day 8: Namespaces and Resource Management

## Overview

As your Kubernetes cluster grows, you need ways to organize resources and control resource consumption. **Namespaces** provide virtual clusters within a physical cluster, while **Resource Quotas** and **Limit Ranges** help manage resource allocation.

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                            │
│                                                                  │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│   │   Namespace:    │  │   Namespace:    │  │   Namespace:    │ │
│   │   development   │  │   staging       │  │   production    │ │
│   │                 │  │                 │  │                 │ │
│   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│   │  │ Pods      │  │  │  │ Pods      │  │  │  │ Pods      │  │ │
│   │  │ Services  │  │  │  │ Services  │  │  │  │ Services  │  │ │
│   │  │ ConfigMaps│  │  │  │ ConfigMaps│  │  │  │ ConfigMaps│  │ │
│   │  │ Secrets   │  │  │  │ Secrets   │  │  │  │ Secrets   │  │ │
│   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│   │                 │  │                 │  │                 │ │
│   │  CPU: 4 cores   │  │  CPU: 8 cores   │  │  CPU: 32 cores  │ │
│   │  Mem: 8Gi       │  │  Mem: 16Gi      │  │  Mem: 64Gi      │ │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                  │
│   Cluster-scoped resources: Nodes, PVs, ClusterRoles            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Namespaces

### What is a Namespace?

A **Namespace** is a virtual cluster within a Kubernetes cluster. Use namespaces to:
- **Isolate environments** (dev, staging, prod)
- **Organize teams** (team-a, team-b)
- **Separate applications** (frontend, backend)
- **Apply resource quotas** per namespace

### Default Namespaces

| Namespace | Purpose |
|-----------|---------|
| `default` | Default namespace for resources without a specified namespace |
| `kube-system` | Kubernetes system components (API server, scheduler, etc.) |
| `kube-public` | Publicly accessible data, like cluster info |
| `kube-node-lease` | Node heartbeat data for node health |

### Creating Namespaces

```bash
# Imperative
kubectl create namespace development

# From YAML
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
    team: backend
EOF
```

### Namespace YAML

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
  annotations:
    description: "Production workloads"
```

### Working with Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create resources in a namespace
kubectl apply -f deployment.yaml -n development

# Set default namespace for kubectl
kubectl config set-context --current --namespace=development

# View current namespace
kubectl config view --minify | grep namespace

# Get resources from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A    # Short form
```

---

## Resource Quotas

**Resource Quotas** limit the total amount of resources that can be consumed in a namespace.

### Types of Quotas

| Category | Resources |
|----------|-----------|
| **Compute** | cpu, memory, requests.cpu, limits.cpu, requests.memory, limits.memory |
| **Storage** | requests.storage, persistentvolumeclaims, \<storage-class\>.storageclass.storage.k8s.io/requests.storage |
| **Object Count** | pods, services, secrets, configmaps, deployments.apps, etc. |

### Example: Resource Quota

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # Compute resources
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    
    # Object counts
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
    persistentvolumeclaims: "5"
    
    # Storage
    requests.storage: 100Gi
```

### Multiple Quotas

```yaml
# compute-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi

---
# object-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: development
spec:
  hard:
    pods: "20"
    services: "10"
    services.loadbalancers: "2"
    services.nodeports: "5"
```

### Scoped Quotas

Apply quotas to specific priority classes or scopes:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - high-priority
```

---

## Limit Ranges

**LimitRange** sets default, minimum, and maximum resource constraints for individual containers/pods in a namespace.

```
┌─────────────────────────────────────────────────────────────────┐
│                ResourceQuota vs LimitRange                       │
│                                                                  │
│   ResourceQuota:                                                │
│   "The entire namespace can use at most 16Gi memory"            │
│                                                                  │
│   LimitRange:                                                   │
│   "Each container must request between 64Mi and 1Gi memory"     │
│   "If not specified, default to 256Mi"                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Example: LimitRange

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: development
spec:
  limits:
  # Container limits
  - type: Container
    default:          # Default limits if not specified
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # Default requests if not specified
      cpu: "100m"
      memory: "128Mi"
    min:              # Minimum allowed
      cpu: "50m"
      memory: "64Mi"
    max:              # Maximum allowed
      cpu: "2"
      memory: "2Gi"
    maxLimitRequestRatio:  # Limit can't exceed 4x request
      cpu: "4"
      memory: "4"
  
  # Pod limits
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
  
  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 50Gi
```

### LimitRange Behavior

```yaml
# Pod without resource specs
apiVersion: v1
kind: Pod
metadata:
  name: no-resources
  namespace: development
spec:
  containers:
  - name: app
    image: nginx
    # No resources specified

# After LimitRange, becomes:
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"      # defaultRequest
        memory: "128Mi"
      limits:
        cpu: "500m"      # default
        memory: "256Mi"
```

---

## Complete Environment Setup

```yaml
# development-environment.yaml

# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development

---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    configmaps: "20"
    secrets: "20"
    persistentvolumeclaims: "10"
    requests.storage: 50Gi

---
# Limit Range
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    min:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "1"
      memory: "1Gi"
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi

---
# Network Policy (basic isolation)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: development
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## Resource Requests and Limits

Every container should specify resource requests and limits:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"       # 0.1 CPU cores (guaranteed)
        memory: "128Mi"   # 128 MB (guaranteed)
      limits:
        cpu: "500m"       # Max 0.5 CPU cores
        memory: "256Mi"   # Max 256 MB (OOMKilled if exceeded)
```

### CPU vs Memory Behavior

| Resource | When Limit Exceeded |
|----------|---------------------|
| **CPU** | Throttled (slowed down) |
| **Memory** | Container killed (OOMKilled) |

### CPU Units

| Value | Meaning |
|-------|---------|
| `1` | 1 CPU core |
| `500m` | 0.5 CPU (500 millicores) |
| `100m` | 0.1 CPU (100 millicores) |

### Memory Units

| Value | Meaning |
|-------|---------|
| `128Mi` | 128 Mebibytes (134 MB) |
| `1Gi` | 1 Gibibyte (1.07 GB) |
| `256M` | 256 Megabytes |

---

## Quality of Service (QoS) Classes

Kubernetes assigns QoS classes based on resource settings:

| QoS Class | Criteria | Eviction Priority |
|-----------|----------|-------------------|
| **Guaranteed** | requests = limits for all containers | Last to be evicted |
| **Burstable** | At least one request or limit set | Medium priority |
| **BestEffort** | No requests or limits | First to be evicted |

### Guaranteed QoS

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"       # Same as request
    memory: "256Mi"   # Same as request
```

### Burstable QoS

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"       # Higher than request
    memory: "256Mi"   # Higher than request
```

### BestEffort QoS

```yaml
# No resources specified
containers:
- name: app
  image: nginx
```

---

## Priority Classes

Control pod scheduling and eviction priority:

```yaml
# priority-class.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority for critical workloads"
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority for batch jobs"
preemptionPolicy: Never
```

### Using Priority Classes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```

---

## Commands Reference

```bash
# Namespaces
kubectl get namespaces
kubectl create namespace dev
kubectl delete namespace dev
kubectl describe namespace dev

# Resource Quotas
kubectl get resourcequota -n development
kubectl describe resourcequota compute-quota -n development

# Limit Ranges
kubectl get limitrange -n development
kubectl describe limitrange resource-limits -n development

# Check resource usage
kubectl top nodes
kubectl top pods -n development

# View QoS class
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# Check quota usage
kubectl get resourcequota -n development -o yaml
```

---

## Best Practices

### 1. Use Namespaces for Environment Separation
```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

### 2. Always Set Resource Requests and Limits
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### 3. Use LimitRanges for Default Values
```yaml
# Ensures all containers have resources set
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

### 4. Set Appropriate Quotas per Environment
```yaml
# Production gets more resources
hard:
  requests.cpu: "32"
  requests.memory: 64Gi
```

### 5. Use Guaranteed QoS for Critical Workloads

---

## Summary

| Concept | Purpose |
|---------|---------|
| **Namespace** | Virtual cluster for resource isolation |
| **ResourceQuota** | Limit total resources in a namespace |
| **LimitRange** | Default/min/max for individual containers |
| **Requests** | Guaranteed resources |
| **Limits** | Maximum resources |
| **QoS Classes** | Eviction priority based on resources |
| **PriorityClass** | Pod scheduling and eviction priority |

### Key Takeaways

1. **Use namespaces** to organize and isolate workloads
2. **Set ResourceQuotas** to prevent resource exhaustion
3. **Use LimitRanges** for sensible defaults
4. **Always specify requests and limits** in production
5. **Understand QoS classes** for predictable behavior

---

## Next Steps

Learn about batch processing with **[Day 9: Jobs and CronJobs](../day-9/README.md)**!
