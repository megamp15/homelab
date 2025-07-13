# Setting Up CI/CD for Your Homelab

This guide covers how to set up a comprehensive CI/CD pipeline for deploying applications to your homelab environment using GitHub Actions with both Docker Compose and Docker Swarm deployment options.

## Overview

A comprehensive CI/CD setup allows you to:

- Automatically deploy code changes to your homelab
- Maintain separation between development, staging, and production environments
- Choose between simple Docker Compose or advanced Docker Swarm deployment
- Follow security best practices for automated deployments
- Scale your setup as your needs grow

## Architecture Options

### Option 1: Single Server (Docker Compose)

- **Best for**: Beginners, small projects, learning
- **Requirements**: 1 server, Docker installed
- **Benefits**: Simple setup, easy to understand, quick to deploy

### Option 2: Multi-Server Cluster (Docker Swarm)

- **Best for**: Production workloads, high availability
- **Requirements**: Multiple servers, Docker Swarm cluster
- **Benefits**: High availability, rolling updates, load balancing

## Prerequisites

### For Docker Compose Setup

- 1 homelab server running Linux
- Docker installed on your server
- SSH access to your server
- GitHub repositories for your projects
- Tailscale set up for secure connections

### For Docker Swarm Setup

- Multiple servers (recommended: 4+ for manager and worker nodes)
- Docker Swarm cluster initialized
- Docker installed on all servers
- SSH access to all servers
- GitHub repositories for your projects
- Tailscale set up on all servers

## Step 1: Choose Your Deployment Strategy

### Docker Compose (Recommended for Beginners)

If you're new to CI/CD or have simple requirements, start with Docker Compose:

```yaml
# docker-compose.yml
version: "3.8"

services:
  web:
    image: localhost:5000/my-app:latest
    ports:
      - "80:80"
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
```

### Docker Swarm (For Advanced Users)

For production workloads requiring high availability:

```yaml
# docker-stack.yml
version: "3.8"

services:
  web:
    image: localhost:5000/my-app:latest
    ports:
      - "80:80"
    environment:
      - NODE_ENV=production
    networks:
      - app-network
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.labels.environment == production
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.environment == production
      restart_policy:
        condition: on-failure

networks:
  app-network:
    driver: overlay
    attachable: true

volumes:
  postgres_data:
    driver: local
```

## Step 2: Set Up Your Server Environment

### Create a Dedicated CI/CD User

On your homelab server(s), create a dedicated user for CI/CD operations:

```bash
# Create the cicd user
sudo adduser cicd

# Add to docker group for Docker commands
sudo usermod -aG docker cicd

# Add sudo privileges for system operations (optional)
sudo usermod -aG sudo cicd

# Create .ssh directory with proper permissions
sudo mkdir -p /home/cicd/.ssh
sudo chmod 700 /home/cicd/.ssh

# Create authorized_keys file
sudo touch /home/cicd/.ssh/authorized_keys
sudo chmod 600 /home/cicd/.ssh/authorized_keys
sudo chown -R cicd:cicd /home/cicd/.ssh

# Test switching to cicd user
sudo su - cicd

# Verify Docker access
docker --version
docker ps
```

### Set Up Directory Structure

Create directories for deployment files:

```bash
# Create directory structure
sudo mkdir -p /opt/homelab/{scripts,deployments}

# Set ownership to cicd user
sudo chown -R cicd:cicd /opt/homelab

# Set proper permissions
sudo chmod -R 755 /opt/homelab

# Verify structure
ls -la /opt/homelab/
```

## Step 3: Generate SSH Key Pair for CI/CD

On your local machine (not the server):

```bash
# Generate a key pair specifically for CI/CD
ssh-keygen -t ed25519 -C "cicd@homelab" -f ~/.ssh/homelab_cicd

# This creates:
# - homelab_cicd (private key)
# - homelab_cicd.pub (public key)
```

## Step 4: Add the Public Key to Your Server

Copy the contents of the public key and add it to the authorized_keys file:

```bash
# On your local machine, display the public key
cat ~/.ssh/homelab_cicd.pub

# Copy the output, then on your server:
sudo bash -c 'echo "ssh-ed25519 AAAA...your key here... cicd@homelab" >> /home/cicd/.ssh/authorized_keys'
```

## Step 5: Get the SSH Known Hosts Entry

To get the known hosts entry for your CI/CD platform secret:

```bash
# Replace with your server's Tailscale IP address
ssh-keyscan YOUR_SERVER_TAILSCALE_IP
```

## Step 6: Set Up CI/CD Secrets

Add these secrets to your GitHub repository:

- `HOMELAB_HOST`: Your server's Tailscale IP address
- `HOMELAB_USER`: Set to `cicd`
- `HOMELAB_SSH_KEY`: The entire contents of the private key file
- `HOMELAB_SSH_KNOWN_HOSTS`: The output from the ssh-keyscan command
- `TAILSCALE_AUTHKEY`: Tailscale authentication key for GitHub Actions
- `REGISTRY_URL`: Your local Docker registry URL (e.g., `localhost:5000`)

## Step 7: Test the Connection

Test the SSH connection from your local machine:

```bash
ssh -i ~/.ssh/homelab_cicd cicd@YOUR_SERVER_TAILSCALE_IP

# Once connected, verify Docker access
docker --version
docker ps
```

## Step 8: Set Up Local Docker Registry

### For Docker Compose

```bash
# Create registry directory
mkdir -p ~/docker-registry

# Run Docker registry
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v ~/docker-registry:/var/lib/registry \
  registry:2
```

### For Docker Swarm

```bash
# Deploy registry as a service
docker service create \
  --name registry \
  --publish 5000:5000 \
  --constraint 'node.role == manager' \
  --mount type=bind,src=/opt/docker-registry,dst=/var/lib/registry \
  registry:2
```

## Step 9: Create Deployment Scripts

### For Docker Compose

Create `/opt/homelab/scripts/deploy-compose.sh`:

```bash
#!/bin/bash

set -e

ENVIRONMENT=${1:-prod}
PROJECT_DIR="/opt/homelab/deployments"
COMPOSE_FILE="docker-compose.${ENVIRONMENT}.yml"

cd $PROJECT_DIR

if [ ! -f "$COMPOSE_FILE" ]; then
    echo "Compose file not found: $COMPOSE_FILE"
    exit 1
fi

echo "Deploying to $ENVIRONMENT environment using Docker Compose..."

# Pull latest images
docker-compose -f $COMPOSE_FILE pull

# Deploy
docker-compose -f $COMPOSE_FILE up -d

# Clean up old images
docker image prune -f

echo "Deployment completed successfully!"

# Check service status
docker-compose -f $COMPOSE_FILE ps
```

### For Docker Swarm

Create `/opt/homelab/scripts/deploy-swarm.sh`:

```bash
#!/bin/bash

set -e

ENVIRONMENT=${1:-prod}
SERVICE=${2:-web}
STACK_NAME="homelab-${ENVIRONMENT}"
STACK_FILE="/opt/homelab/deployments/docker-stack.${ENVIRONMENT}.yml"

if [ ! -f "$STACK_FILE" ]; then
    echo "Stack file not found: $STACK_FILE"
    exit 1
fi

echo "Deploying $SERVICE to $ENVIRONMENT environment using Docker Swarm..."
echo "Stack: $STACK_NAME"
echo "File: $STACK_FILE"

# Deploy the stack
docker stack deploy --with-registry-auth -c "$STACK_FILE" "$STACK_NAME"

echo "Deployment completed successfully!"

# Wait for service to be ready
echo "Waiting for service to be ready..."
sleep 15

# Check service status
docker service ls --filter "name=${STACK_NAME}"
```

Make the scripts executable:

```bash
chmod +x /opt/homelab/scripts/deploy-compose.sh
chmod +x /opt/homelab/scripts/deploy-swarm.sh
```

## Step 10: Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Homelab

on:
  push:
    branches: [main, staging, dev]
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to deploy to"
        required: true
        default: "prod"
        type: choice
        options:
          - dev
          - staging
          - prod
      deployment_type:
        description: "Deployment type"
        required: true
        default: "compose"
        type: choice
        options:
          - compose
          - swarm

env:
  REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
  PROJECT_NAME: my-app
  ENVIRONMENT: ${{ github.event.inputs.environment || (github.ref_name == 'main' && 'prod' || github.ref_name == 'staging' && 'staging' || 'dev') }}
  DEPLOYMENT_TYPE: ${{ github.event.inputs.deployment_type || 'compose' }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Tailscale
        uses: tailscale/github-action@v3
        with:
          authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          tags: tag:ci

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        run: |
          docker build -t ${{ env.REGISTRY_URL }}/${{ env.PROJECT_NAME }}:${{ env.ENVIRONMENT }} .
          docker push ${{ env.REGISTRY_URL }}/${{ env.PROJECT_NAME }}:${{ env.ENVIRONMENT }}

      - name: Deploy to homelab
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.HOMELAB_HOST }}
          username: ${{ secrets.HOMELAB_USER }}
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd /opt/homelab

            # Sync deployment files
            curl -o deployments/docker-compose.${{ env.ENVIRONMENT }}.yml \
              https://raw.githubusercontent.com/${{ github.repository }}/main/docker-compose.${{ env.ENVIRONMENT }}.yml

            if [ "${{ env.DEPLOYMENT_TYPE }}" == "swarm" ]; then
              curl -o deployments/docker-stack.${{ env.ENVIRONMENT }}.yml \
                https://raw.githubusercontent.com/${{ github.repository }}/main/docker-stack.${{ env.ENVIRONMENT }}.yml
              ./scripts/deploy-swarm.sh ${{ env.ENVIRONMENT }}
            else
              ./scripts/deploy-compose.sh ${{ env.ENVIRONMENT }}
            fi

            echo "✅ Deployment to ${{ env.ENVIRONMENT }} completed successfully"

      - name: Cleanup Tailscale
        if: always()
        run: |
          sudo tailscale logout
```

## Deployment Commands Reference

### Docker Compose Commands

```bash
# Deploy to development
./scripts/deploy-compose.sh dev

# Deploy to production
./scripts/deploy-compose.sh prod

# Check status
docker-compose ps

# View logs
docker-compose logs -f

# Scale service
docker-compose up -d --scale web=3
```

### Docker Swarm Commands

```bash
# Deploy to development
./scripts/deploy-swarm.sh dev

# Deploy to production
./scripts/deploy-swarm.sh prod

# List stacks
docker stack ls

# List services
docker service ls

# View service logs
docker service logs homelab-prod_web

# Scale service
docker service scale homelab-prod_web=3
```

## Benefits of Each Approach

### Docker Compose Benefits

- **Simple Setup**: Easy to understand and implement
- **Quick Deployment**: Fast setup for development environments
- **Resource Efficient**: Works well on a single server
- **Easy Debugging**: Simple to troubleshoot and modify

### Docker Swarm Benefits

- **High Availability**: Automatic failover and recovery
- **Zero-Downtime Deployments**: Rolling updates with rollback
- **Load Balancing**: Built-in load balancing across replicas
- **Scalability**: Easy horizontal scaling across nodes
- **Production Ready**: Suitable for production workloads

## Security Considerations

1. Use Tailscale for secure connections between GitHub Actions and your homelab
2. Store sensitive information in GitHub Secrets
3. Use a dedicated CI/CD user with limited permissions
4. Regularly rotate SSH keys and access tokens
5. Use environment-specific configurations
6. Implement proper firewall rules
7. Keep your systems updated

## Troubleshooting

### Common Issues

1. **SSH Connection Issues**:

   ```bash
   # Test SSH connection
   ssh -i ~/.ssh/homelab_cicd cicd@YOUR_SERVER_IP

   # Check SSH service
   sudo systemctl status ssh
   ```

2. **Docker Permission Issues**:

   ```bash
   # Verify user is in docker group
   groups cicd

   # Add user to docker group if needed
   sudo usermod -aG docker cicd
   ```

3. **Registry Connection Issues**:

   ```bash
   # Check registry is running
   docker ps | grep registry

   # Test registry connectivity
   curl http://localhost:5000/v2/_catalog
   ```

## Next Steps

Once you have your CI/CD pipeline working:

1. **Add monitoring**: Implement logging and monitoring for your applications
2. **Add testing**: Include automated testing in your CI/CD pipeline
3. **Implement secrets management**: Use proper secrets management for sensitive data
4. **Add more environments**: Create additional environments as needed
5. **Consider scaling**: Move from Docker Compose to Docker Swarm as your needs grow

This setup provides a solid foundation for homelab CI/CD that can grow with your needs and requirements.
