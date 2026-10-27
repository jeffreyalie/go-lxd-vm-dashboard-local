# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Architecture Overview

This is a **Go web application** that displays live status of LXD instances (VMs and containers) deployed on MicroK8s via Helm. The app authenticates with OpenBao/Vault via AppRole to securely retrieve LXD TLS credentials at runtime, then queries the LXD API and renders results in a clean HTML dashboard.

### Core Components

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│   Browser    │───▶│  Ingress      │───▶│   Service    │───▶│    Pod       │
│             │◀───│ (NGINX)       │◀───│  (ClusterIP) │◀───│  (MicroK8s)  │
└─────────────┘     └──────────────┘     └──────┬───────┘     └─────────────┘
                                                │
              ┌─────────────────────────────────┼─────────────────────────────┐
              │                                 ▼                             │
   OpenBao ──┴──── Vault Auth ─────────→ Fetch TLS cert/key                │
      secrets                               from Vault                     │
              │                                                              │
              └────────────────────────── LXD API Query ───────────────────┘
```

### Directory Structure

```
.
├── app/                           # Go application source code
│   ├── main.go                   # HTTP server with LXD + Vault client logic
│   ├── index.html                # Go HTML template for dashboard
│   ├── dockerfile                # Multi-stage Docker build (Alpine final)
│   ├── go.mod                    # Go module dependencies
│   └── go.sum                    # Dependency checksums
├── helm-chart/                   # Kubernetes Helm deployment manifests
│   ├── Chart.yaml                # Helm package metadata
│   ├── values.yaml               # Deployment configuration values
│   └── templates/
│       ├── deployment.yaml       # K8s Deployment manifest
│       ├── service.yaml          # K8s Service manifest
│       └── ingress.yaml.txt      # Ingress template (uncommented)
├── .gitignore                    # Git ignore patterns
├── .remember/                    # Claude agent session data (do not commit)
└── README.md                     # Project documentation
```

---

## Build Commands

### 1. Build Docker Image

```bash
docker build -t localhost:32000/go-lxd-vm-dashboard:latest ./app
```

### 2. Push to Registry

```bash
docker push localhost:32000/go-lxd-vm-dashboard:latest
```

---

## Deploy Commands

### Setup OpenBao/Vault Credentials Secret

```bash
microk8s kubectl create secret generic openbao-creds \
  --from-literal=VAULT_ADDR=https://<your-vault-addr>:8200 \
  --from-literal=VAULT_ROLE_ID=<role-id> \
  --from-literal=VAULT_SECRET_ID=<secret-id> \
  --namespace dev
```

### Configure Helm Values

Edit `helm-chart/values.yaml`:
- Set `lxd.address` to your LXD API endpoint (e.g., `"https://10.0.0.162:8443"`)
- Update `ingress.host` with your desired hostname
- Ensure `existingSecretName` matches the K8s secret name

### Deploy Application

```bash
microk8s helm upgrade --install go-lxd-vm-dashboard ./helm-chart --namespace dev
```

---

## Run Single Commands for Development

### Run Go App Locally (for testing)

```bash
cd /home/jeffrey/Documents/final-gitea-terraform-ansible/go-lxd-vm-dashboard/app
go run .
```

### Format Go Code

```bash
cd app && go fmt ./...
```

### Validate Go Dependencies

```bash
cd app && go mod verify
```

### Run Linter (if desired)

```bash
cd app && golangci-lint run ./...
```

---

## Update Deployment After Rebuild

After building and pushing a new image:

```bash
# 1. Restart the deployment to pick up new image
microk8s kubectl rollout restart deployment go-lxd-vm-dashboard -n dev

# 2. Verify rollout status
microk8s kubectl rollout status deployment/go-lxd-vm-dashboard -n dev --watch

# 3. Check logs
microk8s kubectl logs -f deployment/go-lxd-vm-dashboard -n dev
```

---

## OpenBao/Vault Setup

The app reads the LXD client certificate and key from the KV path `homelab/data/lxd`.

### Required Vault Secret Structure

Create the secret at `homelab/data/lxd` with:

```json
{
  "client_cert": "<PEM string>",
  "client_key": "<PEM string>"
}
```

### AppRole Policy

The AppRole must have a policy granting `read` on the LXD secrets path:

```hcl
path "homelab/data/lxd" {
  capabilities = ["read"]
}
```

---

## Helm Values Reference

| Key | Default | Description |
|---|---|---|
| `image.repository` | `localhost:32000/go-lxd-vm-dashboard` | Container image repository |
| `image.tag` | `latest` | Image tag (use versioned tags for reproducible deployments) |
| `image.pullPolicy` | `Always` | Kubernetes image pull policy |
| `service.type` | `ClusterIP` | Kubernetes Service type (use LoadBalancer/NodePort for external access) |
| `service.port` | `80` | External service port (exposed via ingress or load balancer) |
| `service.targetPort` | `8080` | Container port the app listens on |
| `lxd.address` | _(required)_ | LXD API endpoint URL (e.g., `"https://10.0.0.162:8443"`) |
| `existingSecretName` | `openbao-creds` | Name of pre-created K8s secret containing Vault credentials |
| `ingress.enabled` | `true` | Toggle Ingress resource creation |
| `ingress.className` | `public` | IngressClass name for your cluster |
| `ingress.host` | `lxd-dashboard.local` | Hostname for ingress rule (add to /etc/hosts or use wildcard SSL) |

---

## Troubleshooting Commands

### Check Deployment Status

```bash
microk8s kubectl get pods -n dev
microk8s kubectl describe pod -n dev <pod-name>
```

### View Application Logs

```bash
microk8s kubectl logs -f deployment/go-lxd-vm-dashboard -n dev
```

### Check Vault Authentication Flow

Temporarily enable debug logging in the app by modifying `main.go`:
- Add `log.Printf("DEBUG: VAULT_ADDR=%s", os.Getenv("VAULT_ADDR"))`
- Rebuild and redeploy to see authentication errors

### Common Issues

1. **Vault connection fails**: Check `VAULT_SKIP_VERIFY=true` is set (in deployment.yaml)
2. **LXD connection errors**: Verify TLS cert/key are correctly stored in Vault
3. **Ingress not working**: Ensure MicroK8s add-ons `dns`, `registry`, `ingress`, `helm3` are enabled
4. **Service unreachable**: Check if service type is `ClusterIP` vs `LoadBalancer`

---

## Helm Chart Templates

The deployment uses Go templating:
- `.Release.Name` → Deployment and pod app label
- `.Values.image.repository:tag` → Docker image reference
- `.Values.existingSecretName` → K8s secret for Vault credentials
- `.Values.lxd.address` → LXD API endpoint

---

## Security Notes

1. **Vault AppRole Auth**: Credentials are fetched at request time, never baked into the image
2. **TLS Certificates**: Stored encrypted in OpenBao/Vault KV secrets
3. **App Role Secret ID**: Rotated periodically; rotate if compromised
4. **VAULT_SKIP_VERIFY**: Enabled for development; disable in production with proper CA certs

---

## Future Roadmap (from README)

- [ ] Multi-project LXD support (`client.UseProject("production")`)
- [ ] Gitea Actions CI/CD pipeline (build → push → rolling restart)
- [ ] Add memory/CPU metrics from LXD state API
- [ ] Dark mode for the dashboard
