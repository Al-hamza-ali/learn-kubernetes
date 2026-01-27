# Day 12: Helm Package Manager

## Overview

**Helm** is the package manager for Kubernetes. It helps you define, install, and upgrade complex Kubernetes applications using reusable templates called **Charts**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM HELM SOLVES                       │
│                                                                  │
│   Without Helm:                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  kubectl apply -f deployment.yaml                        │   │
│   │  kubectl apply -f service.yaml                           │   │
│   │  kubectl apply -f configmap.yaml                         │   │
│   │  kubectl apply -f secret.yaml                            │   │
│   │  kubectl apply -f ingress.yaml                           │   │
│   │  kubectl apply -f pvc.yaml                               │   │
│   │  ... (repeat for each environment with different values) │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   With Helm:                                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  helm install myapp ./mychart -f production-values.yaml  │   │
│   │                                                          │   │
│   │  ✅ All resources deployed                               │   │
│   │  ✅ Environment-specific values                          │   │
│   │  ✅ Easy upgrades and rollbacks                          │   │
│   │  ✅ Version history                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Chart** | A package of pre-configured Kubernetes resources |
| **Release** | A deployed instance of a chart |
| **Repository** | Where charts are stored and shared |
| **Values** | Configuration for customizing charts |

---

## Installing Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows (Chocolatey)
choco install kubernetes-helm

# Verify installation
helm version
```

---

## Helm Commands Cheat Sheet

```bash
# Repository management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo nginx

# Install a chart
helm install my-release bitnami/nginx
helm install my-release ./my-chart
helm install my-release bitnami/nginx -f values.yaml
helm install my-release bitnami/nginx --set replicaCount=3

# List releases
helm list
helm list -A              # All namespaces

# Upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=5
helm upgrade my-release ./my-chart -f new-values.yaml

# Rollback
helm rollback my-release 1    # Rollback to revision 1
helm history my-release       # View revision history

# Uninstall
helm uninstall my-release

# Debugging
helm template my-release ./my-chart    # Render templates locally
helm get values my-release             # Get deployed values
helm get manifest my-release           # Get deployed manifests
```

---

## Chart Structure

```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
├── templates/          # Kubernetes resource templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # Template helpers
│   ├── NOTES.txt       # Post-install notes
│   └── tests/          # Test templates
└── .helmignore         # Files to ignore
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: A Helm chart for My Application
type: application
version: 1.0.0          # Chart version
appVersion: "2.0.0"     # Application version
keywords:
  - webapp
  - nodejs
maintainers:
  - name: John Doe
    email: john@example.com
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myapp
  tag: "2.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

postgresql:
  enabled: true
  auth:
    username: myapp
    database: myapp
```

---

## Writing Templates

### Basic Template Syntax

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        release: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 80
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
```

### Built-in Objects

| Object | Description |
|--------|-------------|
| `.Release` | Release info (name, namespace, revision) |
| `.Chart` | Chart.yaml contents |
| `.Values` | Values from values.yaml and overrides |
| `.Files` | Access non-template files |
| `.Capabilities` | Cluster capabilities info |
| `.Template` | Current template info |

### Template Functions

```yaml
# String functions
{{ .Values.name | upper }}
{{ .Values.name | lower }}
{{ .Values.name | quote }}
{{ .Values.name | default "myapp" }}
{{ .Values.name | required "name is required" }}

# Conditionals
{{- if .Values.ingress.enabled }}
# ingress manifest
{{- end }}

{{- if and .Values.a .Values.b }}
# both a and b are true
{{- end }}

# Loops
{{- range .Values.hosts }}
- host: {{ . }}
{{- end }}

{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}

# Include/toYaml
{{- include "myapp.labels" . | nindent 4 }}
{{- toYaml .Values.resources | nindent 8 }}
```

### Helper Templates (_helpers.tpl)

```yaml
# templates/_helpers.tpl
{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Full name
*/}}
{{- define "myapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
```

---

## Complete Chart Example

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.containerPort | default 80 }}
        {{- if .Values.env }}
        env:
          {{- range $key, $value := .Values.env }}
          - name: {{ $key }}
            value: {{ $value | quote }}
          {{- end }}
        {{- end }}
        {{- if .Values.envFrom }}
        envFrom:
          - configMapRef:
              name: {{ include "myapp.fullname" . }}-config
        {{- end }}
        {{- if .Values.livenessProbe.enabled }}
        livenessProbe:
          httpGet:
            path: {{ .Values.livenessProbe.path }}
            port: http
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
          periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
        {{- end }}
        {{- if .Values.readinessProbe.enabled }}
        readinessProbe:
          httpGet:
            path: {{ .Values.readinessProbe.path }}
            port: http
          initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
          periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: http
    protocol: TCP
    name: http
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

### templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

---

## Using Charts from Repositories

### Popular Chart Repositories

```bash
# Bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# Prometheus Community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Grafana
helm repo add grafana https://grafana.github.io/helm-charts

# Ingress NGINX
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Cert Manager
helm repo add jetstack https://charts.jetstack.io
```

### Installing Popular Charts

```bash
# PostgreSQL
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=secretpassword \
  --set primary.persistence.size=10Gi

# Redis
helm install my-redis bitnami/redis \
  --set auth.password=secretpassword

# MongoDB
helm install my-mongo bitnami/mongodb \
  --set auth.rootPassword=secretpassword

# NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Prometheus Stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

---

## Environment-Specific Values

```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: "latest"
resources:
  limits:
    cpu: 200m
    memory: 256Mi

# values-prod.yaml
replicaCount: 5
image:
  tag: "2.0.0"
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
ingress:
  enabled: true
  hosts:
    - host: app.example.com
```

```bash
# Deploy to different environments
helm install myapp ./mychart -f values-dev.yaml
helm install myapp ./mychart -f values-prod.yaml
```

---

## Helm Hooks

Execute actions at specific points in the release lifecycle:

```yaml
# templates/job-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate", "up"]
      restartPolicy: Never
```

### Hook Types

| Hook | When |
|------|------|
| `pre-install` | Before resources are installed |
| `post-install` | After resources are installed |
| `pre-upgrade` | Before upgrade |
| `post-upgrade` | After upgrade |
| `pre-delete` | Before deletion |
| `post-delete` | After deletion |
| `pre-rollback` | Before rollback |
| `post-rollback` | After rollback |

---

## Best Practices

### 1. Use Semantic Versioning
```yaml
version: 1.2.3      # Chart version
appVersion: "4.5.6" # App version
```

### 2. Make Values Configurable
```yaml
# Good: Configurable
image:
  repository: myapp
  tag: "1.0.0"

# Bad: Hardcoded
image: myapp:1.0.0
```

### 3. Use Required for Mandatory Values
```yaml
{{ required "database.password is required" .Values.database.password }}
```

### 4. Include NOTES.txt
```
# templates/NOTES.txt
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To access the application:
{{- if .Values.ingress.enabled }}
  http://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
  kubectl port-forward svc/{{ include "myapp.fullname" . }} 8080:{{ .Values.service.port }}
{{- end }}
```

### 5. Validate Your Charts
```bash
helm lint ./mychart
helm template ./mychart
helm install --dry-run --debug myapp ./mychart
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Chart** | Package of Kubernetes resources |
| **Release** | Deployed instance of a chart |
| **Values** | Configuration customization |
| **Templates** | Go templates for K8s manifests |
| **Hooks** | Lifecycle event handlers |

### Key Takeaways

1. **Helm simplifies** complex Kubernetes deployments
2. **Charts are reusable** across environments
3. **Values files** customize deployments
4. **Templates use Go** templating syntax
5. **Repositories** share charts publicly or privately
6. **Hooks** enable pre/post deployment actions

---

## Next Steps

Set up observability with **[Day 13: Monitoring and Logging](../day-13/README.md)**!
