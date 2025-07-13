# Docker Swarm CI/CD Quick Reference Guide

This guide provides a comprehensive reference for Docker Swarm commands used in CI/CD operations.

## 🔧 User Management

### Create cicd User

```bash
# Create user with sudo privileges
sudo adduser cicd

# Add to docker group
sudo usermod -aG docker cicd

# Add sudo privileges
sudo usermod -aG sudo cicd

# Switch to cicd user
sudo su - cicd

# Verify docker access
docker --version
docker ps
```

## 📁 Directory Structure Setup

### Create Main Directories

```bash
# Create complete directory structure
sudo mkdir -p /opt/homelab/{scripts,swarm}

# Set ownership to cicd user
sudo chown -R cicd:cicd /opt/homelab

# Set proper permissions
sudo chmod -R 755 /opt/homelab

# Verify structure
ls -la /opt/homelab/
```

## 🐳 Docker Swarm Commands

### Cluster Management

```bash
# Initialize Docker Swarm
docker swarm init --advertise-addr YOUR_MGMT_SERVER_IP

# Join worker to swarm
docker swarm join --token <worker-token> YOUR_MGMT_SERVER_IP:2377

# Get join tokens
docker swarm join-token worker
docker swarm join-token manager

# Leave swarm (run on worker node)
docker swarm leave

# Leave swarm (run on manager node)
docker swarm leave --force

# Update swarm configuration
docker swarm update --cert-expiry 87600h
```

### Node Management

```bash
# List all nodes
docker node ls

# Inspect node details
docker node inspect dev-server

# Inspect node labels
docker node inspect dev-server --format '{{ .Spec.Labels }}'

# Label nodes for environment placement
docker node update --label-add environment=dev dev-server
docker node update --label-add environment=staging staging-server
docker node update --label-add environment=prod prod-server

# Remove node labels
docker node update --label-rm environment dev-server

# Change node availability
docker node update --availability active dev-server
docker node update --availability pause dev-server
docker node update --availability drain dev-server

# Promote worker to manager
docker node promote dev-server

# Demote manager to worker
docker node demote dev-server

# Remove node from swarm
docker node rm dev-server
```

### Stack Management

```bash
# Deploy stack
docker stack deploy --with-registry-auth -c docker-stack.dev.yml my-app-dev

# Deploy stack with environment file
docker stack deploy --with-registry-auth -c docker-stack.dev.yml --env-file .env.dev my-app-dev

# Update stack (same as deploy)
docker stack deploy --with-registry-auth -c docker-stack.dev.yml my-app-dev

# List stacks
docker stack ls

# List services in stack
docker stack services my-app-dev

# Show tasks/containers in stack
docker stack ps my-app-dev

# Show detailed task information
docker stack ps my-app-dev --no-trunc

# Remove stack
docker stack rm my-app-dev

# Config validation (dry run)
docker stack deploy --with-registry-auth -c docker-stack.dev.yml my-app-dev --prune --resolve-image always
```

### Service Management

```bash
# List all services
docker service ls

# List services with filters
docker service ls --filter "name=my-app-dev"
docker service ls --filter "mode=replicated"

# Inspect service
docker service inspect my-app-dev_web

# Scale service
docker service scale my-app-dev_web=3

# Update service image
docker service update --image localhost:5000/my-app:latest my-app-dev_web

# Update service with rollback on failure
docker service update --image localhost:5000/my-app:latest --update-failure-action rollback my-app-dev_web

# Rollback service to previous version
docker service rollback my-app-dev_web

# Remove service
docker service rm my-app-dev_web

# Force update service (restart without image change)
docker service update --force my-app-dev_web
```

### Service Logs

```bash
# View service logs
docker service logs my-app-dev_web

# Follow logs in real-time
docker service logs -f my-app-dev_web

# Show last 50 lines
docker service logs --tail 50 my-app-dev_web

# Show logs with timestamps
docker service logs -t my-app-dev_web

# Show logs since specific time
docker service logs --since 2023-01-01T00:00:00 my-app-dev_web

# View logs for specific task
docker service logs my-app-dev_web.1
```

### Container Logs (from worker nodes)

```bash
# Get container ID first
docker ps

# Check logs by container ID
docker logs <container_id>

# Follow logs
docker logs -f <container_id>

# Last 100 lines
docker logs --tail 100 <container_id>

# Show logs with timestamps
docker logs -t <container_id>
```

## 🗄️ Database Migration Commands

```bash
# Get container ID first
docker ps

# Run database migration
docker exec -it <container_id> python migrate.py

# Run migration with custom environment
docker exec -e DATABASE_HOST="YOUR_DB_HOST" \
            -e DATABASE_USER="postgres" \
            -e DATABASE_PASSWORD="your_password" \
            -e APP_ENV="dev" \
            -it <container_id> python migrate.py

# Run migration on specific service
docker service exec my-app-dev_web python migrate.py
```

## 🔍 Debugging and Troubleshooting

### Service Debugging

```bash
# Check service tasks and their status
docker service ps my-app-dev_web

# Check service tasks with full details
docker service ps my-app-dev_web --no-trunc

# Check service configuration
docker service inspect my-app-dev_web --pretty

# Check service placement constraints
docker service inspect my-app-dev_web --format '{{ .Spec.TaskTemplate.Placement }}'

# Check service update configuration
docker service inspect my-app-dev_web --format '{{ .Spec.UpdateConfig }}'
```

### Network Debugging

```bash
# List networks
docker network ls

# List overlay networks
docker network ls --filter driver=overlay

# Inspect network
docker network inspect app-dev-network

# Test connectivity between services
docker exec -it <container_id> ping service-name
docker exec -it <container_id> nslookup service-name
docker exec -it <container_id> curl http://service-name:port
```

### Resource Monitoring

```bash
# Check node resources
docker node ls

# Check system resource usage
docker stats

# Check service resource usage
docker service inspect my-app-dev_web --format '{{ .Spec.TaskTemplate.Resources }}'

# Check service replicas and placement
docker service ps my-app-dev_web --format "table {{.Node}}\t{{.Name}}\t{{.CurrentState}}\t{{.DesiredState}}"
```

## 🔐 Security and Secrets

### Docker Secrets

```bash
# Create secret from file
docker secret create my-secret /path/to/secret.txt

# Create secret from stdin
echo "my-secret-value" | docker secret create my-secret -

# List secrets
docker secret ls

# Inspect secret
docker secret inspect my-secret

# Remove secret
docker secret rm my-secret

# Use secret in service
docker service create --secret my-secret nginx

# Use secret with custom target and mode
docker service create --secret source=my-secret,target=/run/secrets/custom-secret,mode=0400 nginx
```

### Registry Authentication

```bash
# Login to registry
docker login localhost:5000

# Login with credentials
docker login -u username -p password localhost:5000

# Deploy with registry auth
docker stack deploy --with-registry-auth -c docker-stack.yml stack-name

# Update service with registry auth
docker service update --with-registry-auth --image localhost:5000/my-app:latest service-name
```

## 🔄 CI/CD Specific Commands

### Deployment Script Example

```bash
#!/bin/bash
# /opt/homelab/scripts/deploy.sh

set -e

ENVIRONMENT=$1
SERVICE=$2
STACK_NAME="my-app-$ENVIRONMENT"
STACK_FILE="/opt/homelab/swarm/docker-stack.$ENVIRONMENT.yml"

if [ -z "$ENVIRONMENT" ] || [ -z "$SERVICE" ]; then
    echo "Usage: $0 <environment> <service>"
    echo "Example: $0 dev web"
    exit 1
fi

if [ ! -f "$STACK_FILE" ]; then
    echo "Stack file not found: $STACK_FILE"
    exit 1
fi

echo "Deploying $SERVICE to $ENVIRONMENT environment..."
echo "Stack: $STACK_NAME"
echo "File: $STACK_FILE"

# Deploy the stack
docker stack deploy --with-registry-auth -c "$STACK_FILE" "$STACK_NAME"

echo "Deployment completed successfully!"

# Wait for service to be ready
echo "Waiting for service to be ready..."
sleep 10

# Check service status
docker service ls --filter "name=${STACK_NAME}_${SERVICE}"

# Check if service is running
if docker service ps "${STACK_NAME}_${SERVICE}" --filter "desired-state=running" | grep -q "Running"; then
    echo "✅ Service is running successfully"
else
    echo "❌ Service deployment failed"
    docker service ps "${STACK_NAME}_${SERVICE}" --no-trunc
    exit 1
fi
```

### Health Check Commands

```bash
# Check all services health
docker service ls --format "table {{.Name}}\t{{.Mode}}\t{{.Replicas}}\t{{.Image}}"

# Check specific service health
docker service ps my-app-dev_web --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}\t{{.DesiredState}}\t{{.Error}}"

# Check service events
docker service ps my-app-dev_web --no-trunc

# Test service endpoint
curl -f http://localhost:8080/health || echo "Health check failed"
```

## 🧹 Cleanup Commands

```bash
# Remove unused images
docker image prune -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Remove all unused resources
docker system prune -f

# Remove all unused resources including volumes
docker system prune -a --volumes -f

# Remove specific stack
docker stack rm my-app-dev

# Remove stopped containers
docker container prune -f
```

## 📊 Monitoring Commands

```bash
# Show running containers
docker ps

# Show all containers (including stopped)
docker ps -a

# Show container resource usage
docker stats

# Show container resource usage for specific containers
docker stats $(docker ps --format="{{.Names}}" | grep my-app)

# Show system information
docker system info

# Show system events
docker system events

# Show system disk usage
docker system df
```

## 💾 Backup Commands

```bash
# Backup volume
docker run --rm -v volume_name:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data

# Restore volume
docker run --rm -v volume_name:/data -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /

# Export service configuration
docker service inspect my-app-dev_web > service-config.json

# Export stack configuration
docker stack config my-app-dev > stack-config.yml
```

## 🚀 Performance Optimization

```bash
# Update service with resource limits
docker service update --limit-cpu 1.0 --limit-memory 512M my-app-dev_web

# Update service with resource reservations
docker service update --reserve-cpu 0.5 --reserve-memory 256M my-app-dev_web

# Update service with restart policy
docker service update --restart-condition on-failure --restart-delay 5s --restart-max-attempts 3 my-app-dev_web

# Update service with healthcheck
docker service update --health-cmd "curl -f http://localhost:8080/health || exit 1" --health-interval 30s my-app-dev_web
```

## 🔍 Common Troubleshooting Scenarios

### Service Won't Start

```bash
# Check service tasks
docker service ps my-app-dev_web --no-trunc

# Check node labels
docker node inspect dev-server --format '{{ .Spec.Labels }}'

# Check constraints
docker service inspect my-app-dev_web --format '{{ .Spec.TaskTemplate.Placement }}'

# Check image availability
docker service inspect my-app-dev_web --format '{{ .Spec.TaskTemplate.ContainerSpec.Image }}'
```

### Service Stuck in Pending

```bash
# Check node availability
docker node ls

# Check resource constraints
docker service ps my-app-dev_web --no-trunc

# Check placement constraints
docker service inspect my-app-dev_web --format '{{ .Spec.TaskTemplate.Placement }}'
```

### Image Pull Failures

```bash
# Check registry connectivity
docker service ps my-app-dev_web --no-trunc

# Test image pull manually
docker pull localhost:5000/my-app:dev

# Check registry authentication
docker service update --with-registry-auth my-app-dev_web
```

This reference guide covers the most common Docker Swarm commands you'll need for managing your CI/CD pipeline. Keep this handy for quick reference during deployments and troubleshooting.
