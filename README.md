# laseris-config

Kubernetes configuration for the Laseris website deployment.

## Prerequisites

- Kubernetes cluster (k3s recommended)
- Traefik ingress controller installed
- cert-manager installed for SSL certificates
- ArgoCD installed (optional, for GitOps)

## Quick Start

### 1. Configure Secrets

First, create the required secrets. Copy and fill in the example files:

```bash
# Copy example secrets
cp examples/website/secrets-example.yaml prod/website/secrets.yaml
cp examples/website/github-registery-secret-example.yaml prod/website/github-registery-secret.yaml

# Edit the secrets with your actual values
# Then apply them (before deploying the application)
kubectl apply -f prod/website/secrets.yaml
kubectl apply -f prod/website/github-registery-secret.yaml
```

### 2. Deploy Configuration

#### Option A: Manual Deployment

```bash
# Apply all resources
kubectl apply -f prod/website/

# Note: The Endpoints resource for external proxy must be applied manually
# as ArgoCD excludes it from sync (this is expected)
kubectl apply -f prod/website/ingress.yaml
```

#### Option B: ArgoCD Deployment (GitOps)

If using ArgoCD:

```bash
# Apply the ArgoCD Application
kubectl apply -f website-application.yaml

# The application will sync automatically from the repository
# Note: You may see a warning about Endpoints being excluded - this is expected
```

### 3. Verify Deployment

```bash
# Check all pods are running
kubectl get pods -n laseris-website

# Check ingress is configured
kubectl get ingress -n laseris-website

# Check services
kubectl get svc -n laseris-website
```

## Components

### Main Components

- **Namespace**: `laseris-website`
- **Application**: Next.js frontend (app.yaml)
- **API**: Payload CMS backend (api.yaml)
- **Database**: PostgreSQL (postgres-deployment.yaml)
- **Ingress**: Traefik with SSL certificates (ingress.yaml)

### External Proxy

The configuration includes a proxy for `/shop` that routes to `https://laseris.ch/shop/`:

- Uses **Endpoints + Service** approach (Reddit solution for k3s/Traefik)
- **IngressRoute** for HTTPS proxying to external service
- **ServersTransport** with SNI for TLS
- **Middleware** for Host header rewriting

**Note**: The Endpoints resource is excluded from ArgoCD sync (expected behavior). It must be applied manually if needed.

## SSL Certificates

SSL certificates are automatically managed by cert-manager using Let's Encrypt:

- **ClusterIssuer**: `letsencrypt-prod` (configured in ingress.yaml)
- **Domain**: `laseris.vpr-group.ch`
- Certificates are automatically renewed

## Troubleshooting

### Check Traefik Logs

```bash
kubectl logs -n kube-system deployment/traefik --tail=100
```

### Check Application Logs

```bash
# App logs
kubectl logs -n laseris-website deployment/laseris-website-app

# API logs
kubectl logs -n laseris-website deployment/laseris-website-api
```

### Verify External Proxy

```bash
# Test the /shop proxy
curl -I https://laseris.vpr-group.ch/shop/

# Should return HTTP 200 or 301 (redirect)
```

### ArgoCD Warnings

If you see `ExcludedResourceWarning` for Endpoints:

- This is **expected** - ArgoCD excludes Endpoints by default
- The Endpoints resource is manually managed for the external proxy
- This warning does not affect functionality

## Configuration Files

- `prod/website/` - Main deployment configuration

  - `namespace.yaml` - Namespace definition
  - `app.yaml` - Frontend application deployment
  - `api.yaml` - Backend API deployment
  - `ingress.yaml` - Traefik ingress, SSL, and external proxy
  - `postgres-deployment.yaml` - Database deployment
  - `postgres-volume.yaml` - Database storage
  - `secrets.yaml` - Application secrets (not in git)
  - `github-registery-secret.yaml` - Image pull secrets

- `website-application.yaml` - ArgoCD Application definition
- `examples/` - Example configuration files

## Notes

- Secrets are **not** committed to git - use the example files
- The external proxy IP address (83.166.138.23) may need updating if laseris.ch changes
- All resources are deployed to the `laseris-website` namespace
