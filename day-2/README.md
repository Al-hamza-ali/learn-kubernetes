# Day 2: Kubernetes Architecture

## Overview

Kubernetes follows a **client-server architecture** with a clear separation between the **Control Plane** (the brain) and **Worker Nodes** (the muscles). This design enables Kubernetes to be highly available, scalable, and fault-tolerant.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES CLUSTER                               │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           CONTROL PLANE (Master)                         │ │
│  │                                                                          │ │
│  │   ┌─────────────┐    ┌─────────────┐    ┌──────────────────────────┐    │ │
│  │   │             │    │             │    │                          │    │ │
│  │   │ API Server  │◄───│  Scheduler  │    │   Controller Manager     │    │ │
│  │   │  (kube-     │    │  (kube-     │    │   (kube-controller-      │    │ │
│  │   │  apiserver) │    │  scheduler) │    │    manager)              │    │ │
│  │   │             │    │             │    │                          │    │ │
│  │   └──────┬──────┘    └─────────────┘    └──────────────────────────┘    │ │
│  │          │                                                               │ │
│  │          │           ┌─────────────┐    ┌──────────────────────────┐    │ │
│  │          │           │             │    │                          │    │ │
│  │          └──────────►│    etcd     │    │  Cloud Controller Mgr   │    │ │
│  │                      │  (Database) │    │  (cloud-controller-mgr)  │    │ │
│  │                      │             │    │                          │    │ │
│  │                      └─────────────┘    └──────────────────────────┘    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                        │
│                                      │ Network                                │
│                                      ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                            WORKER NODES                                  │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────┐      ┌──────────────────────────┐        │ │
│  │  │        NODE 1            │      │        NODE 2            │        │ │
│  │  │  ┌────────┐ ┌─────────┐  │      │  ┌────────┐ ┌─────────┐  │        │ │
│  │  │  │kubelet │ │kube-    │  │      │  │kubelet │ │kube-    │  │        │ │
│  │  │  │        │ │proxy    │  │      │  │        │ │proxy    │  │        │ │
│  │  │  └────────┘ └─────────┘  │      │  └────────┘ └─────────┘  │        │ │
│  │  │  ┌────────────────────┐  │      │  ┌────────────────────┐  │        │ │
│  │  │  │ Container Runtime  │  │      │  │ Container Runtime  │  │        │ │
│  │  │  │ (containerd/CRI-O) │  │      │  │ (containerd/CRI-O) │  │        │ │
│  │  │  └────────────────────┘  │      │  └────────────────────┘  │        │ │
│  │  │  ┌─────┐┌─────┐┌─────┐   │      │  ┌─────┐┌─────┐┌─────┐   │        │ │
│  │  │  │ Pod ││ Pod ││ Pod │   │      │  │ Pod ││ Pod ││ Pod │   │        │ │
│  │  │  └─────┘└─────┘└─────┘   │      │  └─────┘└─────┘└─────┘   │        │ │
│  │  └──────────────────────────┘      └──────────────────────────┘        │ │
│  │                                                                          │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Control Plane Components

The Control Plane is responsible for making global decisions about the cluster (scheduling, detecting and responding to events). It can run on any machine in the cluster, but for simplicity, all control plane components are typically started on the same machine(s).

### 1. kube-apiserver (API Server)

The **API Server** is the central management entity and the only component that directly communicates with etcd. It acts as the front-end for the Kubernetes control plane.

#### Key Responsibilities:
- **RESTful API Gateway**: Exposes the Kubernetes API over HTTPS
- **Authentication & Authorization**: Validates and authorizes all API requests
- **Admission Control**: Applies policies before persisting objects
- **API Aggregation**: Extends the API with custom resources

#### How It Works:

```
┌─────────────────────────────────────────────────────────────────┐
│                         API REQUEST FLOW                         │
│                                                                  │
│   kubectl/Client                                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────┐    ┌──────────────┐    ┌──────────────┐           │
│   │  Auth   │───►│  Admission   │───►│  Validation  │           │
│   │ (AuthN) │    │  Controller  │    │              │           │
│   └─────────┘    └──────────────┘    └──────────────┘           │
│        │                                    │                    │
│        ▼                                    ▼                    │
│   ┌─────────┐                         ┌──────────────┐          │
│   │  AuthZ  │                         │    etcd      │          │
│   │ (RBAC)  │                         │   (persist)  │          │
│   └─────────┘                         └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

#### Example: Interacting with the API Server

```bash
# All kubectl commands go through the API Server
kubectl get pods
kubectl create deployment nginx --image=nginx

# Direct API call
curl -k https://<api-server>:6443/api/v1/namespaces/default/pods \
  --header "Authorization: Bearer $TOKEN"
```

---

### 2. etcd

**etcd** is a consistent, distributed key-value store that serves as Kubernetes' backing store for all cluster data.

#### Key Characteristics:
- **Distributed**: Runs across multiple nodes for high availability
- **Consistent**: Uses Raft consensus algorithm
- **Secure**: Supports TLS for client-server and peer communication
- **Watchable**: Clients can subscribe to changes

#### What's Stored in etcd:

| Data Type | Example Keys |
|-----------|--------------|
| Pods | `/registry/pods/default/nginx-pod` |
| Services | `/registry/services/specs/default/my-service` |
| Deployments | `/registry/deployments/default/my-app` |
| Secrets | `/registry/secrets/default/my-secret` |
| ConfigMaps | `/registry/configmaps/default/app-config` |
| Nodes | `/registry/minions/node-1` |

#### etcd Cluster Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                      etcd CLUSTER (HA Setup)                     │
│                                                                  │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│   │   etcd-1    │◄──►│   etcd-2    │◄──►│   etcd-3    │         │
│   │  (Leader)   │    │ (Follower)  │    │ (Follower)  │         │
│   └─────────────┘    └─────────────┘    └─────────────┘         │
│         │                  │                   │                 │
│         └──────────────────┴───────────────────┘                 │
│                    Raft Consensus Protocol                       │
│                                                                  │
│   • Minimum 3 nodes for HA (tolerates 1 failure)                │
│   • Recommended: 5 nodes (tolerates 2 failures)                 │
│   • Always use odd numbers                                       │
└─────────────────────────────────────────────────────────────────┘
```

#### Interacting with etcd:

```bash
# Check etcd cluster health
etcdctl endpoint health

# List all keys (with prefix)
etcdctl get /registry --prefix --keys-only

# Backup etcd
etcdctl snapshot save backup.db

# Restore from backup
etcdctl snapshot restore backup.db
```

---

### 3. kube-scheduler

The **Scheduler** watches for newly created Pods that have no Node assigned and selects a Node for them to run on.

#### Scheduling Process:

```
┌─────────────────────────────────────────────────────────────────┐
│                     SCHEDULING WORKFLOW                          │
│                                                                  │
│   New Pod Created                                                │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    FILTERING PHASE                       │   │
│   │  Remove nodes that don't meet requirements               │   │
│   │                                                          │   │
│   │  • NodeSelector/NodeAffinity                             │   │
│   │  • Taints and Tolerations                                │   │
│   │  • Resource requests (CPU, Memory)                       │   │
│   │  • Port availability                                     │   │
│   │  • Volume constraints                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    SCORING PHASE                         │   │
│   │  Rank remaining nodes by preference                      │   │
│   │                                                          │   │
│   │  • Least requested resources                             │   │
│   │  • Balanced resource allocation                          │   │
│   │  • Pod affinity/anti-affinity                            │   │
│   │  • Node affinity preferences                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                         │
│        ▼                                                         │
│   Best Node Selected → Pod Bound to Node                         │
└─────────────────────────────────────────────────────────────────┘
```

#### Scheduling Factors:

| Factor | Description |
|--------|-------------|
| **Resource Requirements** | CPU and memory requests/limits |
| **Hardware/Software Constraints** | GPU, SSD, specific OS |
| **Affinity/Anti-affinity** | Co-locate or separate pods |
| **Taints & Tolerations** | Node restrictions and pod exceptions |
| **Data Locality** | Place pods near their data |
| **Deadlines** | Priority-based scheduling |

#### Example: Pod with Scheduling Constraints

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-v100
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values:
            - us-west-1a
            - us-west-1b
```

---

### 4. kube-controller-manager

The **Controller Manager** runs controller processes. Each controller is a separate process, but they're compiled into a single binary and run as a single process.

#### Built-in Controllers:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTROLLER MANAGER                            │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Node Controller │  │ Replication    │  │  Endpoints      │  │
│  │                  │  │ Controller     │  │  Controller     │  │
│  │  • Monitor nodes │  │                │  │                 │  │
│  │  • Handle node   │  │ • Maintain     │  │ • Populate      │  │
│  │    failures      │  │   replica      │  │   Endpoints     │  │
│  │                  │  │   counts       │  │   objects       │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ ServiceAccount  │  │  Namespace     │  │  Job Controller │  │
│  │ Controller      │  │  Controller    │  │                 │  │
│  │                 │  │                │  │  • Track job    │  │
│  │ • Create default│  │ • Lifecycle of │  │    completion   │  │
│  │   accounts      │  │   namespaces   │  │  • Create pods  │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Deployment     │  │  StatefulSet   │  │  DaemonSet      │  │
│  │ Controller     │  │  Controller    │  │  Controller     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### Controller Pattern (Reconciliation Loop):

```go
// Simplified controller logic
for {
    desiredState := getDesiredState()  // From API Server (etcd)
    currentState := getCurrentState()  // From cluster observation
    
    if currentState != desiredState {
        reconcile(currentState, desiredState)  // Take action
    }
    
    wait(interval)
}
```

#### Key Controllers Explained:

| Controller | Function |
|------------|----------|
| **Node Controller** | Monitors node health, marks nodes as unavailable, evicts pods from failed nodes |
| **Replication Controller** | Ensures the specified number of pod replicas are running |
| **Deployment Controller** | Manages deployments, handles rolling updates and rollbacks |
| **StatefulSet Controller** | Manages stateful applications with stable identities |
| **DaemonSet Controller** | Ensures all/some nodes run a copy of a pod |
| **Job Controller** | Manages one-off tasks that run to completion |
| **CronJob Controller** | Creates Jobs on a schedule |

---

### 5. cloud-controller-manager

The **Cloud Controller Manager** contains cloud-provider-specific control logic. It lets you link your cluster to your cloud provider's API.

#### Cloud-Specific Controllers:

```
┌─────────────────────────────────────────────────────────────────┐
│                 CLOUD CONTROLLER MANAGER                         │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Node Controller                          │  │
│  │  • Check if node still exists in cloud after it stops     │  │
│  │    responding                                              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Route Controller                         │  │
│  │  • Set up routes in cloud infrastructure                  │  │
│  │  • Enable pod-to-pod communication across nodes           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                  Service Controller                        │  │
│  │  • Create, update, delete cloud load balancers            │  │
│  │  • For Services of type LoadBalancer                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Worker Node Components

Worker Nodes are the machines where your containerized applications actually run. Each node contains the services necessary to run Pods.

### 1. kubelet

The **kubelet** is the primary "node agent" that runs on each worker node. It ensures that containers are running in a Pod.

#### Responsibilities:

```
┌─────────────────────────────────────────────────────────────────┐
│                         KUBELET                                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Pod Lifecycle Management                 │   │
│   │  • Register node with API Server                        │   │
│   │  • Watch for Pod specs assigned to this node            │   │
│   │  • Create/start/stop containers via Container Runtime   │   │
│   │  • Mount volumes                                        │   │
│   │  • Report node and pod status back to API Server        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Health Checking                       │   │
│   │  • Liveness probes: Is container alive?                 │   │
│   │  • Readiness probes: Is container ready for traffic?    │   │
│   │  • Startup probes: Has container started?               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Resource Management                     │   │
│   │  • Enforce resource limits                              │   │
│   │  • cgroups integration                                  │   │
│   │  • Report node capacity and allocatable resources       │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### Kubelet Workflow:

```
API Server ──────► kubelet ──────► Container Runtime (CRI)
                      │                     │
                      │                     ▼
                      │              ┌─────────────┐
                      │              │ containerd  │
                      │              │    or       │
                      │              │  CRI-O      │
                      │              └─────────────┘
                      │                     │
                      ▼                     ▼
                 Pod Status          Container Created
                  Report
```

#### Example: Health Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-check-demo
spec:
  containers:
  - name: app
    image: my-app:v1
    
    # Is the container alive?
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      failureThreshold: 3
    
    # Is the container ready for traffic?
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    
    # Has the container started?
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

---

### 2. kube-proxy

The **kube-proxy** is a network proxy that runs on each node, implementing part of the Kubernetes Service concept.

#### Modes of Operation:

```
┌─────────────────────────────────────────────────────────────────┐
│                      KUBE-PROXY MODES                            │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    iptables Mode                           │  │
│  │                                                            │  │
│  │  • Default mode                                            │  │
│  │  • Uses iptables rules for packet redirection             │  │
│  │  • Random pod selection for load balancing                │  │
│  │  • Scales to ~5000 services                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     IPVS Mode                              │  │
│  │                                                            │  │
│  │  • Better performance at scale                            │  │
│  │  • Multiple load balancing algorithms                     │  │
│  │    (rr, lc, dh, sh, sed, nq)                              │  │
│  │  • Scales to 10,000+ services                             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   nftables Mode                            │  │
│  │                                                            │  │
│  │  • Modern replacement for iptables                        │  │
│  │  • Better performance and scalability                     │  │
│  │  • Available in newer Kubernetes versions                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### How kube-proxy Routes Traffic:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE ROUTING                               │
│                                                                  │
│   Client Request                                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────┐                                           │
│   │ Service VIP     │  10.96.0.100:80                           │
│   │ (ClusterIP)     │                                           │
│   └────────┬────────┘                                           │
│            │                                                     │
│            │  kube-proxy rules                                  │
│            │                                                     │
│            ▼                                                     │
│   ┌────────────────────────────────────────┐                    │
│   │         Load Balance to Pods            │                    │
│   │                                         │                    │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐ │                    │
│   │  │ Pod 1   │  │ Pod 2   │  │ Pod 3   │ │                    │
│   │  │10.1.1.5 │  │10.1.2.3 │  │10.1.3.7 │ │                    │
│   │  │ :8080   │  │ :8080   │  │ :8080   │ │                    │
│   │  └─────────┘  └─────────┘  └─────────┘ │                    │
│   └────────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3. Container Runtime

The **Container Runtime** is the software responsible for running containers. Kubernetes supports any runtime that implements the Container Runtime Interface (CRI).

#### Supported Container Runtimes:

| Runtime | Description |
|---------|-------------|
| **containerd** | Industry-standard runtime, graduated CNCF project, default in most K8s distributions |
| **CRI-O** | Lightweight runtime designed specifically for Kubernetes |
| **Docker Engine** | Via cri-dockerd adapter (Docker shim removed in K8s 1.24) |

#### Container Runtime Interface (CRI):

```
┌─────────────────────────────────────────────────────────────────┐
│                     CONTAINER RUNTIME INTERFACE                  │
│                                                                  │
│                         kubelet                                  │
│                            │                                     │
│                            │ CRI (gRPC)                         │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    CRI Runtime                           │   │
│   │                                                          │   │
│   │    containerd          CRI-O           cri-dockerd       │   │
│   │         │                │                  │            │   │
│   │         ▼                ▼                  ▼            │   │
│   │    ┌─────────┐      ┌─────────┐        ┌─────────┐      │   │
│   │    │  runc   │      │  runc   │        │ Docker  │      │   │
│   │    │         │      │ /crun   │        │ Engine  │      │   │
│   │    └─────────┘      └─────────┘        └─────────┘      │   │
│   │                                                          │   │
│   │    Low-level container runtime (OCI-compliant)          │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Communication Flow in Kubernetes

### How Components Communicate:

```
┌─────────────────────────────────────────────────────────────────┐
│                  KUBERNETES COMMUNICATION                        │
│                                                                  │
│                    ┌──────────────────┐                         │
│                    │    API Server    │                         │
│                    └────────┬─────────┘                         │
│                             │                                    │
│           ┌─────────────────┼─────────────────┐                 │
│           │                 │                 │                 │
│           ▼                 ▼                 ▼                 │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐            │
│   │ Scheduler │     │Controllers│     │   etcd    │            │
│   └───────────┘     └───────────┘     └───────────┘            │
│                                                                  │
│   ─────────────────────────────────────────────────────         │
│                             │                                    │
│           ┌─────────────────┼─────────────────┐                 │
│           │                 │                 │                 │
│           ▼                 ▼                 ▼                 │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐            │
│   │  kubelet  │     │  kubelet  │     │  kubelet  │            │
│   │  (Node 1) │     │  (Node 2) │     │  (Node 3) │            │
│   └───────────┘     └───────────┘     └───────────┘            │
│                                                                  │
│   Key Points:                                                   │
│   • All communication goes through API Server                   │
│   • Components watch for changes (watch API)                    │
│   • etcd is only accessed by API Server                        │
│   • TLS encrypts all communication                              │
└─────────────────────────────────────────────────────────────────┘
```

### Example: Deploying a Pod (Full Flow)

```
┌─────────────────────────────────────────────────────────────────┐
│                POD CREATION WORKFLOW                             │
│                                                                  │
│  1. User runs: kubectl apply -f pod.yaml                        │
│     │                                                            │
│     ▼                                                            │
│  2. kubectl sends request to API Server                         │
│     │                                                            │
│     ▼                                                            │
│  3. API Server authenticates, authorizes, validates             │
│     │                                                            │
│     ▼                                                            │
│  4. API Server writes Pod spec to etcd (Status: Pending)        │
│     │                                                            │
│     ▼                                                            │
│  5. Scheduler watches for unscheduled pods                      │
│     │  └─ Finds best node for the pod                           │
│     │  └─ Updates Pod with nodeName via API Server              │
│     ▼                                                            │
│  6. kubelet on target node watches for pods assigned to it      │
│     │  └─ Sees new pod assignment                               │
│     │  └─ Pulls container image                                 │
│     │  └─ Creates container via Container Runtime               │
│     ▼                                                            │
│  7. kubelet reports Pod status to API Server (Running)          │
│     │                                                            │
│     ▼                                                            │
│  8. API Server updates Pod status in etcd                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## High Availability Architecture

For production clusters, you need a highly available control plane:

```
┌─────────────────────────────────────────────────────────────────┐
│                HIGH AVAILABILITY SETUP                          │
│                                                                 │
│                    ┌──────────────────┐                         │
│                    │   Load Balancer  │                         │
│                    │  (HAProxy/Cloud) │                         │
│                    └────────┬─────────┘                         │
│           ┌─────────────────┼─────────────────┐                 │
│           │                 │                 │                 │
│           ▼                 ▼                 ▼                 │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐         │
│   │  Master 1     │ │  Master 2     │ │  Master 3     │         │
│   │               │ │               │ │               │         │
│   │ • API Server  │ │ • API Server  │ │ • API Server  │         │
│   │ • Scheduler   │ │ • Scheduler   │ │ • Scheduler   │         │
│   │ • Controller  │ │ • Controller  │ │ • Controller  │         │
│   │   Manager     │ │   Manager     │ │   Manager     │         │
│   │ • etcd        │ │ • etcd        │ │ • etcd        │         │
│   └───────────────┘ └───────────────┘ └───────────────┘         │
│                                                                 │
│   Notes:                                                        │
│   • API Server: All instances active (load balanced)            │
│   • Scheduler: Leader election (only 1 active)                  │
│   • Controller Manager: Leader election (only 1 active)         │
│   • etcd: Raft consensus (1 leader, N-1 followers)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Ports Reference

| Component | Default Port | Protocol | Purpose |
|-----------|--------------|----------|---------|
| API Server | 6443 | HTTPS | Kubernetes API |
| etcd | 2379-2380 | HTTPS | Client & peer communication |
| Scheduler | 10259 | HTTPS | Metrics/health |
| Controller Manager | 10257 | HTTPS | Metrics/health |
| kubelet | 10250 | HTTPS | API, metrics |
| kubelet (read-only) | 10255 | HTTP | Read-only API |
| kube-proxy | 10256 | HTTP | Health check |

---

## Summary

| Component | Location | Primary Function |
|-----------|----------|------------------|
| **API Server** | Control Plane | Central API gateway, authentication, authorization |
| **etcd** | Control Plane | Persistent storage of all cluster state |
| **Scheduler** | Control Plane | Assigns pods to nodes |
| **Controller Manager** | Control Plane | Maintains desired state (reconciliation) |
| **Cloud Controller** | Control Plane | Cloud provider integration |
| **kubelet** | Worker Node | Manages pod lifecycle on the node |
| **kube-proxy** | Worker Node | Network proxy for services |
| **Container Runtime** | Worker Node | Runs containers (containerd, CRI-O) |

Understanding Kubernetes architecture is fundamental to:
- Troubleshooting cluster issues
- Designing for high availability
- Securing your cluster
- Optimizing performance

---

## Next Steps

Now that you understand the architecture, head to **[Day 3: Pods, ReplicaSets, and Deployments](../day-3/README.md)** to start creating workloads!

---