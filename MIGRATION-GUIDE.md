# Migration Guide: NGINX Ingress → Gateway API

## Overview

This guide covers migrating from the deprecated NGINX Ingress to Gateway API with Traefik.

**Timeline**: NGINX Ingress retires March 2026. Gateway API is the Kubernetes-standard replacement.

## Architecture Change

```
BEFORE (NGINX Ingress)                 AFTER (Gateway API)
┌─────────────────────┐               ┌─────────────────────┐
│  Ingress Resource   │               │    HTTPRoute        │
│  (per namespace)    │               │  (per namespace)    │
└─────────┬───────────┘               └─────────┬───────────┘
          │                                     │
          ▼                                     ▼
┌─────────────────────┐               ┌─────────────────────┐
│  NGINX Ingress      │               │   Gateway           │
│  Controller         │               │  (shared, cluster)  │
└─────────────────────┘               └─────────┬───────────┘
                                                │
                                                ▼
                                      ┌─────────────────────┐
                                      │   GatewayClass      │
                                      │  (traefik)          │
                                      └─────────────────────┘
```

## Step-by-Step Migration

### Phase 1: Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

### Phase 2: Deploy Shared Gateway

The gateway-api folder contains the shared Traefik Gateway that all apps will use.

```bash
# Via ArgoCD (already configured in k-gitops/apps/gateway-api.yaml)
git add k-apps/gateway-api/
git commit -m "Add Gateway API configuration"
git push

# ArgoCD will auto-sync
```

### Phase 3: Migrate Apps (One at a Time)

For organic-farm, the new `gateway.yaml` HTTPRoute is already created.

**Testing approach:**
1. Keep old `ingress.yaml` temporarily
2. Add new `gateway.yaml`
3. Test with Gateway API
4. Once verified, delete `ingress.yaml`

```bash
# Test that HTTPRoute is working
kubectl get httproutes -n multi-farm
kubectl describe httproute farm-routes -n multi-farm
```

### Phase 4: Update DNS/Certificates

If using cert-manager with Gateway API:

```yaml
# The Gateway already has annotation:
# cert-manager.io/cluster-issuer: letsencrypt-http
```

For wildcard cert, create a Certificate resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-kaayaka-tls
  namespace: traefik-system
spec:
  secretName: wildcard-kaayaka-tls
  issuerRef:
    name: letsencrypt-http
    kind: ClusterIssuer
  dnsNames:
    - "*.kaayaka.in"
    - "kaayaka.in"
```

### Phase 5: Remove NGINX Ingress

Once all apps are migrated:

```bash
# Remove NGINX Ingress controller
kubectl delete namespace ingress-nginx

# Remove old Ingress resources
rm k-apps/organic-farm/ingress.yaml
git commit -m "Remove deprecated NGINX Ingress"
git push
```

## Files Changed

### k-apps/
```
├── gateway-api/
│   └── traefik-gateway.yaml    # NEW: Shared Gateway + GatewayClass
├── organic-farm/
│   ├── gateway.yaml            # NEW: HTTPRoute (of.kaayaka.in)
│   ├── ingress.yaml            # DELETE after migration
│   ├── bend-deploy.yaml        # UPDATED: health checks, resources
│   └── fend-deploy.yaml        # UPDATED: health checks, resources
├── m-site/
│   ├── gateway.yaml            # NEW: HTTPRoute (kaayaka.in, www.kaayaka.in)
│   └── ingress.yaml            # DELETE after migration
├── re-app/
│   ├── gateway.yaml            # NEW: HTTPRoute (re.kaayaka.in)
│   └── ingress.yaml            # DELETE after migration
├── multi-ride-maps/
│   ├── gateway.yaml            # NEW: HTTPRoute (maps.kaayaka.in)
│   └── ingress.yaml            # DELETE after migration
├── ai-api/
│   ├── gateway.yaml            # NEW: HTTPRoute (ai.kaayaka.in)
│   └── ingress.yaml            # DELETE after migration
├── s3-minio/
│   ├── gateway.yaml            # NEW: HTTPRoute (mnio.kaayaka.in)
│   └── ingress.yaml            # DELETE after migration
└── vault/
    ├── vault-deployment.yaml   # NEW: Vault server
    ├── external-secrets.yaml   # NEW: ESO integration
    ├── vault-route.yaml        # NEW: Vault UI (vault.kaayaka.in)
    └── README.md               # NEW: Setup instructions
```

### k-gitops/apps/
```
├── gateway-api.yaml            # NEW: ArgoCD app
└── vault.yaml                  # NEW: ArgoCD app
```

## All Routes Summary

| Domain | Namespace | Backend Services |
|--------|-----------|------------------|
| kaayaka.in | kaayaka-prod | mweb-service:80 |
| www.kaayaka.in | kaayaka-prod | mweb-service:80 |
| of.kaayaka.in | multi-farm | frontend:3000, backend:8000 |
| re.kaayaka.in | default | re-service:80 |
| maps.kaayaka.in | multi-ride-ns | fe:80, be:5000 |
| ai.kaayaka.in | ai | ai-backend:80 |
| mnio.kaayaka.in | infra | minio-service:9000 |
| vault.kaayaka.in | vault | vault:8200 |

## Rollback Plan

If issues occur:

1. Keep `ingress.yaml` in place during testing
2. HTTPRoute and Ingress can coexist
3. To rollback: delete HTTPRoute, Ingress takes over

```bash
kubectl delete httproute farm-routes -n multi-farm
```

## Verification Commands

```bash
# Check Gateway status
kubectl get gateway -A
kubectl describe gateway traefik-gateway -n traefik-system

# Check HTTPRoute status
kubectl get httproute -A
kubectl describe httproute farm-routes -n multi-farm

# Check if Traefik sees the routes
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik

# Test endpoint
curl -I https://of.kaayaka.in/api/v1/products
```
