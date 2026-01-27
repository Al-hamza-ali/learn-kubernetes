# Day 5: Ingress Controllers and Resources

## Overview

While Services expose applications within or outside the cluster, **Ingress** provides a more sophisticated way to route external HTTP/HTTPS traffic to your services. Think of Ingress as a smart reverse proxy that sits at the edge of your cluster.

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHY INGRESS?                                   │
│                                                                  │
│   Without Ingress (multiple LoadBalancers):                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   api.example.com ──► LoadBalancer $$ ──► API Service   │   │
│   │   web.example.com ──► LoadBalancer $$ ──► Web Service   │   │
│   │   admin.example.com ─► LoadBalancer $$ ──► Admin Svc    │   │
│   │                                                          │   │
│   │   💰 Cost: 3 Load Balancers = $$$                        │   │
│   │   🔒 SSL: Managed separately for each                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   With Ingress (single entry point):                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   api.example.com  ─┐                                    │   │
│   │   web.example.com  ─┼──► Ingress ──► ClusterIP Services │   │
│   │   admin.example.com ┘    (1 LB)                          │   │
│   │                                                          │   │
│   │   💰 Cost: 1 Load Balancer = $                           │   │
│   │   🔒 SSL: Centralized TLS termination                    │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ingress vs Service

| Feature | Service (LoadBalancer) | Ingress |
|---------|------------------------|---------|
| **Layer** | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| **Routing** | IP-based only | Host and path-based |
| **SSL/TLS** | Per-service | Centralized |
| **Cost** | One LB per service | One LB for all |
| **Features** | Basic load balancing | URL rewriting, auth, rate limiting |

---

## Ingress Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    INGRESS ARCHITECTURE                          │
│                                                                  │
│   Internet                                                       │
│       │                                                          │
│       ▼                                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Cloud Load Balancer                         │   │
│   │              (External IP: 203.0.113.50)                │   │
│   └───────────────────────┬─────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  INGRESS CONTROLLER                      │   │
│   │            (NGINX, Traefik, HAProxy, etc.)              │   │
│   │                                                          │   │
│   │   Watches for:                                          │   │
│   │   • Ingress resources                                   │   │
│   │   • Services                                            │   │
│   │   • Endpoints                                           │   │
│   └───────────────────────┬─────────────────────────────────┘   │
│                           │                                      │
│                           │ Routes based on rules               │
│                           │                                      │
│           ┌───────────────┼───────────────┐                     │
│           │               │               │                     │
│           ▼               ▼               ▼                     │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐        │
│   │  API Service  │ │  Web Service  │ │ Admin Service │        │
│   │  /api/*       │ │  /           │ │  /admin/*     │        │
│   └───────────────┘ └───────────────┘ └───────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Ingress Controller

The **Ingress Controller** is a pod that runs a reverse proxy (like NGINX). It's NOT included by default — you must install one.

Popular Ingress Controllers:

| Controller | Description |
|------------|-------------|
| **NGINX Ingress** | Most popular, maintained by Kubernetes community |
| **Traefik** | Cloud-native, auto-discovery, great dashboard |
| **HAProxy** | High performance, enterprise features |
| **Contour** | Envoy-based, by VMware |
| **Kong** | API Gateway features built-in |
| **AWS ALB** | Native AWS Application Load Balancer |

### 2. Ingress Resource

The **Ingress Resource** is a YAML configuration that defines routing rules.

---

## Installing NGINX Ingress Controller

### Using Helm (Recommended)

```bash
# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install the controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Using kubectl

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml
```

### Verify Installation

```bash
# Check pods
kubectl get pods -n ingress-nginx

# Check service (get external IP)
kubectl get svc -n ingress-nginx

# Wait for external IP
kubectl get svc ingress-nginx-controller -n ingress-nginx -w
```

---

## Basic Ingress Examples

### Example 1: Simple Single Service Ingress

```yaml
# simple-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### Example 2: Path-Based Routing

Route different paths to different services:

```yaml
# path-based-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    PATH-BASED ROUTING                            │
│                                                                  │
│   myapp.example.com/api/*    ──────►  api-service:8080          │
│   myapp.example.com/web/*    ──────►  web-service:80            │
│   myapp.example.com/*        ──────►  frontend-service:80       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Example 3: Host-Based Routing (Virtual Hosts)

Route different domains to different services:

```yaml
# host-based-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOST-BASED ROUTING                            │
│                                                                  │
│   api.example.com      ──────►  api-service:8080                │
│   web.example.com      ──────►  web-service:80                  │
│   admin.example.com    ──────►  admin-service:8080              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Path Types

| PathType | Description | Example Match |
|----------|-------------|---------------|
| **Prefix** | Matches URL path prefix | `/api` matches `/api`, `/api/`, `/api/users` |
| **Exact** | Exact path match only | `/api` matches only `/api` |
| **ImplementationSpecific** | Depends on IngressClass | Controller-specific behavior |

---

## TLS/HTTPS Configuration

### Step 1: Create TLS Secret

```bash
# Generate self-signed certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com"

# Create Kubernetes secret
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Step 2: Configure Ingress with TLS

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    - api.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Using cert-manager for Automatic Certificates

```yaml
# ingress-with-cert-manager.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-auto   # cert-manager creates this
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

---

## Common NGINX Ingress Annotations

```yaml
metadata:
  annotations:
    # Rewrite URL
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Backend protocol (for HTTPS backends)
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
    # Request body size limit
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # Sticky sessions
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    
    # Canary deployments
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
```

---

## Complete Example: Full Application Stack

```yaml
# complete-ingress-example.yaml

# Backend API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=Hello from API"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678

---
# Frontend Deployment
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
        image: nginx:1.25
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80

---
# Ingress with path-based routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## Default Backend

Handle requests that don't match any rules:

```yaml
# default-backend.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

---

## Ingress Commands Reference

```bash
# List ingress resources
kubectl get ingress
kubectl get ing          # Short form

# Describe ingress
kubectl describe ingress my-ingress

# Get ingress with details
kubectl get ingress -o wide

# View ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Check ingress controller service
kubectl get svc -n ingress-nginx

# Delete ingress
kubectl delete ingress my-ingress

# Test locally (add to /etc/hosts)
echo "$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}') myapp.local" | sudo tee -a /etc/hosts
```

---

## Debugging Ingress Issues

### Common Problems and Solutions

| Problem | Possible Cause | Solution |
|---------|---------------|----------|
| 404 Not Found | Path doesn't match | Check pathType and path patterns |
| 502 Bad Gateway | Backend service unreachable | Verify service and endpoints exist |
| 503 Service Unavailable | No healthy pods | Check pod status and readiness probes |
| SSL Certificate Error | Wrong or missing secret | Verify TLS secret exists and is valid |

### Debug Commands

```bash
# Check if ingress controller is running
kubectl get pods -n ingress-nginx

# Check ingress resource
kubectl describe ingress my-ingress

# Check backend services
kubectl get svc
kubectl get endpoints

# Check ingress controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

# Test from inside cluster
kubectl run debug --rm -it --image=curlimages/curl -- \
  curl -H "Host: myapp.example.com" http://ingress-nginx-controller.ingress-nginx.svc
```

---

## Best Practices

### 1. Always Specify ingressClassName
```yaml
spec:
  ingressClassName: nginx    # Explicit is better
```

### 2. Use TLS in Production
```yaml
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
```

### 3. Set Resource Limits on Controller
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.resources.requests.cpu=100m \
  --set controller.resources.requests.memory=90Mi
```

### 4. Configure Health Checks
```yaml
annotations:
  nginx.ingress.kubernetes.io/healthcheck-path: /health
```

### 5. Use Meaningful Annotations
```yaml
annotations:
  kubernetes.io/ingress.class: nginx   # Legacy, use ingressClassName
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Ingress Controller** | The actual proxy that handles traffic (NGINX, Traefik) |
| **Ingress Resource** | YAML config defining routing rules |
| **Host-based Routing** | Route by domain name |
| **Path-based Routing** | Route by URL path |
| **TLS Termination** | Handle HTTPS at the ingress level |
| **Annotations** | Controller-specific configuration |

### Key Takeaways

1. **Ingress provides L7 routing** (HTTP/HTTPS) compared to Services' L4
2. **One Ingress Controller** can handle multiple domains and paths
3. **Install a controller first** — Ingress resources need one to work
4. **Use TLS** for production workloads
5. **Annotations** provide powerful customization options
6. **cert-manager** can automate certificate management

---

## Next Steps

Your applications need configuration and secrets. Learn how to manage them in **[Day 6: ConfigMaps and Secrets](../day-6/README.md)**!
