# Day 6: ConfigMaps and Secrets

## Overview

Applications need configuration — database URLs, feature flags, API keys, passwords. Kubernetes provides two resources to manage this:

- **ConfigMaps**: Store non-sensitive configuration data
- **Secrets**: Store sensitive data (passwords, tokens, keys)

```
┌─────────────────────────────────────────────────────────────────┐
│                CONFIGURATION MANAGEMENT                          │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      POD                                 │   │
│   │                                                          │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │              Container                           │   │   │
│   │   │                                                  │   │   │
│   │   │   Environment Variables:                        │   │   │
│   │   │   ├── DATABASE_HOST (from ConfigMap)            │   │   │
│   │   │   ├── LOG_LEVEL (from ConfigMap)                │   │   │
│   │   │   ├── DATABASE_PASSWORD (from Secret)           │   │   │
│   │   │   └── API_KEY (from Secret)                     │   │   │
│   │   │                                                  │   │   │
│   │   │   Mounted Files:                                │   │   │
│   │   │   ├── /etc/config/app.properties (ConfigMap)    │   │   │
│   │   │   └── /etc/secrets/credentials (Secret)         │   │   │
│   │   │                                                  │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                          │                │                      │
│                          ▼                ▼                      │
│                    ┌──────────┐     ┌──────────┐                │
│                    │ ConfigMap│     │  Secret  │                │
│                    └──────────┘     └──────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

---

## ConfigMaps

### What is a ConfigMap?

A **ConfigMap** stores non-confidential configuration data as key-value pairs. Use it for:
- Application settings
- Feature flags
- Configuration files
- Environment-specific values

### Creating ConfigMaps

#### Method 1: From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql.default.svc \
  --from-literal=DATABASE_PORT=3306 \
  --from-literal=LOG_LEVEL=info
```

#### Method 2: From File

```bash
# Create a config file
cat > app.properties << EOF
database.host=mysql.default.svc
database.port=3306
log.level=info
feature.dark-mode=true
EOF

# Create ConfigMap from file
kubectl create configmap app-config --from-file=app.properties

# Create with custom key name
kubectl create configmap app-config --from-file=config.properties=app.properties
```

#### Method 3: From Directory

```bash
# Create multiple config files
mkdir config
echo "value1" > config/key1
echo "value2" > config/key2

# Create ConfigMap from all files in directory
kubectl create configmap app-config --from-file=config/
```

#### Method 4: From YAML (Recommended)

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app: myapp
data:
  # Simple key-value pairs
  DATABASE_HOST: "mysql.default.svc"
  DATABASE_PORT: "3306"
  LOG_LEVEL: "info"
  FEATURE_DARK_MODE: "true"
  
  # Multi-line configuration file
  app.properties: |
    database.host=mysql.default.svc
    database.port=3306
    log.level=info
    
  # JSON configuration
  config.json: |
    {
      "database": {
        "host": "mysql.default.svc",
        "port": 3306
      },
      "logging": {
        "level": "info"
      }
    }
```

---

## Using ConfigMaps in Pods

### Method 1: Environment Variables (Single Key)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
```

### Method 2: Environment Variables (All Keys)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config
      prefix: APP_    # Optional: adds prefix to all keys
```

### Method 3: Volume Mount (Files)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Result: Each key becomes a file in `/etc/config/`:
```
/etc/config/
├── DATABASE_HOST
├── DATABASE_PORT
├── LOG_LEVEL
├── app.properties
└── config.json
```

### Method 4: Volume Mount (Specific Keys)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: application.properties
      - key: config.json
        path: app-config.json
```

---

## Secrets

### What is a Secret?

A **Secret** stores sensitive data like passwords, tokens, and keys. While similar to ConfigMaps, Secrets are:
- Base64 encoded (not encrypted by default!)
- Stored in tmpfs (memory) when mounted
- Can be encrypted at rest with additional configuration

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/service-account-token` | Service account token |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH authentication |

### Creating Secrets

#### Method 1: From Literal Values

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123
```

#### Method 2: From File

```bash
# Create files with sensitive data
echo -n 'admin' > username.txt
echo -n 'supersecret123' > password.txt

kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Clean up
rm username.txt password.txt
```

#### Method 3: From YAML

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Values must be base64 encoded
  username: YWRtaW4=           # echo -n 'admin' | base64
  password: c3VwZXJzZWNyZXQxMjM=  # echo -n 'supersecret123' | base64
```

#### Method 4: Using stringData (Plain Text)

```yaml
# secret-stringdata.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  # Plain text - Kubernetes will encode automatically
  username: admin
  password: supersecret123
  config.yaml: |
    database:
      username: admin
      password: supersecret123
```

### Special Secret Types

#### TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

#### Docker Registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com
```

---

## Using Secrets in Pods

### Method 1: Environment Variables (Single Key)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### Method 2: Environment Variables (All Keys)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    envFrom:
    - secretRef:
        name: db-credentials
```

### Method 3: Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400    # Read-only for owner
```

### Using Docker Registry Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: registry.example.com/myapp:1.0
  imagePullSecrets:
  - name: my-registry-secret
```

---

## Complete Example

```yaml
# complete-config-example.yaml

# ConfigMap for non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        proxy_pass http://localhost:8080;
      }
    }

---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://admin:password123@db.example.com:5432/mydb"
  API_KEY: "sk-1234567890abcdef"
  JWT_SECRET: "my-super-secret-jwt-key"

---
# Deployment using both
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
        - containerPort: 8080
        
        # Environment variables from ConfigMap
        envFrom:
        - configMapRef:
            name: webapp-config
        
        # Environment variables from Secret
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: DATABASE_URL
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: API_KEY
        
        # Mount configuration files
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
      
      volumes:
      - name: nginx-config
        configMap:
          name: webapp-config
          items:
          - key: nginx.conf
            path: default.conf
      - name: secrets
        secret:
          secretName: webapp-secrets
```

---

## ConfigMap and Secret Updates

### Automatic Updates (Volume Mounts)

When mounted as volumes, ConfigMaps and Secrets are **automatically updated** (within ~1 minute):

```yaml
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: my-config
```

⚠️ **Note**: Environment variables are NOT updated automatically — requires pod restart.

### Immutable ConfigMaps and Secrets

For better performance and preventing accidental changes:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true    # Cannot be modified after creation
data:
  KEY: "value"
```

---

## Commands Reference

```bash
# ConfigMap commands
kubectl create configmap my-config --from-literal=key=value
kubectl get configmaps
kubectl get cm                        # Short form
kubectl describe configmap my-config
kubectl get configmap my-config -o yaml
kubectl delete configmap my-config

# Secret commands
kubectl create secret generic my-secret --from-literal=password=secret
kubectl get secrets
kubectl describe secret my-secret
kubectl get secret my-secret -o yaml

# Decode secret value
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Edit ConfigMap/Secret
kubectl edit configmap my-config
kubectl edit secret my-secret
```

---

## Best Practices

### 1. Never Store Secrets in Git
```bash
# Add to .gitignore
*-secret.yaml
*.secret.yaml
```

### 2. Use External Secret Management
Consider tools like:
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- External Secrets Operator

### 3. Enable Encryption at Rest
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

### 4. Use RBAC to Limit Access
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["my-secret"]
  verbs: ["get"]
```

### 5. Rotate Secrets Regularly

---

## Summary

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive config | Sensitive data |
| **Encoding** | Plain text | Base64 |
| **Size limit** | 1 MB | 1 MB |
| **Stored in** | etcd | etcd (in tmpfs when mounted) |
| **Updates** | Auto-update via volumes | Auto-update via volumes |

### Key Takeaways

1. **Use ConfigMaps** for configuration, **Secrets** for sensitive data
2. **Secrets are not encrypted by default** — just base64 encoded
3. **Volume mounts auto-update**, environment variables don't
4. **Use external secret management** for production
5. **Never commit secrets to Git**

---

## Next Steps

Applications need to persist data. Learn about storage options in **[Day 7: Volumes and Persistent Storage](../day-7/README.md)**!
