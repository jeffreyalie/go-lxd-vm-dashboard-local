# Docker Deployment Guide

## 📦 Build the Docker Image

```bash
cd /home/jeffrey/Documents/final-gitea-terraform-ansible/go-lxd-vm-dashboard/app
docker build -t localhost:32000/go-lxd-vm-dashboard:latest .
```

**Or with a versioned tag for reproducibility:**

```bash
docker build -t go-lxd-dashboard:v1.0 ./app
```

---

## 📝 Configure Environment Variables

The application reads the following environment variables at runtime:

| Variable | Required | Description | Example Value |
|----------|----------|-------------|---------------|
| `VAULT_ADDR` | ✅ Yes | HashiCorp Vault server address | `https://your-vault-address:8200` |
| `VAULT_ROLE_ID` | ✅ Yes | AppRole role ID for authentication | `xxxx-yyyy-zzzz-wwww` |
| `VAULT_SECRET_ID` | ✅ Yes | AppRole secret ID | `aaaa-bbbb-cccc-dddd` |
| `LXD_ADDR` | ✅ Yes | LXD API endpoint URL | `https://10.0.0.162:8443` |
| `PORT` | ⚠️ Optional | Port to listen on (default: 8080) | `8080` |

### Create `.env` File (Recommended)

```bash
cd /home/jeffrey/Documents/final-gitea-terraform-ansible/go-lxd-vm-dashboard/app
cat > .env << 'EOF'
VAULT_ADDR=https://your-vault-address:8200
VAULT_ROLE_ID=your-role-id-here
VAULT_SECRET_ID=your-secret-id-here
LXD_ADDR=https://10.0.0.162:8443
PORT=8080
EOF
```

---

## 🚀 Run the Container

### Basic Run Command

```bash
docker run -d \
  --name go-lxd-dashboard \
  -p 8080:8080 \
  -e VAULT_ADDR=https://your-vault-address:8200 \
  -e VAULT_ROLE_ID=your-role-id-here \
  -e VAULT_SECRET_ID=your-secret-id-here \
  -e LXD_ADDR=https://10.0.0.162:8443 \
  localhost:32000/go-lxd-vm-dashboard:latest
```

### Alternative: Using `.env` File

If your Docker setup supports it, you can use the env file:

```bash
docker run -d \
  --name go-lxd-dashboard \
  -p 8080:8080 \
  --env-file .env \
  localhost:32000/go-lxd-vm-dashboard:latest
```

### Run with Volume Persistence (Optional)

For persistent logs or configuration:

```bash
docker run -d \
  --name go-lxd-dashboard \
  -p 8080:8080 \
  -v /path/to/logs:/var/log/lxd-dashboard:ro \
  -e VAULT_ADDR=https://your-vault-address:8200 \
  -e VAULT_ROLE_ID=your-role-id-here \
  -e VAULT_SECRET_ID=your-secret-id-here \
  -e LXD_ADDR=https://10.0.0.162:8443 \
  localhost:32000/go-lxd-vm-dashboard:latest
```

---

## 🔍 Useful Docker Commands

### View Logs

```bash
docker logs -f go-lxd-dashboard
```

### Stop and Remove Container

```bash
docker stop go-lxd-dashboard
docker rm go-lxd-dashboard
```

### Restart Container

```bash
docker restart go-lxd-dashboard
```

### Check Running Containers

```bash
docker ps -a | grep go-lxd-dashboard
```

### Build with Debug Information

```bash
docker build --no-cache \
  -t localhost:32000/go-lxd-vm-dashboard:debug \
  --progress=plain \
  ./app
```

---

## 📋 Verify Deployment

### Check Application Status

```bash
# Verify deployment is running
microk8s kubectl get pods -n dev

# View application logs
docker logs go-lxd-dashboard

# Check container health
docker exec go-lxd-dashboard curl http://localhost:8080
```

### Inspect Environment Variables

```bash
docker exec go-lxd-dashboard printenv
```

---

## 🔄 Update Deployment After Rebuild

After building and pushing a new image:

```bash
# Restart the deployment to pick up new image
microk8s kubectl rollout restart deployment/go-lxd-vm-dashboard -n dev

# Verify rollout status
microk8s kubectl rollout status deployment/go-lxd-vm-dashboard -n dev --watch

# Check logs
docker logs -f go-lxd-dashboard
```

---

## 🔐 Vault Setup Required

Before running the container, you must create the secret at `homelab/data/lxd` with this structure:

```json
{
  "client_cert": "<PEM string>",
  "client_key": "<PEM string>"
}
```

### AppRole Policy Required

The AppRole must have a policy granting `read` on the LXD secrets path:

```hcl
path "homelab/data/lxd" {
  capabilities = ["read"]
}
```

---

## 🐳 Push Image to Registry

After building, push the image to your registry:

```bash
docker push localhost:32000/go-lxd-vm-dashboard:latest
```

---

## ⚠️ Security Notes

1. **Vault Skip Verify**: The app uses `VAULT_SKIP_VERIFY=true` by default for development — disable in production with proper CA certs
2. **TLS Certificates**: Must be correctly stored in Vault KV secrets at `homelab/data/lxd`
3. **AppRole Secrets**: Rotate periodically; rotate immediately if compromised
4. **Internal Network Only**: The app uses `InsecureSkipVerify: true` for LXD — only acceptable on trusted internal networks

---

## 🧪 Development Mode (Localhost Testing)

To test without Docker:

```bash
cd /home/jeffrey/Documents/final-gitea-terraform-ansible/go-lxd-vm-dashboard/app
go run .
```

This runs the app directly for local testing and debugging.

---

## 📈 Image Details

The Dockerfile uses a multi-stage build:

- **Builder**: `golang:1.26.2-alpine` — Compiles Go binary
- **Final**: `alpine:latest` — Lightweight ~5-10MB production image
- **Exposes**: Port 8080 (container) mapped to port 80 (Kubernetes service)

---

## 📦 Helm Alternative Deployment

For Kubernetes/MicroK8s deployment, see the `helm-chart/` directory.
