# HashiCorp Vault Setup for K3s

## Prerequisites

1. **Install Gateway API CRDs** (if not already installed):
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
   ```

2. **Install External Secrets Operator**:
   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm install external-secrets external-secrets/external-secrets \
     -n external-secrets --create-namespace
   ```

## Deployment Order

1. Deploy Gateway API (traefik-gateway)
2. Deploy Vault
3. Initialize and unseal Vault
4. Configure Vault for Kubernetes auth
5. Store secrets in Vault
6. Deploy ExternalSecret resources

## Initialize Vault

After Vault pod is running:

```bash
# Port forward to Vault
kubectl port-forward -n vault svc/vault 8200:8200

# In another terminal, initialize Vault
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator init -key-shares=1 -key-threshold=1

# SAVE THE UNSEAL KEY AND ROOT TOKEN!
# Example output:
# Unseal Key 1: xxxxx
# Initial Root Token: hvs.xxxxx

# Unseal Vault
vault operator unseal <unseal-key>

# Login
vault login <root-token>
```

## Configure Kubernetes Auth

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Configure it to talk to K3s
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local:443"

# Create policy for external-secrets
vault policy write external-secrets - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

# Create role for external-secrets
vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h
```

## Store Organic Farm Secrets

```bash
# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Store organic-farm secrets
vault kv put secret/organic-farm/secrets \
  jwt-secret-key="$(openssl rand -hex 32)" \
  database-url="postgresql+asyncpg://user:pass@postgres.infra.svc.cluster.local:5432/farmdb" \
  minio-root-user="minio-admin" \
  minio-password="your-minio-password"
```

## Verify External Secrets

After deploying the ExternalSecret:

```bash
# Check ExternalSecret status
kubectl get externalsecrets -n multi-farm

# Check if K8s secret was created
kubectl get secrets -n multi-farm farm-secrets -o yaml
```

## Production Recommendations

1. **High Availability**: Use Vault with Raft storage (3+ nodes)
2. **Auto-unseal**: Configure with cloud KMS or Transit secrets engine
3. **Backup**: Regular snapshots of Vault data
4. **Audit**: Enable audit logging
   ```bash
   vault audit enable file file_path=/vault/logs/audit.log
   ```
