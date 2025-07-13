# Basic CI/CD Pipeline for Homelab

This guide will help you set up your first CI/CD pipeline in your homelab using a simple single-server setup with Docker Compose. Perfect for beginners who want to learn CI/CD concepts without the complexity of multi-server deployments.

## What You'll Build

By the end of this guide, you'll have:

- Automated deployments triggered by code pushes to GitHub
- A local Docker registry for storing your images
- Secure remote access using Tailscale
- Environment separation (development and production)
- A simple web application deployed automatically

## Prerequisites

- 1 homelab server (can be a VM or physical machine)
- Docker installed on your server
- A GitHub account and repository
- Basic understanding of Docker and containers
- About 30 minutes to complete the setup

## Architecture Overview

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│   Your Code     │       │  GitHub Actions │       │  Homelab Server │
│   Repository    │──────▶│    (CI/CD)      │──────▶│                 │
│                 │       │                 │       │  Docker Registry│
└─────────────────┘       └─────────────────┘       │  + Application  │
                                                      └─────────────────┘
```

## Step 1: Prepare Your Server

### Install Docker

On your homelab server, install Docker:

```bash
# Update package list
sudo apt update

# Install Docker
sudo apt install -y docker.io

# Add your user to the docker group
sudo usermod -aG docker $USER

# Log out and back in, then test Docker
docker --version
```

### Install Tailscale

For secure access to your homelab from GitHub Actions:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale
sudo tailscale up

# Note your server's Tailscale IP address
tailscale ip -4
```

## Step 2: Set Up Local Docker Registry

A local registry stores your Docker images privately:

```bash
# Create directory for registry data
mkdir -p ~/docker-registry

# Run Docker registry
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v ~/docker-registry:/var/lib/registry \
  registry:2

# Test the registry
curl http://localhost:5000/v2/_catalog
```

## Step 3: Create Your Project Structure

Let's create a simple web application:

```bash
# Create project directory
mkdir ~/my-app
cd ~/my-app

# Create a simple HTML file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>My Homelab App</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
        .container { max-width: 600px; margin: 0 auto; }
        .env { background: #f0f0f0; padding: 20px; margin: 20px 0; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Welcome to My Homelab!</h1>
        <p>This application was deployed automatically using CI/CD!</p>
        <div class="env">
            <h3>Environment: <span id="env">Production</span></h3>
            <p>Deployed at: <span id="timestamp"></span></p>
        </div>
    </div>
    <script>
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:alpine

# Copy HTML file
COPY index.html /usr/share/nginx/html/

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
EOF

# Create nginx configuration
cat > nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF

# Create docker-compose files
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    image: localhost:5000/my-app:latest
    ports:
      - "80:80"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
EOF

# Create development environment
cat > docker-compose.dev.yml << 'EOF'
version: '3.8'

services:
  web:
    image: localhost:5000/my-app:dev
    ports:
      - "8080:80"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
EOF
```

## Step 4: Create CI/CD User

Create a dedicated user for CI/CD operations:

```bash
# Create cicd user
sudo useradd -m -s /bin/bash cicd

# Add to docker group
sudo usermod -aG docker cicd

# Create .ssh directory
sudo mkdir -p /home/cicd/.ssh
sudo chmod 700 /home/cicd/.ssh

# Create project directory for cicd user
sudo mkdir -p /home/cicd/deployments
sudo chown -R cicd:cicd /home/cicd/deployments
```

## Step 5: Set Up SSH Keys

On your local machine, generate SSH keys for GitHub Actions:

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "cicd@homelab" -f ~/.ssh/homelab_cicd

# Copy public key to server
cat ~/.ssh/homelab_cicd.pub
```

On your server, add the public key:

```bash
# Add public key to cicd user
sudo bash -c 'echo "YOUR_PUBLIC_KEY_HERE" >> /home/cicd/.ssh/authorized_keys'
sudo chmod 600 /home/cicd/.ssh/authorized_keys
sudo chown cicd:cicd /home/cicd/.ssh/authorized_keys
```

## Step 6: Create GitHub Repository

1. Create a new repository on GitHub
2. Clone it locally and copy your project files
3. Add and commit the files:

```bash
git add .
git commit -m "Initial commit with basic web app"
git push origin main
```

## Step 7: Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Homelab

on:
  push:
    branches: [main, dev]
  workflow_dispatch:

env:
  REGISTRY: your-tailscale-ip:5000
  IMAGE_NAME: my-app

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set environment based on branch
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "ENVIRONMENT=prod" >> $GITHUB_ENV
            echo "PORT=80" >> $GITHUB_ENV
          else
            echo "ENVIRONMENT=dev" >> $GITHUB_ENV
            echo "PORT=8080" >> $GITHUB_ENV
          fi

      - name: Setup Tailscale
        uses: tailscale/github-action@v3
        with:
          authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          tags: tag:ci

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.ENVIRONMENT }} .

      - name: Push to registry
        run: |
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.ENVIRONMENT }}

      - name: Deploy to homelab
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.HOMELAB_HOST }}
          username: cicd
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/cicd/deployments

            # Create docker-compose file if it doesn't exist
            if [ ! -f docker-compose.${{ env.ENVIRONMENT }}.yml ]; then
              curl -o docker-compose.${{ env.ENVIRONMENT }}.yml https://raw.githubusercontent.com/${{ github.repository }}/main/docker-compose.${{ env.ENVIRONMENT }}.yml
            fi

            # Pull latest image
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.ENVIRONMENT }}

            # Deploy
            docker-compose -f docker-compose.${{ env.ENVIRONMENT }}.yml up -d

            # Clean up old images
            docker image prune -f

            echo "Deployment completed! App available at http://${{ secrets.HOMELAB_HOST }}:${{ env.PORT }}"

      - name: Health check
        run: |
          sleep 10
          curl -f http://${{ secrets.HOMELAB_HOST }}:${{ env.PORT }}/health || exit 1
```

## Step 8: Configure GitHub Secrets

In your GitHub repository settings, add these secrets:

1. Go to Settings → Secrets and variables → Actions
2. Add these secrets:
   - `TAILSCALE_AUTHKEY`: Your Tailscale auth key
   - `HOMELAB_HOST`: Your server's Tailscale IP
   - `SSH_PRIVATE_KEY`: Contents of your private key file

### Get Tailscale Auth Key

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. Generate a new auth key
3. Copy the key to your GitHub secrets

## Step 9: Test Your Pipeline

1. Make a change to your `index.html` file
2. Commit and push to the `main` branch
3. Watch the GitHub Actions workflow run
4. Visit your app at `http://YOUR_TAILSCALE_IP`

For development testing:

1. Create a `dev` branch
2. Make changes and push to `dev`
3. Visit your dev app at `http://YOUR_TAILSCALE_IP:8080`

## Step 10: Monitoring and Troubleshooting

### Check Application Status

```bash
# Check running containers
docker ps

# Check application logs
docker logs container_name

# Check health
curl http://localhost/health
```

### Common Issues

1. **Docker registry connection failed**:

   ```bash
   # Check registry is running
   docker ps | grep registry

   # Test registry locally
   curl http://localhost:5000/v2/_catalog
   ```

2. **SSH connection failed**:

   ```bash
   # Test SSH connection
   ssh -i ~/.ssh/homelab_cicd cicd@YOUR_TAILSCALE_IP
   ```

3. **Docker permission denied**:
   ```bash
   # Add user to docker group
   sudo usermod -aG docker cicd
   ```

## Next Steps

Now that you have a basic CI/CD pipeline working, you can:

1. **Add more environments**: Create staging environment
2. **Implement proper secrets management**: Use Docker secrets or environment files
3. **Add monitoring**: Set up logging and monitoring for your applications
4. **Scale up**: Consider moving to Docker Swarm for high availability
5. **Add tests**: Include automated testing in your CI/CD pipeline

## Benefits of This Setup

- **Automated deployments**: Push code, get deployed automatically
- **Environment separation**: Different environments for development and production
- **Secure**: Uses Tailscale for secure remote access
- **Private**: Your images stay in your homelab
- **Simple**: Easy to understand and maintain
- **Scalable**: Can grow with your needs

This basic setup gives you a solid foundation for homelab CI/CD. As you become more comfortable, you can explore more advanced topics like Docker Swarm, multi-repository workflows, and advanced deployment strategies.
