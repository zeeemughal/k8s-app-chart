# Generic Service Helm Chart - Deployment Guide

Complete guide for deploying services to Kubernetes using the multi-environment values structure.

## Table of Contents

1. [Quick Start](#quick-start)
2. [Values File Structure](#values-file-structure)
3. [Deployment Methods](#deployment-methods)
4. [ArgoCD Integration](#argocd-integration)
5. [Common Scenarios](#common-scenarios)
6. [Troubleshooting](#troubleshooting)

---

## Quick Start

### Deploy to Development (Single Service)

```bash
# Using Helm CLI
helm install my-service . \
  -f values.yaml \
  -f values/develop.yaml \
  -n my-namespace \
  --create-namespace

# With custom image tag
helm install my-service . \
  -f values.yaml \
  -f values/develop.yaml \
  --set image.tag=v1.2.3 \
  -n my-namespace
```

### Deploy to Production

```bash
helm install my-service . \
  -f values.yaml \
  -f values/production.yaml \
  --set image.tag=v2.0.1 \
  -n my-namespace-production \
  --create-namespace
```

---

## Values File Structure

### File Organization

```
generic-service/
├── values.yaml                    # Common defaults (ALL environments)
├── values/
│   ├── README.md                  # Values structure documentation
│   ├── develop.yaml               # Development overrides
│   ├── staging.yaml               # Staging overrides
│   └── production.yaml            # Production overrides
├── templates/                     # Helm templates
├── examples/                      # ArgoCD examples
│   ├── argocd-backend-idp.yaml
│   └── argocd-applicationset.yaml
├── Chart.yaml
└── DEPLOYMENT_GUIDE.md            # This file
```

### What Goes Where

| Configuration | values.yaml | values/{env}.yaml | ArgoCD Inline |
|--------------|-------------|-------------------|---------------|
| Probe settings | ✅ | ❌ | ❌ |
| Default resources | ✅ | Override if needed | Override if needed |
| Project ID | ❌ | ✅ | ❌ |
| Service name | ❌ | ❌ | ✅ |
| Image repository | ❌ | ✅ | ✅ (override tag) |
| Domain names | ❌ | ✅ | ✅ (service-specific) |
| Database names | ❌ | ✅ | ✅ (service-specific) |
| Secret names | ❌ | ✅ | ✅ (service-specific) |

### Value Precedence

```
1. values.yaml (lowest priority - defaults)
   ↓
2. values/develop.yaml (environment overrides)
   ↓
3. --set or inline values (highest priority - runtime overrides)
```

**Example:**
```yaml
# values.yaml
resources:
  limits:
    cpu: 1000m
    memory: 2Gi

# values/production.yaml
resources:
  limits:
    cpu: 2000m      # Overrides values.yaml
    memory: 4Gi     # Overrides values.yaml

# ArgoCD inline or --set
resources:
  limits:
    cpu: 4000m      # Overrides everything above
```

---

## Deployment Methods

### Method 1: Helm CLI (Direct)

**Advantages:**
- Quick testing
- Full control
- Good for development

**Example:**
```bash
# Development
helm upgrade --install my-service . \
  -f values.yaml \
  -f values/develop.yaml \
  -n my-namespace \
  --create-namespace

# Production with specific version
helm upgrade --install my-service . \
  -f values.yaml \
  -f values/production.yaml \
  --set image.tag=v2.1.0 \
  -n my-namespace-production \
  --create-namespace
```

### Method 2: ArgoCD Application (Single Service)

**Advantages:**
- GitOps workflow
- Automatic sync
- Audit trail
- Drift detection

**Example:** `argocd-my-service-develop.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-develop
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-org/your-iac-repo.git
    targetRevision: develop
    path: helm-chart/generic-service
    helm:
      valueFiles:
        - values.yaml
        - values/develop.yaml
      values: |
        serviceName: my-service
        image:
          repository: gcr.io/my-project/my-service
          tag: "latest"
  destination:
    namespace: my-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply:
```bash
kubectl apply -f argocd-my-service-develop.yaml -n argocd
```

### Method 3: ArgoCD ApplicationSet (Multiple Services/Environments)

**Advantages:**
- Deploy all services at once
- Consistent configuration
- Matrix support (environment × service)
- Single source of truth

**Example:** See `examples/argocd-applicationset.yaml`

This single manifest can deploy:
- 3 services × 3 environments = 9 applications automatically
- All using the same Helm chart
- Consistent configuration with environment-specific overrides

Apply:
```bash
kubectl apply -f examples/argocd-applicationset.yaml -n argocd
```

---

## ArgoCD Integration

### Prerequisites

1. **ArgoCD installed** in the cluster
2. **Git repository** with Helm chart accessible to ArgoCD
3. **Namespace** created (or use `CreateNamespace=true`)
4. **External Secrets Operator** installed (if using ESO)
5. **Cert-Manager** with ClusterIssuer configured
6. **Nginx Ingress Controller** running

### Verify Prerequisites

```bash
# Check ArgoCD
kubectl get pods -n argocd

# Check External Secrets Operator
kubectl get pods -n external-secrets-system
kubectl get clustersecretstore gcp-secrets-manager

# Check Cert-Manager
kubectl get pods -n cert-manager
kubectl get clusterissuer letsencrypt-prod

# Check Nginx Ingress
kubectl get pods -n ingress-nginx
```

### Single Application Deployment

**Step 1:** Create Application manifest
```yaml
# my-service-develop.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-develop
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-org/your-iac-repo.git
    targetRevision: develop
    path: helm-chart/generic-service
    helm:
      valueFiles:
        - values.yaml
        - values/develop.yaml
      values: |
        serviceName: my-service
        # Add service-specific overrides here
  destination:
    namespace: my-namespace-develop
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Step 2:** Apply to ArgoCD
```bash
kubectl apply -f my-service-develop.yaml -n argocd
```

**Step 3:** Monitor sync
```bash
# Via CLI
argocd app get my-service-develop

# Via UI
# Open https://<argocd-server>/applications/my-service-develop
```

### ApplicationSet Deployment (Recommended for Multiple Services)

**Edit:** `examples/argocd-applicationset.yaml`

Add your services to the generator:
```yaml
- service: my-new-service
  type: backend
  dbName: my_new_service_db
  dbUser: my_new_service_db_admin
  dbInstance: my-new-service-postgres
```

Apply:
```bash
kubectl apply -f examples/argocd-applicationset.yaml -n argocd
```

This creates applications for ALL combinations:
- `my-new-service-develop`
- `my-new-service-staging`
- `my-new-service-production`

---

## Common Scenarios

### Scenario 1: Deploy New Service to Development

```bash
# Option A: Helm CLI
helm install my-new-service . \
  -f values.yaml \
  -f values/develop.yaml \
  --set serviceName=my-new-service \
  --set image.repository=gcr.io/my-project/my-new-service \
  -n my-namespace-develop

# Option B: ArgoCD (recommended)
# 1. Create argocd-my-new-service-develop.yaml
# 2. kubectl apply -f argocd-my-new-service-develop.yaml -n argocd
```

### Scenario 2: Update Image Tag in Production

```bash
# Option A: Helm CLI
helm upgrade my-service . \
  -f values.yaml \
  -f values/production.yaml \
  --set image.tag=v2.1.5 \
  -n my-namespace-production

# Option B: ArgoCD
# Update the Application manifest in Git:
spec:
  source:
    helm:
      values: |
        image:
          tag: "v2.1.5"
# ArgoCD will auto-sync if automated sync is enabled
```

### Scenario 3: Scale Production Service

```bash
# Update values/production.yaml in Git
autoscaling:
  minReplicas: 3    # Was 2
  maxReplicas: 20   # Was 10

# Commit and push
# ArgoCD will auto-sync
```

### Scenario 4: Add New Environment Variable

**For ALL environments:**
Edit `values.yaml`:
```yaml
env:
  NEW_FEATURE_FLAG: "true"
```

**For specific environment:**
Edit `values/production.yaml`:
```yaml
env:
  NEW_FEATURE_FLAG: "true"
```

### Scenario 5: Promote from Staging to Production

```bash
# 1. Get current staging image tag
argocd app get my-service-staging -o json | jq '.spec.source.helm.values'

# 2. Update production Application with same tag
# Edit argocd-my-service-production.yaml
spec:
  source:
    helm:
      values: |
        image:
          tag: "v1.5.3"  # Promoted from staging

# 3. Commit and push, or apply directly
kubectl apply -f argocd-my-service-production.yaml -n argocd
```

---

## Troubleshooting

### Issue: Values not being applied

**Symptom:** Changes to values files not reflected in deployment

**Check:**
```bash
# View rendered template
helm template test . \
  -f values.yaml \
  -f values/develop.yaml \
  --debug

# Check ArgoCD sync status
argocd app get my-service-develop
```

**Solution:**
- Ensure ArgoCD has access to Git repository
- Check for YAML syntax errors
- Verify value precedence (later files override earlier)

### Issue: 502 Bad Gateway

**Symptom:** Ingress returns 502

**Root Cause:** Application not listening on the expected port

**Check:**
```bash
# Test from inside cluster
kubectl run test-curl --image=curlimages/curl:latest --rm -it --restart=Never -n my-namespace \
  -- curl -v http://my-service:80/

# Check if PORT environment variable is set
kubectl exec -n my-namespace POD_NAME -- env | grep PORT
```

**Solution:**
Ensure `PORT: "80"` is set in environment variables (required for many Node.js/NestJS apps)

### Issue: Probes Failing

**Symptom:** Pods showing 1/2 Ready or CrashLoopBackOff

**Check:**
```bash
kubectl describe pod POD_NAME -n my-namespace | grep -A 10 "Events:"
```

**Common Causes:**
1. Application not listening on `0.0.0.0` (all interfaces)
2. Missing `PORT` environment variable
3. Wrong probe endpoints
4. Application takes too long to start (increase `failureThreshold`)

**Solution:**
```yaml
# Ensure PORT is set
env:
  PORT: "80"

# Adjust probe timeouts if needed
startupProbe:
  failureThreshold: 30  # Allow more time for startup
```

### Issue: External Secrets Not Syncing

**Symptom:** `SecretSyncedError` in ExternalSecret status

**Check:**
```bash
kubectl get externalsecret -n my-namespace
kubectl describe externalsecret my-service -n my-namespace
```

**Common Causes:**
1. Secret doesn't exist in GCP Secret Manager
2. Wrong secret name (typo)
3. Workload Identity not configured
4. GCP SA missing `secretmanager.secretAccessor` role

### Issue: Workload Identity Not Working

**Symptom:** `Permission denied` errors accessing GCP services

**Check:**
```bash
# Get K8s SA name
kubectl get sa -n my-namespace

# Check if binding exists
gcloud iam service-accounts get-iam-policy SA@PROJECT.iam.gserviceaccount.com
```

**Solution:**
```bash
# Bind Workload Identity
gcloud iam service-accounts add-iam-policy-binding \
  SA@PROJECT.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]"
```

---

## Best Practices

### DO

1. **Use specific image tags in production** (not `:latest`)
2. **Keep common settings in values.yaml**
3. **Use environment files for environment-specific overrides only**
4. **Test in development/staging before production**
5. **Use ArgoCD for GitOps workflow**
6. **Enable automated sync with `prune` and `selfHeal`**
7. **Set resource requests and limits**
8. **Use External Secrets Operator for secrets**
9. **Enable probes (startup, liveness, readiness)**
10. **Use HPA for auto-scaling**

### DON'T

1. **Don't use `:latest` tag in production**
2. **Don't put secrets in values files (use External Secrets)**
3. **Don't duplicate values across environment files**
4. **Don't commit sensitive data to Git**
5. **Don't disable probes in production**
6. **Don't skip testing in staging**
7. **Don't directly edit resources in cluster (use GitOps)**
8. **Don't forget to set `PORT` environment variable for Node.js apps**
9. **Don't ignore resource limits**
10. **Don't skip Workload Identity binding**

---

## Useful Commands

```bash
# Helm
helm list -n my-namespace
helm get values my-service -n my-namespace
helm upgrade --install my-service . -f values.yaml -f values/develop.yaml -n my-namespace
helm uninstall my-service -n my-namespace

# ArgoCD
argocd app list
argocd app get my-service-develop
argocd app sync my-service-develop
argocd app diff my-service-develop

# Kubernetes
kubectl get pods -n my-namespace
kubectl get ingress -n my-namespace
kubectl get externalsecret -n my-namespace
kubectl get certificate -n my-namespace
kubectl logs -n my-namespace POD_NAME
kubectl describe pod -n my-namespace POD_NAME
```
