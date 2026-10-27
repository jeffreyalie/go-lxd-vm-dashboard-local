# Docker Compose Deployment for go-lxd-vm-dashboard

This directory includes files for easy deployment using Docker Compose.

## Quick Start (Development)

1. **Setup Vault credentials secret** (run on MicroK8s host):

```bash
microk8s kubectl create secret generic openbao-creds \
  --from-literal=VAULT_ADDR=https://<your-vault-host>:8200 \
  --from-literal=VAULT_ROLE_ID=<your-role-id> \
  --from-literal=VAULT_SECRET_ID=<your-secret-id> \
  --namespace dev
```

2. **Copy example configuration**:

```bash
cp docker-compose.example.yml docker-compose.yml
```

3. **Edit `docker-compose.yml`** with your actual values:
   - Set `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID`
   - Update `LXD_API_ADDRESS` to point to your LXD server
   - Set `VAULT_SKIP_VERIFY=false` for production (with proper CA certs)

4. **Deploy**:

```bash
microk8s kubectl apply -f docker-compose.yml
```

5. **Verify deployment**:

```bash
microk8s kubectl get pods -n dev
curl http://localhost:8080
```

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Main deployment file (edit with your values) |
| `docker-compose.example.yml` | Template with comments explaining each setting |
| `dockerfile` | Multi-stage Alpine build used by docker-compose |

## Usage Scenarios

### Local Development
- Run container directly on host: `microk8s kubectl apply -f docker-compose.yml`
- Quick rebuild: Edit files, then `microk8s kubectl rollout restart deployment/go-lxd-vm-dashboard -n dev`

### Simple Deployment
- Use docker-compose for isolated testing before Helm deployment
- Easy rollback: Just stop and remove containers, redeploy with previous image tag

### Hybrid K8s + Docker Compose
- Keep Vault secrets in K8s Secret
- Mount them into docker-compose volumes (requires host access)
- Good for development with production-grade secret management

## Comparison: Helm vs Docker Compose

| Feature | Helm Chart | Docker Compose |
|---------|-----------|----------------|
| **Best for** | Production, cluster-wide deployment | Development, testing, simple setups |
| **Complexity** | Higher (templating, multiple manifests) | Lower (single YAML file) |
| **Secrets** | K8s Secrets integrated | Environment variables or mounted secrets |
| **Rolling Updates** | Built-in | Manual image tag update needed |
| **Persistence** | Volumes defined in manifests | Can add volume mounts easily |
| **Ports** | Configurable via values | `ports` section in YAML |

## Security Notes

1. **Never commit real secrets**: Always use `.example.yml` as template
2. **VAULT_SKIP_VERIFY**: Set to `false` in production with proper CA certificates
3. **Secret rotation**: Use K8s Secret + environment variable injection for better secret management
4. **Network isolation**: Consider adding network constraints in docker-compose

## Troubleshooting

### Container won't start
```bash
microk8s kubectl logs -f pod-name -n dev
microk8s kubectl describe pod -n dev <pod-name>
```

### Vault auth fails
- Check `VAULT_ADDR` is reachable from container
- Verify AppRole credentials are correct
- Ensure Vault policy grants read access to `homelab/data/lxd`

### LXD API errors
- Confirm `LXD_API_ADDRESS` URL is accessible
- Check TLS certificates in Vault are valid for the LXD server
