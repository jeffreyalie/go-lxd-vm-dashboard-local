# Makefile for go-lxd-vm-dashboard deployment tasks
# Usage: make <target>

.PHONY: build push deploy logs restart verify clean help

# Default target
help:
	@echo "Available targets:"
	@echo "  build    - Build Docker image"
	@echo "  push     - Push to registry"
	@echo "  deploy   - Deploy using docker-compose.yml"
	@echo "  logs     - View container logs (follow mode)"
	@echo "  restart  - Restart deployment"
	@echo "  verify   - Check Go dependencies and validate code"
	@echo "  clean    - Clean build artifacts"
	@echo "  format   - Format Go code"

# Build Docker image
build:
	@echo "Building go-lxd-vm-dashboard image..."
	docker build -t localhost:32000/go-lxd-vm-dashboard:latest ./app

# Push to registry
push: build
	@echo "Pushing to registry..."
	docker push localhost:32000/go-lxd-vm-dashboard:latest

# Deploy using docker-compose
deploy:
	@echo "Deploying with docker-compose..."
	microk8s kubectl apply -f docker-compose.yml

# Helm deployment (alternative)
helm-deploy:
	@echo "Deploying with Helm..."
	microk8s helm upgrade --install go-lxd-vm-dashboard ./../helm-chart --namespace dev

# View logs
logs:
	@echo "Viewing container logs..."
	microk8s kubectl logs -f deployment/go-lxd-vm-dashboard -n dev

# Restart deployment (pick up new image)
restart:
	@echo "Rolling restart to pick up changes..."
	microk8s kubectl rollout restart deployment/go-lxd-vm-dashboard -n dev
	microk8s kubectl rollout status deployment/go-lxd-vm-dashboard -n dev --watch

# Verify code and dependencies
verify:
	@echo "Verifying Go dependencies..."
	cd app && go mod verify
	@echo "Formatting code..."
	cd app && go fmt ./...
	@echo "Verification complete"

# Format code only
format:
	@echo "Formatting Go code..."
	cd app && go fmt ./...

# Clean build artifacts
clean:
	@echo "Cleaning Docker images and build cache..."
	docker system prune -f
	@echo "Clean complete"
