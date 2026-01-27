# Day 14: Troubleshooting Guide

## Overview

Troubleshooting Kubernetes requires a systematic approach. This guide covers common issues and how to diagnose them.

```
┌─────────────────────────────────────────────────────────────────┐
│                TROUBLESHOOTING WORKFLOW                          │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  1. Identify the Problem                                 │   │
│   │     What's not working? Pod? Service? Ingress?          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  2. Gather Information                                   │   │
│   │     kubectl describe, logs, events                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  3. Analyze and Diagnose                                 │   │
│   │     Understand the error messages                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  4. Fix and Verify                                       │   │
│   │     Apply fix, confirm resolution                        │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Essential Commands

### Information Gathering

```bash
# Cluster status
kubectl cluster-info
kubectl get nodes
kubectl get componentstatuses

# Get everything
kubectl get all
kubectl get all -A

# Detailed information
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe service <svc-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>
kubectl logs <pod-name> --previous
kubectl logs -f <pod-name>
kubectl logs --tail=100 <pod-name>

# Events (sorted by time)
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -w

# Resource usage
kubectl top nodes
kubectl top pods
```

### Interactive Debugging

```bash
# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- /bin/bash

# Port forwarding
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<svc-name> 8080:80

# Run debug pod
kubectl run debug --rm -it --image=busybox -- /bin/sh
kubectl run debug --rm -it --image=curlimages/curl -- /bin/sh
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash
```

---

## Pod Issues

### Pod States and Meanings

| State | Meaning |
|-------|---------|
| **Pending** | Pod accepted but not scheduled |
| **ContainerCreating** | Pulling image, creating container |
| **Running** | At least one container running |
| **Completed** | All containers exited with code 0 |
| **Error** | Container exited with error |
| **CrashLoopBackOff** | Container keeps crashing |
| **ImagePullBackOff** | Cannot pull container image |
| **ErrImagePull** | Error pulling image |

### CrashLoopBackOff

**Symptoms**: Pod keeps restarting

```bash
# Check what's happening
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous    # Logs from crashed container
```

**Common Causes**:
1. **Application error** - Check logs
2. **Missing configuration** - ConfigMap/Secret not mounted
3. **Health check failing** - Probe configuration wrong
4. **Resource limits** - OOMKilled

```bash
# Check for OOMKilled
kubectl describe pod <pod-name> | grep -A5 "Last State"
# Look for: Reason: OOMKilled
```

**Solutions**:
```yaml
# Increase memory limit
resources:
  limits:
    memory: "512Mi"   # Increase this

# Fix liveness probe
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60   # Give app time to start
  periodSeconds: 10
```

### ImagePullBackOff

**Symptoms**: Pod stuck, cannot pull image

```bash
kubectl describe pod <pod-name>
# Look for: Failed to pull image
```

**Common Causes**:
1. **Wrong image name/tag**
2. **Image doesn't exist**
3. **Private registry without credentials**
4. **Network issues**

**Solutions**:
```yaml
# Verify image exists
docker pull <image-name>

# Add imagePullSecrets for private registry
spec:
  imagePullSecrets:
  - name: my-registry-secret
  containers:
  - name: app
    image: private-registry.example.com/myapp:1.0
```

### Pending Pod

**Symptoms**: Pod stays in Pending state

```bash
kubectl describe pod <pod-name>
# Check Events section
```

**Common Causes**:

| Event Message | Cause | Solution |
|---------------|-------|----------|
| `Insufficient cpu` | Not enough CPU | Scale cluster or reduce requests |
| `Insufficient memory` | Not enough memory | Scale cluster or reduce requests |
| `No nodes available` | No matching nodes | Check node labels/taints |
| `PersistentVolumeClaim not found` | PVC doesn't exist | Create PVC |
| `node(s) didn't match node selector` | Wrong nodeSelector | Check labels |

**Solutions**:
```bash
# Check node resources
kubectl describe nodes | grep -A5 "Allocated resources"

# Check node taints
kubectl describe nodes | grep Taints

# Check if PVC is bound
kubectl get pvc
```

---

## Service Issues

### Service Not Accessible

```
┌─────────────────────────────────────────────────────────────────┐
│                 SERVICE DEBUGGING FLOW                           │
│                                                                  │
│   Can you reach the Service?                                    │
│         │                                                        │
│         ├─── NO ──► Check Service exists and has endpoints      │
│         │                                                        │
│         └─── YES ─► Can you reach the Pods directly?            │
│                           │                                      │
│                           ├─── NO ──► Check Pod health          │
│                           │                                      │
│                           └─── YES ─► Service config is wrong   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Check service
kubectl get svc <svc-name>
kubectl describe svc <svc-name>

# Check endpoints (should list pod IPs)
kubectl get endpoints <svc-name>

# If endpoints empty, check selector matches
kubectl get pods --show-labels
kubectl get svc <svc-name> -o yaml | grep -A5 selector

# Test from within cluster
kubectl run debug --rm -it --image=busybox -- /bin/sh
wget -qO- http://<service-name>:<port>
nslookup <service-name>
```

**Common Causes**:
1. **No endpoints** - Selector doesn't match pods
2. **Wrong port** - Service port doesn't match container port
3. **Pods not ready** - Readiness probe failing

### DNS Issues

```bash
# Test DNS resolution
kubectl run debug --rm -it --image=busybox -- nslookup kubernetes
kubectl run debug --rm -it --image=busybox -- nslookup <service-name>

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# DNS format
<service>.<namespace>.svc.cluster.local
```

---

## Ingress Issues

### Ingress Not Working

```bash
# Check ingress
kubectl get ingress
kubectl describe ingress <ingress-name>

# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verify backend service works
kubectl get svc
kubectl get endpoints <backend-service>

# Test from ingress controller
kubectl exec -it -n ingress-nginx <controller-pod> -- curl http://localhost/healthz
```

**Checklist**:
1. ✅ Ingress Controller installed
2. ✅ Ingress has correct ingressClassName
3. ✅ Backend service exists and has endpoints
4. ✅ Service port matches ingress backend port
5. ✅ DNS/hosts points to ingress IP

---

## Node Issues

### Node Not Ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check conditions
kubectl get nodes -o json | jq '.items[].status.conditions'

# Common conditions
# - Ready: True = healthy
# - MemoryPressure: True = low memory
# - DiskPressure: True = low disk
# - PIDPressure: True = too many processes
# - NetworkUnavailable: True = network issues
```

**Common Causes**:
1. **Kubelet not running** - Check kubelet service
2. **Network plugin issues** - Check CNI pods
3. **Resource exhaustion** - Disk/memory full
4. **Connectivity** - Cannot reach API server

```bash
# On the node (SSH in):
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

---

## Storage Issues

### PVC Not Binding

```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc <pvc-name>

# Check PVs
kubectl get pv

# Check storage class
kubectl get storageclass
```

**Common Causes**:
1. **No matching PV** - Size/access mode mismatch
2. **StorageClass not found** - Wrong storageClassName
3. **Provisioner not running** - Check cloud provider CSI

```yaml
# Check PVC spec
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  accessModes:
    - ReadWriteOnce     # Must match PV
  resources:
    requests:
      storage: 10Gi     # Must be <= PV size
  storageClassName: standard  # Must exist
```

---

## Network Debugging

### Pod-to-Pod Connectivity

```bash
# Get pod IPs
kubectl get pods -o wide

# Test connectivity from one pod to another
kubectl exec -it <pod-a> -- ping <pod-b-ip>
kubectl exec -it <pod-a> -- curl http://<pod-b-ip>:8080

# Check network policies
kubectl get networkpolicies -A
```

### Using netshoot for Debugging

```bash
# Run netshoot (contains many network tools)
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash

# Inside netshoot:
nslookup my-service
curl http://my-service:80
ping <pod-ip>
traceroute <pod-ip>
netstat -tulpn
tcpdump -i eth0
```

---

## Resource Issues

### OOMKilled

```bash
# Check for OOM events
kubectl describe pod <pod-name> | grep -i oom

# Check resource usage
kubectl top pods

# Compare with limits
kubectl get pod <pod-name> -o yaml | grep -A10 resources
```

**Solution**: Increase memory limits or optimize application

### CPU Throttling

```bash
# Check CPU usage vs limits
kubectl top pods

# Signs of throttling:
# - High latency
# - Slow responses
# - CPU usage at limit
```

**Solution**: Increase CPU limits or optimize application

---

## Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `CrashLoopBackOff` | App keeps crashing | Check logs, fix app |
| `ImagePullBackOff` | Cannot pull image | Verify image, add secrets |
| `Pending` | Cannot schedule | Check resources, taints |
| `OOMKilled` | Out of memory | Increase memory limit |
| `Error: ImagePull` | Wrong image name | Fix image reference |
| `FailedMount` | Volume mount failed | Check PVC, secrets |
| `FailedScheduling` | No suitable node | Check requirements |
| `Evicted` | Node resource pressure | Increase resources |
| `CreateContainerConfigError` | Config error | Check ConfigMaps/Secrets |

---

## Quick Reference Cheat Sheet

```bash
# === POD DEBUGGING ===
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- /bin/sh
kubectl get events --field-selector involvedObject.name=<pod>

# === SERVICE DEBUGGING ===
kubectl get svc,endpoints
kubectl describe svc <svc>
kubectl run test --rm -it --image=busybox -- wget -qO- http://<svc>

# === NODE DEBUGGING ===
kubectl get nodes
kubectl describe node <node>
kubectl top nodes

# === GENERAL ===
kubectl get events --sort-by='.lastTimestamp'
kubectl api-resources
kubectl explain pod.spec.containers

# === CLEANUP ===
kubectl delete pod <pod> --force --grace-period=0
kubectl delete all --all -n <namespace>
```

---

## Debugging Decision Tree

```
Problem: Application Not Working
│
├── Is Pod running?
│   ├── No → Check: kubectl describe pod
│   │   ├── ImagePullBackOff → Check image name, registry secrets
│   │   ├── Pending → Check resources, node selector, taints
│   │   ├── CrashLoopBackOff → Check logs, app config
│   │   └── ContainerCreating → Check PVC, ConfigMaps, Secrets
│   │
│   └── Yes → Is app responding in pod?
│       ├── No → kubectl exec + curl localhost
│       │   └── App issue → Check logs, app config
│       │
│       └── Yes → Can you reach via Service?
│           ├── No → Check service selector, endpoints
│           │
│           └── Yes → Can you reach via Ingress?
│               └── No → Check ingress, ingress controller
```

---

## Best Practices

### 1. Use Proper Labels
```yaml
metadata:
  labels:
    app: myapp
    version: v1.0
    environment: production
```

### 2. Always Set Resource Requests/Limits
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### 3. Configure Health Probes
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

### 4. Use Namespaces
```bash
kubectl create namespace production
kubectl apply -f app.yaml -n production
```

### 5. Check Events First
```bash
kubectl get events --sort-by='.lastTimestamp' -n <namespace>
```

---

## Summary

| Area | Key Commands |
|------|--------------|
| **Pods** | `describe`, `logs`, `exec` |
| **Services** | `get endpoints`, DNS test |
| **Nodes** | `describe node`, `top nodes` |
| **Network** | `netshoot`, `curl`, `nslookup` |
| **Storage** | `get pvc`, `describe pvc` |
| **Events** | `get events --sort-by=lastTimestamp` |

### Key Takeaways

1. **Always start with `kubectl describe`**
2. **Check events for recent issues**
3. **Use logs to understand app behavior**
4. **Verify endpoints for service issues**
5. **Use debug pods for network testing**
6. **Check resources for scheduling issues**

---

## 🎉 Congratulations!

You've completed the 14-day Kubernetes learning path! You now have a solid foundation in:

- ✅ Kubernetes architecture and core concepts
- ✅ Deploying and managing applications
- ✅ Networking with Services and Ingress
- ✅ Configuration and secret management
- ✅ Persistent storage
- ✅ Resource management
- ✅ Batch processing
- ✅ Stateful workloads
- ✅ Security best practices
- ✅ Helm package management
- ✅ Monitoring and logging
- ✅ Troubleshooting techniques

### What's Next?

1. **Practice** — Build and deploy your own applications
2. **Certify** — Consider CKA or CKAD certification
3. **Explore** — Dive deeper into service mesh, GitOps, operators
4. **Contribute** — Join the Kubernetes community!

Keep the **[kubectl Cheat Sheet](../cheatsheet.md)** handy for quick reference.

Happy Kuberneting! 🚀
