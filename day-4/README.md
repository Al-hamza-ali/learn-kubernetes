# Day 4: Kubernetes Services

## Overview

In Kubernetes, Pods are ephemeral — they can be created, destroyed, and replaced at any time. Each Pod gets its own IP address, but these IPs change when Pods are recreated. **Services** provide a stable networking endpoint to access Pods, regardless of their lifecycle.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM SERVICES SOLVE                    │
│                                                                  │
│   Without Services:                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Frontend Pod                                            │   │
│   │       │                                                  │   │
│   │       │  How do I reach the backend?                    │   │
│   │       │  Pod IPs keep changing!                         │   │
│   │       ▼                                                  │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐                  │   │
│   │  │Backend 1│  │Backend 2│  │Backend 3│  IP addresses    │   │
│   │  │10.1.1.5 │  │10.1.2.3 │  │10.1.3.7 │  change when     │   │
│   │  │   ✗     │  │   ?     │  │   ?     │  pods restart!   │   │
│   │  └─────────┘  └─────────┘  └─────────┘                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   With Services:                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Frontend Pod                                            │   │
│   │       │                                                  │   │
│   │       │  backend-service:80                             │   │
│   │       │  (stable DNS name & IP)                         │   │
│   │       ▼                                                  │   │
│   │  ┌──────────────────────────────────────────┐           │   │
│   │  │           SERVICE (backend-service)       │           │   │
│   │  │           ClusterIP: 10.96.50.100        │           │   │
│   │  └──────────────────┬───────────────────────┘           │   │
│   │                     │ Load Balances                      │   │
│   │       ┌─────────────┼─────────────┐                     │   │
│   │       ▼             ▼             ▼                     │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐                  │   │
│   │  │Backend 1│  │Backend 2│  │Backend 3│                  │   │
│   │  └─────────┘  └─────────┘  └─────────┘                  │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What is a Service?

A **Service** is an abstraction that defines a logical set of Pods and a policy to access them. Services enable:

- **Stable Network Identity**: Fixed IP address and DNS name
- **Load Balancing**: Distribute traffic across multiple Pods
- **Service Discovery**: Find Pods using DNS or environment variables
- **Decoupling**: Frontend doesn't need to know about backend Pod details

### How Services Find Pods

Services use **label selectors** to identify which Pods to route traffic to:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE LABEL SELECTION                       │
│                                                                  │
│   Service: backend-service                                       │
│   Selector: app=backend                                          │
│                                                                  │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                    All Pods in Cluster                     │ │
│   │                                                            │ │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │ │
│   │  │ Pod A       │  │ Pod B       │  │ Pod C       │        │ │
│   │  │ app=backend │  │ app=backend │  │ app=frontend│        │ │
│   │  │     ✓       │  │     ✓       │  │     ✗       │        │ │
│   │  └─────────────┘  └─────────────┘  └─────────────┘        │ │
│   │                                                            │ │
│   │  ┌─────────────┐  ┌─────────────┐                         │ │
│   │  │ Pod D       │  │ Pod E       │                         │ │
│   │  │ app=backend │  │ app=cache   │                         │ │
│   │  │     ✓       │  │     ✗       │                         │ │
│   │  └─────────────┘  └─────────────┘                         │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   Service routes traffic to: Pod A, Pod B, Pod D                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Service Types

Kubernetes offers four types of Services, each serving different use cases:

```
┌─────────────────────────────────────────────────────────────────┐
│                      SERVICE TYPES                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                              ││
│  │   ClusterIP ──► NodePort ──► LoadBalancer                   ││
│  │   (internal)    (+ node     (+ cloud LB)                    ││
│  │                  ports)                                      ││
│  │                                                              ││
│  │   ExternalName (DNS alias - different category)             ││
│  │                                                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Each type builds on the previous:                              │
│  • ClusterIP: Internal cluster access only                      │
│  • NodePort: ClusterIP + port on every node                     │
│  • LoadBalancer: NodePort + cloud load balancer                 │
│  • ExternalName: DNS CNAME record for external services         │
└─────────────────────────────────────────────────────────────────┘
```

| Type | Access From | Use Case |
|------|-------------|----------|
| **ClusterIP** | Inside cluster only | Internal microservices communication |
| **NodePort** | Outside via node IP:port | Development, testing, simple external access |
| **LoadBalancer** | External via cloud LB | Production external access on cloud |
| **ExternalName** | N/A (DNS alias) | Connect to external services |

---

## 1. ClusterIP Service (Default)

**ClusterIP** exposes the Service on an internal IP in the cluster. This is the default Service type and is only reachable from within the cluster.

### ClusterIP Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLUSTER                                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    ClusterIP Service                     │   │
│   │                                                          │   │
│   │           ┌─────────────────────────────┐               │   │
│   │           │   backend-service           │               │   │
│   │           │   ClusterIP: 10.96.50.100   │               │   │
│   │           │   Port: 80                  │               │   │
│   │           └──────────────┬──────────────┘               │   │
│   │                          │                               │   │
│   │              ┌───────────┼───────────┐                  │   │
│   │              ▼           ▼           ▼                  │   │
│   │         ┌────────┐  ┌────────┐  ┌────────┐             │   │
│   │         │ Pod 1  │  │ Pod 2  │  │ Pod 3  │             │   │
│   │         │ :8080  │  │ :8080  │  │ :8080  │             │   │
│   │         └────────┘  └────────┘  └────────┘             │   │
│   │                                                          │   │
│   │   Access: http://backend-service:80                     │   │
│   │           http://10.96.50.100:80                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ❌ NOT accessible from outside the cluster                    │
└─────────────────────────────────────────────────────────────────┘
```

### Example: ClusterIP Service

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  type: ClusterIP          # Default, can be omitted
  selector:
    app: backend           # Matches pods with this label
  ports:
  - name: http
    port: 80               # Service port (what clients connect to)
    targetPort: 8080       # Pod port (where app listens)
    protocol: TCP
```

### Example: Service with Deployment

```yaml
# backend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx:1.25
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

### Accessing ClusterIP Service

```bash
# From within a pod in the cluster:

# Using service name (DNS)
curl http://backend-service:80

# Using service name with namespace
curl http://backend-service.default.svc.cluster.local:80

# Using ClusterIP directly
curl http://10.96.50.100:80

# Test from a debug pod
kubectl run debug --rm -it --image=busybox -- /bin/sh
# Inside the pod:
wget -qO- http://backend-service:80
```

---

## 2. NodePort Service

**NodePort** exposes the Service on each Node's IP at a static port. A ClusterIP Service is automatically created, and the NodePort Service routes to it.

### NodePort Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   EXTERNAL CLIENT                                                │
│        │                                                         │
│        │  http://<NodeIP>:30080                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                       CLUSTER                            │   │
│   │                                                          │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │   Node 1          Node 2          Node 3          │  │   │
│   │  │   Port: 30080     Port: 30080     Port: 30080     │  │   │
│   │  │      │               │               │            │  │   │
│   │  └──────┼───────────────┼───────────────┼────────────┘  │   │
│   │         │               │               │                │   │
│   │         └───────────────┼───────────────┘                │   │
│   │                         │                                │   │
│   │                         ▼                                │   │
│   │              ┌───────────────────┐                      │   │
│   │              │  NodePort Service │                      │   │
│   │              │  ClusterIP:       │                      │   │
│   │              │  10.96.50.100:80  │                      │   │
│   │              │  NodePort: 30080  │                      │   │
│   │              └─────────┬─────────┘                      │   │
│   │                        │                                 │   │
│   │           ┌────────────┼────────────┐                   │   │
│   │           ▼            ▼            ▼                   │   │
│   │      ┌────────┐   ┌────────┐   ┌────────┐              │   │
│   │      │ Pod 1  │   │ Pod 2  │   │ Pod 3  │              │   │
│   │      │ :8080  │   │ :8080  │   │ :8080  │              │   │
│   │      └────────┘   └────────┘   └────────┘              │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Access externally: http://<any-node-ip>:30080                 │
│   Access internally: http://backend-service:80                  │
└─────────────────────────────────────────────────────────────────┘
```

### NodePort Range

- Default range: **30000-32767**
- You can specify a port or let Kubernetes assign one
- Same port is opened on ALL nodes

### Example: NodePort Service

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - name: http
    port: 80              # ClusterIP port (internal)
    targetPort: 8080      # Pod port
    nodePort: 30080       # Node port (external) - optional, auto-assigned if not specified
    protocol: TCP
```

### Example: Complete NodePort Setup

```yaml
# frontend-nodeport.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.25
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### Accessing NodePort Service

```bash
# Get node IPs
kubectl get nodes -o wide

# Access from outside the cluster
curl http://<node-ip>:30080

# If using Minikube
minikube service frontend-nodeport --url

# Still works internally
curl http://frontend-nodeport:80
```

### NodePort Limitations

| Limitation | Description |
|------------|-------------|
| **Port range** | Only 30000-32767 |
| **No load balancing** | Client connects directly to node |
| **Security** | Exposes ports on all nodes |
| **HA concerns** | If node goes down, clients must switch to another node |

---

## 3. LoadBalancer Service

**LoadBalancer** provisions an external load balancer (in cloud environments) and assigns a fixed external IP to the Service.

### LoadBalancer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   INTERNET                                                       │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              CLOUD LOAD BALANCER                         │   │
│   │              External IP: 203.0.113.50                   │   │
│   │              Port: 80                                    │   │
│   └────────────────────────┬────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                       CLUSTER                            │   │
│   │                                                          │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │   Node 1          Node 2          Node 3          │  │   │
│   │  │   Port: 31234     Port: 31234     Port: 31234     │  │   │
│   │  └──────┬───────────────┬───────────────┬────────────┘  │   │
│   │         │               │               │                │   │
│   │         └───────────────┼───────────────┘                │   │
│   │                         │                                │   │
│   │              ┌──────────┴──────────┐                    │   │
│   │              │  LoadBalancer Svc   │                    │   │
│   │              │  ClusterIP:         │                    │   │
│   │              │  10.96.50.100:80    │                    │   │
│   │              └──────────┬──────────┘                    │   │
│   │                         │                                │   │
│   │           ┌─────────────┼─────────────┐                 │   │
│   │           ▼             ▼             ▼                 │   │
│   │      ┌────────┐    ┌────────┐    ┌────────┐            │   │
│   │      │ Pod 1  │    │ Pod 2  │    │ Pod 3  │            │   │
│   │      └────────┘    └────────┘    └────────┘            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   External Access: http://203.0.113.50:80                       │
└─────────────────────────────────────────────────────────────────┘
```

### Example: LoadBalancer Service

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-lb
  annotations:
    # Cloud provider specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
```

### Example: Complete LoadBalancer Setup

```yaml
# webapp-loadbalancer.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: webapp-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
```

### Checking LoadBalancer Status

```bash
# Create the service
kubectl apply -f webapp-loadbalancer.yaml

# Check service (EXTERNAL-IP shows <pending> initially)
kubectl get service webapp-loadbalancer

# Wait for external IP
kubectl get service webapp-loadbalancer -w

# Example output when ready:
# NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
# webapp-loadbalancer   LoadBalancer   10.96.100.50    203.0.113.50    80:31234/TCP   2m

# Access the service
curl http://203.0.113.50:80
```

### Cloud Provider Annotations

#### AWS (EKS)

```yaml
metadata:
  annotations:
    # Use Network Load Balancer
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    
    # Internal load balancer
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    
    # SSL certificate
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:..."
```

#### Google Cloud (GKE)

```yaml
metadata:
  annotations:
    # Internal load balancer
    cloud.google.com/load-balancer-type: "Internal"
    
    # Static IP
    kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
```

#### Azure (AKS)

```yaml
metadata:
  annotations:
    # Internal load balancer
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    
    # Resource group
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "my-rg"
```

---

## 4. ExternalName Service

**ExternalName** maps a Service to a DNS name (CNAME record). It does not create a proxy or load balancer — it simply returns the CNAME record.

### ExternalName Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ExternalName Service                          │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                       CLUSTER                            │   │
│   │                                                          │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │                    Pod                           │   │   │
│   │   │                                                  │   │   │
│   │   │   curl http://my-database:5432                  │   │   │
│   │   │         │                                        │   │   │
│   │   │         ▼                                        │   │   │
│   │   │   ┌───────────────────────────────────────┐     │   │   │
│   │   │   │  Service: my-database                  │     │   │   │
│   │   │   │  Type: ExternalName                    │     │   │   │
│   │   │   │  ExternalName: db.example.com         │     │   │   │
│   │   │   └───────────────────────────────────────┘     │   │   │
│   │   │         │                                        │   │   │
│   │   │         │ DNS CNAME lookup                      │   │   │
│   │   │         ▼                                        │   │   │
│   │   │   Returns: db.example.com                       │   │   │
│   │   │                                                  │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            │ Actual connection                   │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 EXTERNAL SERVICE                         │   │
│   │                 db.example.com:5432                      │   │
│   │                 (RDS, CloudSQL, etc.)                    │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Example: ExternalName Service

```yaml
# externalname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: database.us-east-1.rds.amazonaws.com
```

### Use Cases for ExternalName

```yaml
# Connect to external database
apiVersion: v1
kind: Service
metadata:
  name: production-db
  namespace: myapp
spec:
  type: ExternalName
  externalName: mydb.abc123.us-east-1.rds.amazonaws.com

---
# Connect to external API
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  type: ExternalName
  externalName: api.stripe.com

---
# Cross-namespace service reference
apiVersion: v1
kind: Service
metadata:
  name: shared-cache
  namespace: frontend
spec:
  type: ExternalName
  externalName: redis.backend.svc.cluster.local
```

### Accessing ExternalName Service

```bash
# From within a pod:
# Instead of connecting to database.us-east-1.rds.amazonaws.com
# You can use the service name:

psql -h my-database -U admin -d myapp

# The service name resolves to the external DNS name
```

---

## 5. Headless Service

A **Headless Service** (ClusterIP: None) doesn't allocate a cluster IP. Instead, DNS returns the Pod IPs directly. This is useful for stateful applications where clients need to connect to specific Pods.

### Headless Service Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│               REGULAR SERVICE vs HEADLESS SERVICE                │
│                                                                  │
│   Regular ClusterIP Service:                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   DNS: my-service.default.svc.cluster.local             │   │
│   │   Returns: 10.96.50.100 (ClusterIP)                     │   │
│   │                                                          │   │
│   │   Traffic is load-balanced across pods                  │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Headless Service (clusterIP: None):                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   DNS: my-service.default.svc.cluster.local             │   │
│   │   Returns: 10.1.0.5, 10.1.0.6, 10.1.0.7 (Pod IPs)       │   │
│   │                                                          │   │
│   │   Client can connect to specific pod                    │   │
│   │                                                          │   │
│   │   Individual Pod DNS:                                    │   │
│   │   pod-0.my-service.default.svc.cluster.local → 10.1.0.5 │   │
│   │   pod-1.my-service.default.svc.cluster.local → 10.1.0.6 │   │
│   │   pod-2.my-service.default.svc.cluster.local → 10.1.0.7 │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Example: Headless Service

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None          # This makes it headless
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

### Headless Service with StatefulSet

```yaml
# statefulset-with-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql       # References the headless service
  replicas: 3
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
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
```

### DNS Records for Headless Service

```bash
# Query the service (returns all pod IPs)
nslookup mysql.default.svc.cluster.local

# Query individual pod (StatefulSet)
nslookup mysql-0.mysql.default.svc.cluster.local
nslookup mysql-1.mysql.default.svc.cluster.local
nslookup mysql-2.mysql.default.svc.cluster.local
```

---

## Service Discovery

Kubernetes provides two ways for Pods to discover Services:

### 1. Environment Variables

When a Pod starts, Kubernetes injects environment variables for each active Service:

```bash
# Environment variables for a service named "backend"
BACKEND_SERVICE_HOST=10.96.50.100
BACKEND_SERVICE_PORT=80
BACKEND_PORT=tcp://10.96.50.100:80
BACKEND_PORT_80_TCP=tcp://10.96.50.100:80
BACKEND_PORT_80_TCP_PROTO=tcp
BACKEND_PORT_80_TCP_PORT=80
BACKEND_PORT_80_TCP_ADDR=10.96.50.100
```

**Limitation**: Service must exist before the Pod is created.

### 2. DNS (Recommended)

CoreDNS provides DNS resolution for Services:

```
┌─────────────────────────────────────────────────────────────────┐
│                      DNS NAMING CONVENTION                       │
│                                                                  │
│   Full DNS Name:                                                 │
│   <service-name>.<namespace>.svc.<cluster-domain>               │
│                                                                  │
│   Examples:                                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   backend.default.svc.cluster.local                     │   │
│   │      │       │      │       │                            │   │
│   │      │       │      │       └── Cluster domain          │   │
│   │      │       │      └── Service indicator               │   │
│   │      │       └── Namespace                               │   │
│   │      └── Service name                                    │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Short forms (within same namespace):                          │
│   • backend                                                      │
│   • backend.default                                              │
│   • backend.default.svc                                          │
│   • backend.default.svc.cluster.local                           │
│                                                                  │
│   Cross-namespace access:                                        │
│   • backend.other-namespace                                      │
│   • backend.other-namespace.svc.cluster.local                   │
└─────────────────────────────────────────────────────────────────┘
```

### Testing DNS Resolution

```bash
# Create a debug pod
kubectl run debug --rm -it --image=busybox:1.36 -- /bin/sh

# Inside the pod:

# Short name (same namespace)
nslookup backend

# Full name
nslookup backend.default.svc.cluster.local

# Cross-namespace
nslookup backend.production.svc.cluster.local

# Test connectivity
wget -qO- http://backend:80
```

---

## Endpoints and EndpointSlices

### Endpoints

**Endpoints** track the IP addresses of Pods that match a Service's selector:

```bash
# View endpoints
kubectl get endpoints backend-service

# Output:
# NAME              ENDPOINTS                                   AGE
# backend-service   10.1.0.5:8080,10.1.0.6:8080,10.1.0.7:8080   5m

# Detailed view
kubectl describe endpoints backend-service
```

### EndpointSlices (Kubernetes 1.21+)

**EndpointSlices** provide a more scalable alternative to Endpoints:

```yaml
# Example EndpointSlice (auto-generated)
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-service-abc12
  labels:
    kubernetes.io/service-name: backend-service
addressType: IPv4
endpoints:
- addresses:
  - "10.1.0.5"
  conditions:
    ready: true
  nodeName: node-1
- addresses:
  - "10.1.0.6"
  conditions:
    ready: true
  nodeName: node-2
ports:
- name: http
  port: 8080
  protocol: TCP
```

```bash
# View EndpointSlices
kubectl get endpointslices -l kubernetes.io/service-name=backend-service
```

---

## Multi-Port Services

Services can expose multiple ports:

```yaml
# multi-port-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - name: http           # Name required when multiple ports
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

---

## Service with Named Ports

Using named ports in Pods makes configurations more flexible:

```yaml
# deployment-with-named-ports.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: myapp:1.0
        ports:
        - name: http-port       # Named port
          containerPort: 8080
        - name: metrics-port    # Named port
          containerPort: 9090

---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: http-port      # Reference by name
  - name: metrics
    port: 9090
    targetPort: metrics-port   # Reference by name
```

---

## Session Affinity

By default, Services use random load balancing. Use **sessionAffinity** to route requests from the same client to the same Pod:

```yaml
# session-affinity-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  sessionAffinity: ClientIP       # Sticky sessions based on client IP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600        # 1 hour
  ports:
  - port: 80
    targetPort: 8080
```

### Session Affinity Options

| Value | Description |
|-------|-------------|
| `None` | Default, random load balancing |
| `ClientIP` | Route all requests from same client IP to same Pod |

---

## External Traffic Policy

Control how traffic is routed to Pods:

```yaml
# external-traffic-policy.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: LoadBalancer
  selector:
    app: webapp
  externalTrafficPolicy: Local    # Preserve client source IP
  ports:
  - port: 80
    targetPort: 8080
```

### Traffic Policy Options

| Policy | Description | Use Case |
|--------|-------------|----------|
| `Cluster` (default) | Traffic may hop to other nodes | Better load distribution |
| `Local` | Traffic only goes to pods on the same node | Preserve source IP, lower latency |

```
┌─────────────────────────────────────────────────────────────────┐
│         externalTrafficPolicy: Cluster vs Local                  │
│                                                                  │
│   Cluster (default):                                             │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Traffic → Node 1 → kube-proxy → Pod on Node 2         │   │
│   │                                                          │   │
│   │  ✓ Better load balancing                                │   │
│   │  ✗ Extra network hop                                    │   │
│   │  ✗ Source IP is lost (SNAT)                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Local:                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Traffic → Node 1 → Pod on Node 1 (only)                │   │
│   │                                                          │   │
│   │  ✓ Preserves source IP                                  │   │
│   │  ✓ Lower latency                                        │   │
│   │  ✗ Uneven load if pods not evenly distributed          │   │
│   │  ✗ No traffic if no pod on receiving node               │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Service Commands Reference

```bash
# Create service
kubectl apply -f service.yaml

# Create service imperatively
kubectl expose deployment nginx --port=80 --target-port=8080 --type=ClusterIP

# List services
kubectl get services
kubectl get svc             # Short form

# Get service details
kubectl get svc webapp -o wide
kubectl describe svc webapp

# Get service YAML
kubectl get svc webapp -o yaml

# Get endpoints
kubectl get endpoints webapp
kubectl get endpointslices -l kubernetes.io/service-name=webapp

# Test service from within cluster
kubectl run debug --rm -it --image=busybox -- wget -qO- http://webapp:80

# Delete service
kubectl delete svc webapp

# Edit service
kubectl edit svc webapp
```

---

## Complete Example: Microservices Application

```yaml
# microservices-app.yaml

# Frontend Deployment + LoadBalancer Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: frontend-app:1.0
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_URL
          value: "http://backend:8080"

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80

---
# Backend Deployment + ClusterIP Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: backend-app:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_HOST
          value: "database"
        - name: CACHE_HOST
          value: "redis"

---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080

---
# Redis Cache - ClusterIP
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379

---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379

---
# External Database - ExternalName
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ExternalName
  externalName: mydb.us-east-1.rds.amazonaws.com
```

---

## Best Practices

### 1. Always Use DNS for Service Discovery
```yaml
# ✅ Good - Use service DNS name
env:
- name: BACKEND_URL
  value: "http://backend.default.svc.cluster.local:8080"

# ❌ Avoid - Hardcoded IP addresses
env:
- name: BACKEND_URL
  value: "http://10.96.50.100:8080"
```

### 2. Use Meaningful Port Names
```yaml
ports:
- name: http          # ✅ Descriptive
  port: 80
- name: grpc          # ✅ Descriptive
  port: 9000
```

### 3. Choose the Right Service Type
| Scenario | Service Type |
|----------|--------------|
| Internal microservices | ClusterIP |
| Development/testing access | NodePort |
| Production external access | LoadBalancer + Ingress |
| External resources | ExternalName |
| StatefulSets | Headless |

### 4. Configure Health Checks
Services only route to ready Pods. Ensure your Pods have readiness probes:
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 5. Use Labels Consistently
```yaml
metadata:
  labels:
    app: myapp
    tier: backend
    version: v1.0.0
    environment: production
```

---

## Summary

| Service Type | Access Scope | Use Case |
|--------------|--------------|----------|
| **ClusterIP** | Internal only | Microservices, databases, caches |
| **NodePort** | External via node ports | Development, simple external access |
| **LoadBalancer** | External via cloud LB | Production external services |
| **ExternalName** | DNS alias | External databases, APIs |
| **Headless** | Direct pod access | StatefulSets, service discovery |

### Key Takeaways

1. **Services provide stable endpoints** for ephemeral Pods
2. **Use DNS** for service discovery (more flexible than env vars)
3. **ClusterIP** is the default and most common type
4. **LoadBalancer** provisions cloud resources and has costs
5. **Headless services** are essential for StatefulSets
6. **Always configure readiness probes** for proper routing

---

## Next Steps

Services work great for internal traffic, but for HTTP routing from the internet, you need Ingress. Continue to **[Day 5: Ingress](../day-5/README.md)**!

---