# kubectl Cheat Sheet 📋

A quick reference for the most commonly used kubectl commands.

---

## Cluster & Context

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# Context management
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=<namespace>
```

---

## Getting Resources

```bash
# Basic get commands
kubectl get pods
kubectl get pods -o wide                    # More details
kubectl get pods -o yaml                    # YAML output
kubectl get pods -o json                    # JSON output
kubectl get pods -w                         # Watch for changes
kubectl get pods --show-labels              # Show labels
kubectl get pods -l app=nginx               # Filter by label
kubectl get pods -A                         # All namespaces
kubectl get pods -n <namespace>             # Specific namespace

# Get multiple resources
kubectl get pods,svc,deploy
kubectl get all
kubectl get all -A

# Common resources
kubectl get pods | po
kubectl get services | svc
kubectl get deployments | deploy
kubectl get replicasets | rs
kubectl get configmaps | cm
kubectl get secrets
kubectl get namespaces | ns
kubectl get nodes | no
kubectl get persistentvolumeclaims | pvc
kubectl get persistentvolumes | pv
kubectl get statefulsets | sts
kubectl get daemonsets | ds
kubectl get ingress | ing
kubectl get jobs
kubectl get cronjobs | cj
```

---

## Describing Resources

```bash
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe svc <service-name>
kubectl describe deploy <deployment-name>
```

---

## Creating Resources

```bash
# From YAML file
kubectl apply -f <file.yaml>
kubectl apply -f <directory>/
kubectl apply -f https://url/to/file.yaml

# Create vs Apply
kubectl create -f file.yaml     # Fails if exists
kubectl apply -f file.yaml      # Creates or updates

# Imperative creation
kubectl create namespace <name>
kubectl create deployment <name> --image=<image>
kubectl create service clusterip <name> --tcp=<port>:<target-port>
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value

# Quick pod creation
kubectl run <name> --image=<image>
kubectl run <name> --image=<image> --port=<port>
kubectl run <name> --image=<image> -- <command>
```

---

## Editing & Updating

```bash
# Edit in default editor
kubectl edit deployment <name>
kubectl edit svc <name>

# Update image
kubectl set image deployment/<name> <container>=<image>:<tag>

# Scale
kubectl scale deployment <name> --replicas=<count>
kubectl scale --replicas=3 -f deployment.yaml

# Autoscale
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=80

# Patch (JSON merge)
kubectl patch deployment <name> -p '{"spec":{"replicas":3}}'

# Rollout management
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=<n>
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
kubectl rollout restart deployment/<name>
```

---

## Deleting Resources

```bash
kubectl delete pod <name>
kubectl delete pod <name> --force --grace-period=0
kubectl delete -f <file.yaml>
kubectl delete deployment <name>
kubectl delete svc <name>
kubectl delete all --all -n <namespace>
kubectl delete namespace <name>
```

---

## Logs & Debugging

```bash
# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>      # Multi-container pod
kubectl logs <pod-name> --previous          # Previous container
kubectl logs -f <pod-name>                  # Follow/stream
kubectl logs --tail=100 <pod-name>          # Last 100 lines
kubectl logs --since=1h <pod-name>          # Last hour
kubectl logs -l app=nginx                   # By label
kubectl logs deployment/<name>              # Deployment logs

# Execute commands
kubectl exec <pod-name> -- <command>
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container> -- /bin/sh

# Port forwarding
kubectl port-forward pod/<name> <local>:<pod>
kubectl port-forward svc/<name> <local>:<svc>
kubectl port-forward deploy/<name> <local>:<container>

# Copy files
kubectl cp <pod>:<path> <local-path>
kubectl cp <local-path> <pod>:<path>
kubectl cp <pod>:<path> <local-path> -c <container>

# Debug pod
kubectl run debug --rm -it --image=busybox -- /bin/sh
kubectl run debug --rm -it --image=curlimages/curl -- sh
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
```

---

## Events & Troubleshooting

```bash
# Get events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -w                       # Watch
kubectl get events --field-selector type=Warning

# Resource usage
kubectl top nodes
kubectl top pods
kubectl top pods --containers
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# Check API resources
kubectl api-resources
kubectl api-versions
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
```

---

## Labels & Annotations

```bash
# Labels
kubectl label pod <name> key=value
kubectl label pod <name> key-               # Remove label
kubectl label pod <name> key=value --overwrite

# Annotations
kubectl annotate pod <name> key=value
kubectl annotate pod <name> key-            # Remove

# Select by label
kubectl get pods -l key=value
kubectl get pods -l 'key in (v1,v2)'
kubectl get pods -l key!=value
kubectl get pods -l key                     # Key exists
kubectl get pods -l '!key'                  # Key doesn't exist
```

---

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>

# Set default namespace
kubectl config set-context --current --namespace=<name>

# Temporary namespace
kubectl -n <namespace> get pods
kubectl --namespace=<namespace> get pods
```

---

## ConfigMaps & Secrets

```bash
# ConfigMaps
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=<path>
kubectl create configmap <name> --from-env-file=<path>
kubectl get configmap <name> -o yaml

# Secrets
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret generic <name> --from-file=<path>
kubectl create secret tls <name> --cert=<cert> --key=<key>
kubectl create secret docker-registry <name> \
  --docker-server=<server> \
  --docker-username=<user> \
  --docker-password=<pass>

# View secret (base64 decoded)
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d
```

---

## Services & Networking

```bash
# Expose deployment
kubectl expose deployment <name> --port=<port> --target-port=<target>
kubectl expose deployment <name> --type=NodePort --port=<port>
kubectl expose deployment <name> --type=LoadBalancer --port=<port>

# Get endpoints
kubectl get endpoints
kubectl get endpoints <service-name>

# DNS testing
kubectl run test --rm -it --image=busybox -- nslookup <service>
kubectl run test --rm -it --image=busybox -- wget -qO- http://<service>
```

---

## RBAC

```bash
# Check permissions
kubectl auth can-i create pods
kubectl auth can-i create pods --as <user>
kubectl auth can-i create pods --as system:serviceaccount:<ns>:<sa>
kubectl auth can-i --list

# Create role/binding
kubectl create role <name> --verb=get,list --resource=pods
kubectl create rolebinding <name> --role=<role> --user=<user>
kubectl create clusterrole <name> --verb=get,list --resource=pods
kubectl create clusterrolebinding <name> --clusterrole=<role> --user=<user>
```

---

## Dry Run & Diff

```bash
# Dry run (client-side)
kubectl apply -f file.yaml --dry-run=client

# Dry run (server-side)
kubectl apply -f file.yaml --dry-run=server

# Generate YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Diff before apply
kubectl diff -f file.yaml
```

---

## Output Formatting

```bash
# Output formats
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
kubectl get node -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Sort
kubectl get pods --sort-by='.metadata.creationTimestamp'
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
```

---

## Quick Reference Table

| Resource | Short | API Group |
|----------|-------|-----------|
| pods | po | core |
| services | svc | core |
| deployments | deploy | apps |
| replicasets | rs | apps |
| statefulsets | sts | apps |
| daemonsets | ds | apps |
| configmaps | cm | core |
| secrets | - | core |
| namespaces | ns | core |
| nodes | no | core |
| persistentvolumeclaims | pvc | core |
| persistentvolumes | pv | core |
| ingresses | ing | networking.k8s.io |
| networkpolicies | netpol | networking.k8s.io |
| jobs | - | batch |
| cronjobs | cj | batch |
| serviceaccounts | sa | core |
| roles | - | rbac.authorization.k8s.io |
| rolebindings | - | rbac.authorization.k8s.io |
| clusterroles | - | rbac.authorization.k8s.io |
| clusterrolebindings | - | rbac.authorization.k8s.io |

---

## Common Patterns

### Deploy an app quickly
```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

### Debug a failing pod
```bash
kubectl describe pod <name>
kubectl logs <name> --previous
kubectl get events --field-selector involvedObject.name=<name>
```

### Execute into a running container
```bash
kubectl exec -it <pod> -- /bin/sh
```

### Get all resources in namespace
```bash
kubectl get all -n <namespace>
```

### Force delete stuck pod
```bash
kubectl delete pod <name> --force --grace-period=0
```

### Watch resources
```bash
kubectl get pods -w
kubectl get events -w
```

---

**Tip**: Create aliases for frequently used commands:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias ka='kubectl apply -f'
alias kx='kubectl delete'
```

---

📚 **Full Documentation**: [kubernetes.io/docs/reference/kubectl/](https://kubernetes.io/docs/reference/kubectl/)
