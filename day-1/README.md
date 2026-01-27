# Day 1: Introduction to Kubernetes

## What is Kubernetes?

**Kubernetes** (often abbreviated as **K8s** — where "8" represents the eight letters between "K" and "s") is an open-source container orchestration platform originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF).

At its core, Kubernetes automates the deployment, scaling, and management of containerized applications. Think of it as an operating system for your cluster — it takes care of where your containers run, how they communicate, how they recover from failures, and how they scale based on demand.

### The Origins

Kubernetes was born from Google's internal system called **Borg**, which Google used for over a decade to run billions of containers per week. In 2014, Google open-sourced Kubernetes, bringing enterprise-grade container orchestration to the masses.

### Key Terminology

| Term | Description |
|------|-------------|
| **Cluster** | A set of machines (nodes) that run containerized applications managed by Kubernetes |
| **Node** | A single machine in the cluster (can be physical or virtual) |
| **Pod** | The smallest deployable unit in K8s; a wrapper around one or more containers |
| **Deployment** | Describes the desired state of your application (replicas, image version, etc.) |
| **Service** | An abstraction that exposes your pods to network traffic |
| **Namespace** | Virtual clusters within a physical cluster for resource isolation |

---

## Why Kubernetes?

### The Problem Before Kubernetes

Before container orchestration, deploying and managing applications came with significant challenges:

1. **Manual Deployment Pain**: Deploying applications across multiple servers required manual intervention, shell scripts, or complex configuration management tools.

2. **Scaling Nightmares**: Scaling applications meant provisioning new servers, installing dependencies, configuring load balancers — all manually.

3. **No Self-Healing**: When a server or application crashed, someone had to notice and fix it manually, often leading to downtime.

4. **Resource Wastage**: Traditional VMs led to underutilized resources because each application needed its own OS.

5. **Environment Inconsistency**: "It works on my machine" syndrome — applications behaved differently across development, staging, and production.

### How Kubernetes Solves These Problems

| Challenge | Kubernetes Solution |
|-----------|---------------------|
| Manual Deployment | Declarative configuration with YAML manifests |
| Scaling | Automatic horizontal pod autoscaling |
| Failures | Self-healing with automatic restarts and rescheduling |
| Resource Wastage | Efficient bin-packing and resource allocation |
| Environment Drift | Consistent container images across all environments |

---

## Advantages of Kubernetes

### 1. 🚀 Container Orchestration at Scale

Kubernetes excels at managing hundreds or thousands of containers across multiple nodes. It handles:
- **Scheduling**: Deciding which node should run which container
- **Load Balancing**: Distributing traffic across healthy pods
- **Resource Allocation**: Ensuring containers have the CPU and memory they need

### 2. 🔄 Self-Healing Capabilities

Kubernetes continuously monitors the health of your applications and automatically:
- **Restarts** containers that fail
- **Replaces** containers on nodes that die
- **Kills** containers that don't respond to health checks
- **Reschedules** workloads when nodes become unavailable

```yaml
# Example: Health check configuration
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

### 3. 📈 Horizontal Scaling

Scale your applications up or down based on demand — either manually or automatically:

```bash
# Manual scaling
kubectl scale deployment my-app --replicas=10

# Auto-scaling based on CPU usage
kubectl autoscale deployment my-app --min=3 --max=10 --cpu-percent=80
```

### 4. 🔄 Rolling Updates & Rollbacks

Deploy new versions of your application with zero downtime:
- **Rolling Updates**: Gradually replace old pods with new ones
- **Rollbacks**: Instantly revert to a previous version if something goes wrong

```bash
# Update image version
kubectl set image deployment/my-app my-app=my-app:v2

# Rollback if needed
kubectl rollout undo deployment/my-app
```

### 5. 🔐 Service Discovery & Load Balancing

Kubernetes provides built-in service discovery:
- Each service gets a stable DNS name
- Automatic load balancing across pods
- No need for external service discovery tools

```yaml
# Example: Service definition
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

### 6. 🔒 Secret & Configuration Management

Securely manage sensitive information and configuration:

```yaml
# ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.example.com"
  LOG_LEVEL: "info"

---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded
```

### 7. 💾 Storage Orchestration

Kubernetes abstracts storage, allowing you to:
- Automatically mount storage systems (local, cloud, NFS, etc.)
- Dynamically provision storage
- Persist data even when pods are destroyed

```yaml
# PersistentVolumeClaim example
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

### 8. 🌐 Multi-Cloud & Hybrid Cloud Support

Kubernetes runs consistently across:
- **Public Clouds**: AWS (EKS), Google Cloud (GKE), Azure (AKS)
- **Private Data Centers**: On-premises infrastructure
- **Hybrid Environments**: Mix of cloud and on-prem
- **Edge Locations**: IoT and edge computing scenarios

This prevents vendor lock-in and enables true cloud portability.

### 9. 📊 Resource Optimization

Kubernetes optimizes infrastructure usage through:
- **Bin Packing**: Efficiently packing containers onto nodes
- **Resource Requests/Limits**: Defining minimum and maximum resources

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 10. 🔌 Extensibility & Ecosystem

Kubernetes has a rich ecosystem:
- **Custom Resource Definitions (CRDs)**: Extend Kubernetes with your own APIs
- **Operators**: Automate complex application lifecycle management
- **Helm**: Package manager for Kubernetes
- **Service Mesh**: Istio, Linkerd for advanced networking
- **Monitoring**: Prometheus, Grafana for observability

---

## Kubernetes vs Traditional Deployment

| Aspect | Traditional | Kubernetes |
|--------|-------------|------------|
| **Deployment** | Manual scripts, SSH | Declarative YAML, `kubectl apply` |
| **Scaling** | Manual VM provisioning | `kubectl scale` or auto-scaling |
| **Updates** | Downtime or complex blue-green | Zero-downtime rolling updates |
| **Recovery** | Manual intervention | Automatic self-healing |
| **Load Balancing** | External LB configuration | Built-in Services |
| **Configuration** | Environment-specific files | ConfigMaps & Secrets |
| **Monitoring** | Various disconnected tools | Integrated with ecosystem |

---

## When to Use Kubernetes

### ✅ Good Use Cases

- Microservices architectures
- Applications requiring high availability
- Multi-cloud or hybrid cloud deployments
- Workloads with variable traffic patterns
- Teams practicing DevOps and CI/CD
- Organizations running many containerized services

### ❌ When Kubernetes Might Be Overkill

- Simple monolithic applications
- Small teams with limited DevOps expertise
- Applications with minimal scaling requirements
- Quick prototypes or MVPs
- Very low-traffic internal tools

---

## Summary

Kubernetes has become the de facto standard for container orchestration because it solves real-world operational challenges at scale. Its key benefits include:

| Benefit | Impact |
|---------|--------|
| **Automation** | Reduces manual operations and human error |
| **Scalability** | Handles growth from 10 to 10,000 containers |
| **Resilience** | Self-heals and maintains desired state |
| **Portability** | Works across any cloud or on-premises |
| **Efficiency** | Optimizes infrastructure utilization |
| **Ecosystem** | Rich tooling and community support |

Learning Kubernetes is an investment that pays off as you build and operate modern cloud-native applications.

---

## 🚀 Getting Started

Before diving into the architecture (Day 2), let's set up a local Kubernetes cluster to experiment with.

### Option 1: Minikube (Recommended for Learning)

```bash
# Install Minikube
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Windows (PowerShell as Admin)
choco install minikube

# Start cluster
minikube start

# Verify
kubectl get nodes
```

### Option 2: Docker Desktop

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
2. Open Settings → Kubernetes
3. Check "Enable Kubernetes"
4. Click "Apply & Restart"

### Option 3: Kind (Kubernetes in Docker)

```bash
# Install Kind
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name learning

# Verify
kubectl cluster-info
```

### Your First kubectl Commands

```bash
# Check cluster info
kubectl cluster-info

# List all nodes
kubectl get nodes

# List all namespaces
kubectl get namespaces

# Run your first pod!
kubectl run hello --image=nginx --port=80

# See it running
kubectl get pods

# Clean up
kubectl delete pod hello
```

🎉 **Congratulations!** You have a working Kubernetes cluster!

---

## Next Steps

Now that you understand what Kubernetes is and have a local cluster running, head to **[Day 2: Kubernetes Architecture](../day-2/README.md)** to learn about the components that make it all work.

---
