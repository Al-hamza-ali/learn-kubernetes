# Day 11: RBAC and Security Basics

## Overview

Kubernetes provides robust security controls. **RBAC (Role-Based Access Control)** controls who can do what in your cluster. Understanding security is crucial for production deployments.

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES SECURITY                           │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     API Server                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│           ┌────────────────┼────────────────┐                   │
│           ▼                ▼                ▼                   │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐        │
│   │ Authentication│ │ Authorization │ │  Admission    │        │
│   │               │ │    (RBAC)     │ │   Control     │        │
│   │ "Who are you?"│ │ "What can you │ │ "Is this      │        │
│   │               │ │     do?"      │ │  allowed?"    │        │
│   └───────────────┘ └───────────────┘ └───────────────┘        │
│         │                  │                  │                  │
│         ▼                  ▼                  ▼                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Request Allowed                        │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Authentication

### Authentication Methods

| Method | Description |
|--------|-------------|
| **Client Certificates** | X.509 certificates signed by cluster CA |
| **Bearer Tokens** | Static tokens or ServiceAccount tokens |
| **OpenID Connect** | OAuth2/OIDC providers (Google, Azure AD) |
| **Webhook** | External authentication service |

### Service Accounts

Every pod runs with a ServiceAccount that provides its identity:

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
automountServiceAccountToken: true
```

```yaml
# Using ServiceAccount in Pod
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: myapp:1.0
```

---

## RBAC Components

```
┌─────────────────────────────────────────────────────────────────┐
│                       RBAC MODEL                                 │
│                                                                  │
│   WHO (Subject)          WHAT (Rules)         WHERE (Scope)     │
│   ┌─────────────┐       ┌─────────────┐                        │
│   │    User     │       │    Role     │────► Namespace         │
│   │   Group     │──────►│ ClusterRole │────► Cluster-wide      │
│   │ServiceAcct  │       └─────────────┘                        │
│   └─────────────┘              │                                │
│                                │                                │
│                                ▼                                │
│                        ┌─────────────┐                         │
│                        │  Binding    │                         │
│                        │RoleBinding  │────► Namespace          │
│                        │ClusterRole  │────► Cluster-wide       │
│                        │  Binding    │                         │
│                        └─────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

### Role and ClusterRole

- **Role**: Grants permissions within a namespace
- **ClusterRole**: Grants cluster-wide permissions or across namespaces

### RoleBinding and ClusterRoleBinding

- **RoleBinding**: Binds Role/ClusterRole to subjects in a namespace
- **ClusterRoleBinding**: Binds ClusterRole to subjects cluster-wide

---

## Creating Roles and RoleBindings

### Example 1: Read-Only Access to Pods

```yaml
# role-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```yaml
# rolebinding-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-service-account
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Example 2: Deployment Manager

```yaml
# role-deployment-manager.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### Example 3: ClusterRole for Node Access

```yaml
# clusterrole-node-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["nodes/status"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Common Verbs

| Verb | Description |
|------|-------------|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create resources |
| `update` | Update resources |
| `patch` | Partially update resources |
| `delete` | Delete resources |
| `deletecollection` | Delete multiple resources |

### API Groups

| API Group | Resources |
|-----------|-----------|
| `""` (core) | pods, services, secrets, configmaps, nodes |
| `apps` | deployments, statefulsets, daemonsets, replicasets |
| `batch` | jobs, cronjobs |
| `networking.k8s.io` | ingresses, networkpolicies |
| `rbac.authorization.k8s.io` | roles, rolebindings, clusterroles |

---

## Default ClusterRoles

Kubernetes includes built-in ClusterRoles:

| ClusterRole | Permissions |
|-------------|-------------|
| `cluster-admin` | Full access to everything |
| `admin` | Full access within namespace |
| `edit` | Read/write most resources |
| `view` | Read-only access |

### Using Default ClusterRoles

```yaml
# Give user "alice" admin access to "production" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-admin
  namespace: production
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole      # Using ClusterRole as Role
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

---

## ServiceAccount for Applications

### Example: Application with Specific Permissions

```yaml
# app-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"]    # Specific secret only
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: config-reader
  namespace: default
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: config-reader
      containers:
      - name: app
        image: myapp:1.0
```

---

## Security Contexts

Control pod and container security settings:

### Pod Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

### Security Context Options

| Field | Description |
|-------|-------------|
| `runAsNonRoot` | Must run as non-root user |
| `runAsUser` | Specific user ID |
| `runAsGroup` | Specific group ID |
| `fsGroup` | Group for volume ownership |
| `readOnlyRootFilesystem` | Immutable root filesystem |
| `allowPrivilegeEscalation` | Prevent privilege escalation |
| `capabilities` | Linux capabilities to add/drop |

---

## Network Policies

Control pod-to-pod communication:

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Default Deny All

```yaml
# deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow DNS

```yaml
# allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

---

## Pod Security Standards

Kubernetes defines three security profiles:

| Profile | Description |
|---------|-------------|
| **Privileged** | Unrestricted, for trusted workloads |
| **Baseline** | Minimal restrictions, prevents known escalations |
| **Restricted** | Heavily restricted, security best practices |

### Enforce with Namespace Labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Commands Reference

```bash
# Check permissions
kubectl auth can-i create pods
kubectl auth can-i create pods --as jane
kubectl auth can-i create pods --as system:serviceaccount:default:mysa
kubectl auth can-i --list --as jane

# View roles and bindings
kubectl get roles -n development
kubectl get rolebindings -n development
kubectl get clusterroles
kubectl get clusterrolebindings

# Describe role
kubectl describe role pod-reader -n development

# Create role imperatively
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev

# Create rolebinding imperatively
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n dev

# ServiceAccounts
kubectl get serviceaccounts
kubectl create serviceaccount my-sa
kubectl describe serviceaccount my-sa
```

---

## Best Practices

### 1. Principle of Least Privilege
```yaml
# Only grant necessary permissions
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]    # Not "delete"!
```

### 2. Use ResourceNames for Specific Resources
```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["my-specific-secret"]
  verbs: ["get"]
```

### 3. Use Groups Instead of Users
```yaml
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
```

### 4. Regularly Audit Permissions
```bash
kubectl auth can-i --list --as system:serviceaccount:default:mysa
```

### 5. Don't Use Default ServiceAccount
```yaml
# Create dedicated ServiceAccounts
spec:
  serviceAccountName: my-app-sa
```

### 6. Enable Pod Security Standards
```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

---

## Summary

| Concept | Scope | Purpose |
|---------|-------|---------|
| **Role** | Namespace | Define permissions |
| **ClusterRole** | Cluster | Define cluster-wide permissions |
| **RoleBinding** | Namespace | Bind Role to subjects |
| **ClusterRoleBinding** | Cluster | Bind ClusterRole cluster-wide |
| **ServiceAccount** | Namespace | Pod identity |
| **SecurityContext** | Pod/Container | Runtime security |
| **NetworkPolicy** | Namespace | Network isolation |

### Key Takeaways

1. **RBAC controls authorization** — who can do what
2. **Use least privilege** — only grant what's needed
3. **Create dedicated ServiceAccounts** for applications
4. **Use SecurityContexts** for runtime security
5. **Implement NetworkPolicies** for network isolation
6. **Enable Pod Security Standards** in production

---

## Next Steps

Simplify deployments with **[Day 12: Helm Package Manager](../day-12/README.md)**!
