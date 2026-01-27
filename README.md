# Learn Kubernetes - 14-Day Guide 🚀

A comprehensive, beginner-friendly Kubernetes learning path. Each day covers a specific topic with detailed explanations, diagrams, examples, and hands-on exercises.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ██╗  ██╗ █████╗ ███████╗                                      │
│   ██║ ██╔╝██╔══██╗██╔════╝                                      │
│   █████╔╝ ╚█████╔╝███████╗                                      │
│   ██╔═██╗ ██╔══██╗╚════██║                                      │
│   ██║  ██╗╚█████╔╝███████║                                      │
│   ╚═╝  ╚═╝ ╚════╝ ╚══════╝                                      │
│                                                                  │
│   Kubernetes Learning Path                                       │
│   From Zero to Hero in 14 Days                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 📚 Course Structure

### Week 1: Foundations

| Day | Topic | Description |
|-----|-------|-------------|
| [Day 1](./day-1/README.md) | **Introduction to Kubernetes** | What is K8s, why use it, key concepts and advantages |
| [Day 2](./day-2/README.md) | **Kubernetes Architecture** | Control plane, worker nodes, components deep dive |
| [Day 3](./day-3/README.md) | **Pods, ReplicaSets, Deployments** | Core workload resources with examples |
| [Day 4](./day-4/README.md) | **Services** | ClusterIP, NodePort, LoadBalancer, service discovery |
| [Day 5](./day-5/README.md) | **Ingress** | HTTP routing, TLS, ingress controllers |
| [Day 6](./day-6/README.md) | **ConfigMaps & Secrets** | Configuration and secret management |
| [Day 7](./day-7/README.md) | **Volumes & Storage** | Persistent storage, PV, PVC, StorageClasses |

### Week 2: Advanced Topics

| Day | Topic | Description |
|-----|-------|-------------|
| [Day 8](./day-8/README.md) | **Namespaces & Resources** | Resource quotas, limit ranges, QoS |
| [Day 9](./day-9/README.md) | **Jobs & CronJobs** | Batch processing and scheduled tasks |
| [Day 10](./day-10/README.md) | **StatefulSets & DaemonSets** | Stateful apps and node-level daemons |
| [Day 11](./day-11/README.md) | **RBAC & Security** | Authorization, security contexts, network policies |
| [Day 12](./day-12/README.md) | **Helm** | Package management for Kubernetes |
| [Day 13](./day-13/README.md) | **Monitoring & Logging** | Prometheus, Grafana, logging best practices |
| [Day 14](./day-14/README.md) | **Troubleshooting** | Common issues and debugging techniques |

### 📋 Quick Reference

| Resource | Description |
|----------|-------------|
| [kubectl Cheat Sheet](./cheatsheet.md) | Complete command reference for daily use |

---

## 🎯 Learning Objectives

By the end of this course, you will be able to:

- ✅ Understand Kubernetes architecture and components
- ✅ Deploy and manage containerized applications
- ✅ Configure networking with Services and Ingress
- ✅ Manage configuration and secrets securely
- ✅ Implement persistent storage solutions
- ✅ Set up resource quotas and limits
- ✅ Run batch jobs and scheduled tasks
- ✅ Deploy stateful applications
- ✅ Implement RBAC and security best practices
- ✅ Use Helm for package management
- ✅ Set up monitoring and logging
- ✅ Troubleshoot common issues

---

## 🛠 Prerequisites

Before starting, you should have:

- Basic understanding of containers (Docker)
- Command line experience
- A local Kubernetes cluster (see setup below)

### Setting Up a Local Cluster

```bash
# Option 1: Minikube
brew install minikube    # macOS
minikube start

# Option 2: Docker Desktop
# Enable Kubernetes in Docker Desktop settings

# Option 3: Kind (Kubernetes in Docker)
brew install kind
kind create cluster

# Verify installation
kubectl cluster-info
kubectl get nodes
```

---

## 📖 How to Use This Guide

### For Self-Paced Learning

1. **Follow in order** - Each day builds on previous concepts
2. **Read the theory** - Understand the "why" before the "how"
3. **Type the examples** - Don't just copy-paste, type them out
4. **Experiment** - Modify examples and see what happens
5. **Use the commands** - Practice kubectl commands regularly

### Daily Study Plan

```
┌─────────────────────────────────────────────────────────────────┐
│                    RECOMMENDED DAILY SCHEDULE                    │
│                                                                  │
│   📖 30 min - Read the day's content                            │
│   💻 30 min - Follow along with examples                        │
│   🧪 30 min - Experiment and practice                           │
│   📝 15 min - Review and take notes                             │
│                                                                  │
│   Total: ~2 hours per day                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Quick Reference

### Essential Commands

```bash
# Cluster Info
kubectl cluster-info
kubectl get nodes

# Workloads
kubectl get pods [-n namespace]
kubectl get deployments
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Services
kubectl get services
kubectl get endpoints

# Configuration
kubectl get configmaps
kubectl get secrets

# Debugging
kubectl exec -it <pod> -- /bin/sh
kubectl port-forward <pod> 8080:80
kubectl get events --sort-by='.lastTimestamp'
```

### Resource Shortcuts

| Full Name | Short |
|-----------|-------|
| pods | po |
| services | svc |
| deployments | deploy |
| replicasets | rs |
| configmaps | cm |
| namespaces | ns |
| persistentvolumeclaims | pvc |
| persistentvolumes | pv |
| statefulsets | sts |
| daemonsets | ds |
| ingresses | ing |

---

## 🔗 Additional Resources

### Official Documentation
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### Interactive Learning
- [Kubernetes Playground](https://www.katacoda.com/courses/kubernetes)
- [Play with Kubernetes](https://labs.play-with-k8s.com/)

### Certification
- [CKA - Certified Kubernetes Administrator](https://www.cncf.io/certification/cka/)
- [CKAD - Certified Kubernetes Application Developer](https://www.cncf.io/certification/ckad/)

---

## 📁 Repository Structure

```
learn-kubernetes/
├── README.md                 # This file
├── cheatsheet.md            # kubectl Quick Reference
├── day-1/README.md          # Introduction to Kubernetes
├── day-2/README.md          # Architecture
├── day-3/README.md          # Pods, ReplicaSets, Deployments
├── day-4/README.md          # Services
├── day-5/README.md          # Ingress
├── day-6/README.md          # ConfigMaps & Secrets
├── day-7/README.md          # Volumes & Storage
├── day-8/README.md          # Namespaces & Resources
├── day-9/README.md          # Jobs & CronJobs
├── day-10/README.md         # StatefulSets & DaemonSets
├── day-11/README.md         # RBAC & Security
├── day-12/README.md         # Helm
├── day-13/README.md         # Monitoring & Logging
└── day-14/README.md         # Troubleshooting
```

---

## 🚀 Getting Started

Ready to begin? Start with [Day 1: Introduction to Kubernetes](./day-1/README.md)!

---

## 📝 Notes

- All YAML examples are ready to use - just `kubectl apply -f`
- Each day includes practical examples and best practices
- Diagrams use ASCII art for universal compatibility
- Commands are tested on Kubernetes 1.28+

---

**Happy Learning! 🎉**

*If you find this helpful, consider starring the repo!*
