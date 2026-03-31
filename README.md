# Generic Service Helm Chart

A generic, production-ready Helm chart for deploying frontend and backend services on Kubernetes with ArgoCD, Nginx Ingress Controller, and Cert Manager.

## Features

- **Universal Service Deployment**: Single chart for both frontend and backend services
- **Auto-scaling**: Horizontal Pod Autoscaler (HPA) with CPU and memory-based scaling
- **TLS/SSL**: Automated certificate management with cert-manager
- **Ingress**: Nginx Ingress Controller with advanced configuration
- **Cloud SQL Integration**: Built-in Cloud SQL Proxy sidecar support for GKE
- **Secret Management**: Support for Kubernetes Secrets and External Secrets Operator
- **Health Checks**: Configurable startup, liveness, and readiness probes
- **Security**: Pod Security Context, Network Policies, and RBAC
- **High Availability**: Pod Disruption Budgets and multi-replica support
- **ArgoCD Ready**: Native support for GitOps deployments

## Prerequisites

- Kubernetes cluster 1.21+
- Helm 3.8+
- Nginx Ingress Controller installed
- Cert Manager installed
- ArgoCD installed (for GitOps deployments)

## Installation

### Quick Start

#### 1. Deploy with Helm CLI

```bash
# Install backend service
helm install my-backend ./generic-service \
  -f examples/values-backend-idp.yaml \
  --namespace my-namespace \
  --create-namespace

# Install frontend service
helm install my-frontend ./generic-service \
  -f examples/values-frontend-loyalty.yaml \
  --namespace my-namespace
```

#### 2. Deploy with ArgoCD

```bash
# Single application
kubectl apply -f examples/argocd-backend-idp.yaml -n argocd

# Multiple applications via ApplicationSet
kubectl apply -f examples/argocd-applicationset.yaml -n argocd
```

### Customization

#### Create Your Own Values File

```bash
# Copy example and customize
cp examples/values-backend-idp.yaml my-service-values.yaml

# Edit the file
vim my-service-values.yaml

# Install
helm install my-service ./generic-service -f my-service-values.yaml
```

## Configuration

### Key Configuration Sections

#### Image Configuration

```yaml
image:
  repository: gcr.io/my-project/my-app
  pullPolicy: IfNotPresent
  tag: "v1.2.3"
```

#### Resource Limits

```yaml
resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi
```

#### Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
```

#### Ingress & TLS

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com
```

#### Environment Variables

```yaml
# Non-sensitive environment variables
env:
  NODE_ENV: production
  DB_HOST: "127.0.0.1"
  DB_PORT: "5432"
  LOG_LEVEL: info

# Sensitive environment variables
secrets:
  DB_PASS: "your-secure-password"
  API_KEY: "your-api-key"
```

#### Cloud SQL Proxy (GKE)

```yaml
cloudSqlProxy:
  enabled: true
  instanceConnectionName: "project:region:instance"
  port: 5432
  resources:
    requests:
      memory: 64Mi
      cpu: 50m
```

#### GKE Workload Identity

```yaml
serviceAccount:
  create: true
  annotations:
    iam.gke.io/gcp-service-account: SERVICE_ACCOUNT@PROJECT_ID.iam.gserviceaccount.com
```

## Architecture Comparison: Cloud Run vs Kubernetes

| Feature | Cloud Run | Kubernetes (This Chart) |
|---------|-----------|------------------------|
| **Secrets** | Secret Manager | Kubernetes Secrets / External Secrets |
| **Environment Variables** | Container env vars | ConfigMap + Secrets |
| **Load Balancer** | Google LB + NEG | Nginx Ingress |
| **SSL/TLS** | Certificate Manager | Cert Manager |
| **Auto-scaling** | Built-in | HPA |
| **VPC Access** | VPC Connector | Native pod networking |
| **Cloud SQL** | Built-in Cloud SQL Auth | Cloud SQL Proxy sidecar |
| **Health Checks** | Startup/Liveness probes | Startup/Liveness/Readiness |
| **IAM** | Service Account | Workload Identity |

## Migrating from Cloud Run

### 1. Map Environment Variables

Cloud Run's `environment_variables` → Helm's `env` (ConfigMap)
Cloud Run's `secret_environment_variables` → Helm's `secrets`

### 2. Configure Cloud SQL Access

Cloud Run uses direct Cloud SQL integration. In Kubernetes, enable the Cloud SQL Proxy sidecar:

```yaml
cloudSqlProxy:
  enabled: true
  instanceConnectionName: "my-project:us-central1:my-instance"
```

Update `DB_HOST` to `127.0.0.1` to use the proxy.

### 3. Set Up Ingress

Cloud Run uses Google Load Balancer with NEGs. In Kubernetes:

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: your-domain.com
      paths:
        - path: /
          pathType: Prefix
```

### 4. Configure Workload Identity (GKE)

Replace Cloud Run service account with GKE Workload Identity:

```yaml
serviceAccount:
  create: true
  annotations:
    iam.gke.io/gcp-service-account: SA@PROJECT.iam.gserviceaccount.com
```

Bind the Kubernetes SA to the GCP SA:
```bash
gcloud iam service-accounts add-iam-policy-binding \
  SA@PROJECT.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]"
```

## Secret Management Best Practices

### Option 1: Sealed Secrets

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Seal your secret
echo -n 'my-secret-password' | kubectl create secret generic my-secret \
  --dry-run=client --from-file=DB_PASS=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Apply sealed secret
kubectl apply -f sealed-secret.yaml
```

### Option 2: External Secrets Operator with GCP Secret Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcpsm-secret-store
  namespace: my-namespace
spec:
  provider:
    gcpsm:
      projectID: my-project-id
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: my-cluster
          serviceAccountRef:
            name: my-service-app

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-service-secrets
  namespace: my-namespace
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcpsm-secret-store
    kind: SecretStore
  target:
    name: my-service-app
  data:
    - secretKey: DB_PASS
      remoteRef:
        key: my-db-password-secret
    - secretKey: API_KEY
      remoteRef:
        key: my-api-key-secret
```

Then reference in values:
```yaml
envFromSecret:
  enabled: true
  existingSecret: my-service-app  # Use the external secret
```

## ArgoCD Best Practices

### 1. Use ApplicationSets for Multiple Services

Deploy all services with a single ApplicationSet manifest (see `examples/argocd-applicationset.yaml`)

### 2. Enable Auto-Sync with Self-Heal

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### 3. Ignore HPA Replica Changes

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
```

### 4. Use Sync Waves for Ordered Deployment

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy after wave 0
```

## Monitoring & Observability

### Prometheus Metrics

```yaml
# Enable in values.yaml (add to chart if needed)
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

### Logging

Configure log aggregation with FluentBit sidecar:

```yaml
sidecars:
  - name: fluent-bit
    image: fluent/fluent-bit:1.9
    volumeMounts:
      - name: logs
        mountPath: /var/log
```

## Troubleshooting

### Check Deployment Status

```bash
# Check pods
kubectl get pods -n my-namespace

# Check pod logs
kubectl logs -n my-namespace POD_NAME

# Check ingress
kubectl get ingress -n my-namespace

# Check certificate
kubectl get certificate -n my-namespace

# Describe certificate for issues
kubectl describe certificate -n my-namespace
```

### Common Issues

#### 1. Certificate Not Issuing

```bash
# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager

# Check certificate status
kubectl describe certificate -n my-namespace my-tls-secret
```

#### 2. Ingress Not Working

```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Verify ingress resource
kubectl get ingress -n my-namespace -o yaml
```

#### 3. Cloud SQL Proxy Connection Failures

```bash
# Check proxy sidecar logs
kubectl logs -n my-namespace POD_NAME -c cloud-sql-proxy

# Verify Workload Identity binding
gcloud iam service-accounts get-iam-policy SA@PROJECT.iam.gserviceaccount.com
```

## Advanced Configuration

### Network Policies

```yaml
networkPolicy:
  enabled: true
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: ingress-nginx
  egress:
    - to:
      - namespaceSelector: {}
      ports:
        - port: 5432
```

### Pod Affinity

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-service
          topologyKey: kubernetes.io/hostname
```

## Contributing

Contributions are welcome. Please open a pull request or issue.

## License

MIT License

## Maintainer

Muhammad Zeeshan
