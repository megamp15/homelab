# Docker Swarm Setup and Management Guide

This guide covers the complete setup and management of a Docker Swarm cluster for your homelab CI/CD pipeline.

## Overview

Our Docker Swarm cluster consists of:

- **Management Server**: Swarm manager node, Docker registry, databases
- **Development Server**: Worker node for development environment
- **Staging Server**: Worker node for staging environment
- **Production Server**: Worker node for production environment

All servers are connected via Tailscale for secure communication.

## Prerequisites

- 4 servers (VMs or physical machines) running Linux
- Docker installed on all servers
- Tailscale installed and configured on all servers
- SSH access to all servers
- Basic understanding of Docker concepts

## Step 1: Initialize Docker Swarm

### On the Management Server

Initialize the Docker Swarm cluster:

```bash
# Initialize Docker Swarm with the management server's Tailscale IP
sudo docker swarm init --advertise-addr YOUR_MGMT_SERVER_IP

# Output will show something like:
# To add a worker to this swarm, run the following command:
# docker swarm join --token SWMTKN-1-xyz... YOUR_MGMT_SERVER_IP:2377
```

Get the join token for worker nodes:

```bash
# Get worker join token
sudo docker swarm join-token worker

# Get manager join token (for future manager nodes)
sudo docker swarm join-token manager
```

### On Each Worker Server

Join the Docker Swarm cluster using the token from the manager:

```bash
# Use the exact command from the manager node output
sudo docker swarm join --token SWMTKN-1-xyz... YOUR_MGMT_SERVER_IP:2377
```

## Step 2: Verify Swarm Cluster

On the management server, verify all nodes joined successfully:

```bash
# List all nodes in the swarm
sudo docker node ls

# Expected output:
# ID                            HOSTNAME       STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
# abc123...                     mgmt-server    Ready     Active         Leader           20.10.21
# def456...                     dev-server     Ready     Active                          20.10.21
# ghi789...                     staging-server Ready     Active                          20.10.21
# jkl012...                     prod-server    Ready     Active                          20.10.21
```

## Step 3: Label Nodes for Environment Placement

Label each worker node to enable environment-specific deployments:

```bash
# Label development server
sudo docker node update --label-add environment=dev dev-server

# Label staging server
sudo docker node update --label-add environment=staging staging-server

# Label production server
sudo docker node update --label-add environment=prod prod-server

# Optional: Label management server (usually doesn't run workloads)
sudo docker node update --label-add environment=mgmt mgmt-server
```

Verify the labels:

```bash
# Check labels for each node
sudo docker node inspect dev-server --format '{{ .Spec.Labels }}'
sudo docker node inspect staging-server --format '{{ .Spec.Labels }}'
sudo docker node inspect prod-server --format '{{ .Spec.Labels }}'

# Expected output for dev-server:
# map[environment:dev]
```

## Step 4: Configure Node Constraints (Optional)

Prevent workloads from running on the management server:

```bash
# Drain the management server (prevent new tasks from being scheduled)
sudo docker node update --availability drain mgmt-server

# Or set to active if you want to allow workloads
sudo docker node update --availability active mgmt-server
```

## Step 5: Create Overlay Networks

Create overlay networks for inter-service communication:

```bash
# Create networks for each environment
sudo docker network create --driver overlay --attachable app-dev-network
sudo docker network create --driver overlay --attachable app-staging-network
sudo docker network create --driver overlay --attachable app-prod-network

# Create a management network
sudo docker network create --driver overlay --attachable mgmt-network
```

List created networks:

```bash
sudo docker network ls --filter driver=overlay
```

## Step 6: Set Up CI/CD User

Create a dedicated user for CI/CD operations on the management server:

```bash
# Create cicd user with sudo privileges
sudo adduser cicd
sudo usermod -aG docker cicd
sudo usermod -aG sudo cicd

# Test Docker access
sudo su - cicd
docker --version
docker node ls
```

## Step 7: Create Directory Structure

Set up directories for deployment files:

```bash
# Create directory structure
sudo mkdir -p /opt/homelab/{scripts,swarm}

# Set ownership to cicd user
sudo chown -R cicd:cicd /opt/homelab

# Set permissions
sudo chmod -R 755 /opt/homelab

# Verify structure
ls -la /opt/homelab/
```

## Step 8: Configure Docker Registry

Set up a local Docker registry on the management server:

```bash
# Create registry storage directory
sudo mkdir -p /opt/docker-registry/data
sudo chown -R cicd:cicd /opt/docker-registry

# Run Docker registry as a service
sudo docker service create \
  --name registry \
  --publish 5000:5000 \
  --constraint 'node.labels.environment == mgmt' \
  --mount type=bind,src=/opt/docker-registry/data,dst=/var/lib/registry \
  registry:2
```

Test the registry:

```bash
# Check registry health
curl -X GET http://localhost:5000/v2/_catalog

# Should return: {"repositories":[]}
```

## Step 9: Set Up Database (Optional)

Deploy a database on the management server if needed:

```bash
# Create database storage directory
sudo mkdir -p /opt/postgres/data
sudo chown -R cicd:cicd /opt/postgres

# Create PostgreSQL service
sudo docker service create \
  --name postgres \
  --publish 5432:5432 \
  --constraint 'node.labels.environment == mgmt' \
  --mount type=bind,src=/opt/postgres/data,dst=/var/lib/postgresql/data \
  --env POSTGRES_PASSWORD=your-secure-password \
  --env POSTGRES_USER=postgres \
  postgres:13
```

## Step 10: Test Deployment

Create a simple test stack to verify the setup:

```yaml
# /opt/homelab/swarm/test-stack.yml
version: "3.8"

services:
  nginx-dev:
    image: nginx:alpine
    ports:
      - "8080:80"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.environment == dev
    networks:
      - app-dev-network

  nginx-staging:
    image: nginx:alpine
    ports:
      - "8081:80"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.environment == staging
    networks:
      - app-staging-network

  nginx-prod:
    image: nginx:alpine
    ports:
      - "8082:80"
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.environment == prod
    networks:
      - app-prod-network

networks:
  app-dev-network:
    external: true
  app-staging-network:
    external: true
  app-prod-network:
    external: true
```

Deploy the test stack:

```bash
# Deploy test stack
sudo docker stack deploy -c /opt/homelab/swarm/test-stack.yml test-stack

# Check deployment
sudo docker stack ls
sudo docker service ls
sudo docker stack ps test-stack

# Test connectivity
curl http://YOUR_MGMT_SERVER_IP:8080  # Should show nginx welcome page
curl http://YOUR_MGMT_SERVER_IP:8081  # Should show nginx welcome page
curl http://YOUR_MGMT_SERVER_IP:8082  # Should show nginx welcome page

# Clean up test stack
sudo docker stack rm test-stack
```

## Swarm Management Commands

### Node Management

```bash
# List all nodes
docker node ls

# Inspect node details
docker node inspect dev-server

# Update node labels
docker node update --label-add key=value dev-server
docker node update --label-rm key dev-server

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

### Service Management

```bash
# List services
docker service ls

# Inspect service
docker service inspect my-app-dev_web

# Scale service
docker service scale my-app-dev_web=3

# Update service image
docker service update --image localhost:5000/my-app:latest my-app-dev_web

# View service logs
docker service logs my-app-dev_web

# Remove service
docker service rm my-app-dev_web
```

### Stack Management

```bash
# Deploy stack
docker stack deploy -c docker-stack.yml stack-name

# List stacks
docker stack ls

# List services in stack
docker stack services stack-name

# View stack tasks
docker stack ps stack-name

# Remove stack
docker stack rm stack-name
```

## Troubleshooting

### Common Issues

1. **Node not joining swarm**:

   ```bash
   # Check firewall ports (2377/tcp, 7946/tcp/udp, 4789/udp)
   sudo ufw allow 2377/tcp
   sudo ufw allow 7946
   sudo ufw allow 4789/udp

   # Check Docker daemon
   sudo systemctl status docker
   sudo journalctl -u docker
   ```

2. **Services not starting**:

   ```bash
   # Check service tasks
   docker service ps service-name --no-trunc

   # Check node labels
   docker node inspect node-name --format '{{ .Spec.Labels }}'

   # Check constraints
   docker service inspect service-name --format '{{ .Spec.TaskTemplate.Placement }}'
   ```

3. **Network connectivity issues**:

   ```bash
   # List networks
   docker network ls

   # Inspect network
   docker network inspect network-name

   # Test connectivity between containers
   docker exec -it container-name ping other-container-name
   ```

### Maintenance Tasks

```bash
# Update Docker Swarm join tokens
docker swarm join-token --rotate worker
docker swarm join-token --rotate manager

# Prune unused resources
docker system prune -f
docker volume prune -f
docker network prune -f

# Backup swarm configuration
docker swarm ca --cert-expiry 87600h > swarm-ca-cert.pem
docker swarm ca --ca-cert swarm-ca-cert.pem --ca-key swarm-ca-key.pem
```

## Security Best Practices

1. **Use TLS encryption** (enabled by default in Docker Swarm)
2. **Rotate join tokens regularly**
3. **Use secrets for sensitive data**:

   ```bash
   # Create secret
   echo "secret-value" | docker secret create my-secret -

   # Use in service
   docker service create --secret my-secret image-name
   ```

4. **Implement proper firewall rules**
5. **Use Tailscale for secure communication**
6. **Regular security updates**

## Monitoring and Logging

### Basic Monitoring

```bash
# Check cluster health
docker node ls
docker service ls

# Monitor resource usage
docker stats

# Check service logs
docker service logs service-name
```

### Advanced Monitoring

Consider setting up monitoring tools like:

- Prometheus + Grafana for metrics
- ELK stack for log aggregation
- Docker Swarm Visualizer for cluster visualization

## Backup and Recovery

1. **Regular backups** of critical data volumes
2. **Document your swarm configuration**
3. **Test recovery procedures**
4. **Keep configuration files in version control**

This Docker Swarm setup provides a robust foundation for your CI/CD pipeline with high availability, scalability, and security.
