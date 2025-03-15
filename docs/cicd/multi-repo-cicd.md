# Multi-Repository CI/CD with Local Docker Registry

This guide explains how to set up a comprehensive CI/CD pipeline across multiple GitHub repositories using a local Docker registry in your homelab.

## Overview

The CI/CD structure consists of:

1. **Source Repositories**: Individual project repositories that build Docker images
2. **Local Docker Registry**: A private registry in your homelab for storing images
3. **Deployment Repository**: A central repository that handles the actual deployment
4. **GitHub Actions**: Workflows that coordinate the build, push, and deployment process

This approach separates concerns between code development and infrastructure deployment, providing a clean and maintainable architecture.

## Architecture Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Source Repo 1  │     │  Source Repo 2  │     │  Source Repo 3  │
│  (Frontend)     │     │  (API)          │     │  (Backend)      │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │                       │                       │
         │                       ▼                       │
         │              ┌─────────────────┐              │
         └─────────────►│  GitHub Actions │◄─────────────┘
                        │  Build & Push   │
                        └────────┬────────┘
                                 │
                                 │ Tailscale Secure Connection
                                 ▼
                        ┌─────────────────┐
                        │  Local Docker   │
                        │    Registry     │
                        └────────┬────────┘
                                 │
                                 │ Repository Dispatch Event
                                 ▼
                        ┌─────────────────┐
                        │  Deployment     │
                        │  Repository     │
                        └────────┬────────┘
                                 │
                                 │ Pull & Deploy
                                 ▼
                        ┌─────────────────┐
                        │  Homelab        │
                        │  Services       │
                        └─────────────────┘
```

## Components

### 1. Source Repositories

Each source repository contains:
- Application code for a specific service
- Dockerfile for building the service
- GitHub Actions workflow for CI/CD

For detailed examples of GitHub Actions workflow files for each source repository, see the [build-push.yml](scripts/workflows/build-push.yml) example in the scripts directory.

Each workflow follows the same general pattern but is customized for the specific service:

1. **Checkout code** from the repository
2. **Set up the appropriate runtime environment** (Node.js for Frontend/API, Python for Backend)
3. **Install dependencies and run tests** specific to each service
4. **Build a Docker image** with the service-specific name
5. **Push the image** to your local Docker registry
6. **Trigger deployment** with the specific service name

This approach allows each repository to maintain its own CI/CD workflow while still integrating with the central deployment process.

### 2. Local Docker Registry

The local Docker registry:
- Stores all your Docker images privately in your homelab
- Is accessible via Tailscale for secure remote access
- Provides faster image pulls for deployments
- Avoids rate limits from public registries

See [Docker Registry Setup Guide](../services/docker-registry.md) for details on setting up your local registry.

### 3. Deployment Repository

The deployment repository:
- Contains Docker Compose files for each environment
- Listens for repository dispatch events from source repositories
- Pulls the latest images from your local registry
- Deploys or updates services using Docker Compose

For detailed examples of Docker Compose files for each environment, see:
- [docker-compose.dev.yml](scripts/docker-compose/docker-compose.dev.yml) - Development environment
- [docker-compose.staging.yml](scripts/docker-compose/docker-compose.staging.yml) - Staging environment
- [docker-compose.prod.yml](scripts/docker-compose/docker-compose.prod.yml) - Production environment

These Docker Compose files follow a consistent pattern but with environment-specific configurations:

1. **Development (`docker-compose.dev.yml`)**:
   - Uses different ports to avoid conflicts with other environments
   - Enables debugging features
   - Uses development-specific database
   - Sets verbose logging

2. **Staging (`docker-compose.staging.yml`)**:
   - Middle ground between development and production
   - Uses isolated network and database
   - Maintains service naming convention with `-staging` suffix

3. **Production (`docker-compose.prod.yml`)**:
   - Uses standard ports (80 for frontend)
   - Disables debugging features
   - Includes resource limits for stability
   - Uses production-specific database
   - Sets minimal logging

Each environment has its own network and volume names to ensure complete isolation between environments.

### 4. GitHub Actions Workflow for Deployment

The deployment repository should have a workflow that listens for repository dispatch events. For a detailed example, see [deploy.yml](scripts/workflows/deploy.yml) in the scripts directory.

Key points about the deployment workflow:

1. **Event Types**: The workflow listens for specific event types that include both the service name and environment (e.g., `deploy-frontend-staging`, `deploy-api-prod`).

2. **Environment Extraction**: The workflow extracts the environment (dev, staging, prod) and service name from the event payload.

3. **File Selection**: The workflow uses the environment variable to select the correct Docker Compose file.

4. **Service Targeting**: The command targets only the specific service that was updated, using the naming convention `{service}-{environment}` (e.g., `frontend-staging`, `api-prod`).

This approach allows you to maintain separate configuration files for each environment while using a single workflow to handle deployments to any environment.

## Setting Up the Multi-Repository CI/CD Pipeline

### Prerequisites

1. Multiple GitHub repositories for your services
2. A local Docker registry in your homelab
3. A dedicated deployment repository
4. GitHub Personal Access Token (PAT) with repository access

### Repository Structure

This CI/CD pipeline uses a multi-repository approach with clear separation of concerns:

1. **Source Repositories** - Where your application code lives
   - Each service (frontend, API, backend) has its own repository
   - Each repository contains its own Dockerfile and CI/CD workflow
   - The workflow builds and pushes Docker images, then triggers deployment

2. **Deployment Repository** - Where your deployment configuration lives
   - Contains Docker Compose files for each environment
   - Contains a workflow that responds to deployment triggers
   - Handles the actual deployment to your homelab

For example files demonstrating this structure, see the [scripts](scripts/) directory:
- [Source repository examples](scripts/source-repos/) - Frontend, API, and Backend repositories
- [Deployment repository example](scripts/deployment-repo/) - Central deployment repository

### Service Types Explained

In our example, we use three common service types that represent a typical modern web application architecture:

1. **Frontend** - User interface layer
   - A web application that users interact with directly (React, Vue, Angular, etc.)
   - Runs in the user's browser and communicates with the API
   - Example: An e-commerce website's user interface with product listings, shopping cart, checkout pages
   - Typically served as static files via Nginx or similar web server

2. **API** (Application Programming Interface) - Communication layer
   - Handles HTTP requests from the frontend and provides data
   - Implements authentication, validation, and business rules
   - Example: REST or GraphQL endpoints for user authentication, product searches, order processing
   - Often built with Node.js (Express), Python (FastAPI), or similar frameworks

3. **Backend** - Business logic and processing layer
   - Handles complex operations, data processing, and integrations
   - Often runs background jobs or processes resource-intensive tasks
   - Example: Order fulfillment, payment processing, inventory management, analytics
   - Typically built with Python, Java, Go, or other server-side languages

This separation allows teams to:
- Work independently on different components
- Scale each component based on its specific needs
- Use the most appropriate technology for each purpose
- Deploy and update services independently

### Step 1: Configure Source Repositories

For each source repository:

1. Set up the repository structure:
   ```
   service-repo/
   ├── src/                      # Source code for your service
   ├── .github/
   │   └── workflows/
   │       └── build-push.yml    # CI/CD workflow file
   ├── Dockerfile                # Instructions for building the Docker image
   ├── .dockerignore             # Files to exclude from Docker build
   ├── package.json              # For Node.js projects
   └── README.md
   ```

2. Create a `Dockerfile` appropriate for your service. Examples:

   **Frontend (React/Node.js):**
   ```dockerfile
   # Build stage
   FROM node:18-alpine as build
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build

   # Production stage
   FROM nginx:alpine
   COPY --from=build /app/build /usr/share/nginx/html
   COPY nginx.conf /etc/nginx/conf.d/default.conf
   EXPOSE 80
   CMD ["nginx", "-g", "daemon off;"]
   ```

   **API (Node.js):**
   ```dockerfile
   FROM node:18-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci --only=production
   COPY . .
   EXPOSE 3000
   CMD ["node", "server.js"]
   ```

   **Backend (Python):**
   ```dockerfile
   FROM python:3.10-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   EXPOSE 4000
   CMD ["python", "app.py"]
   ```

3. Add the GitHub Actions workflow file (`.github/workflows/build-push.yml`). See [scripts/source-repos/frontend/.github/workflows/build-push.yml](scripts/source-repos/frontend/.github/workflows/build-push.yml) for a complete example.

4. Configure repository secrets:
   - `HOMELAB_HOST`: Your server's Tailscale IP address
   - `HOMELAB_USER`: Your CI/CD user (e.g., cicd-bot)
   - `HOMELAB_SSH_KEY`: SSH private key for the CI/CD user
   - `HOMELAB_SSH_KNOWN_HOSTS`: SSH known hosts entry for your server
   - `TAILSCALE_AUTHKEY`: Tailscale auth key
   - `CICD_PAT`: GitHub Personal Access Token for triggering the deployment repository

This setup ensures that:
- Each service repository contains everything needed to build a Docker image
- The CI/CD workflow can build and push the image to your local registry
- The workflow can trigger a deployment in the central deployment repository

### Step 2: Set Up the Deployment Repository

1. Create a new GitHub repository for deployments (e.g., `myorg/cicd`)

2. Set up the repository structure:
   ```
   cicd/
   ├── .github/
   │   └── workflows/
   │       └── deploy.yml
   ├── docker-compose.dev.yml
   ├── docker-compose.staging.yml
   ├── docker-compose.prod.yml
   └── README.md
   ```

3. Create Docker Compose files for each environment:
   - `docker-compose.dev.yml` - Development environment configuration ([example](scripts/deployment-repo/docker-compose.dev.yml))
   - `docker-compose.staging.yml` - Staging environment configuration ([example](scripts/deployment-repo/docker-compose.staging.yml))
   - `docker-compose.prod.yml` - Production environment configuration ([example](scripts/deployment-repo/docker-compose.prod.yml))

   Each file should follow the structure shown in the examples, with environment-specific settings.

4. Add the deployment workflow file (`.github/workflows/deploy.yml`) that listens for repository dispatch events from source repositories. See [scripts/deployment-repo/.github/workflows/deploy.yml](scripts/deployment-repo/.github/workflows/deploy.yml) for a complete example.

5. Configure repository secrets (similar to source repositories, plus any environment-specific secrets):
   - `HOMELAB_HOST`: Your server's Tailscale IP address
   - `HOMELAB_USER`: Your CI/CD user (e.g., cicd-bot)
   - `HOMELAB_SSH_KEY`: SSH private key for the CI/CD user
   - `HOMELAB_SSH_KNOWN_HOSTS`: SSH known hosts entry for your server
   - `TAILSCALE_AUTHKEY`: Tailscale auth key
   - `POSTGRES_PASSWORD`: Database password
   - Any other environment-specific secrets needed by your services

6. Create the deployment directories on your homelab server:
   ```bash
   # On your homelab server
   sudo mkdir -p /opt/deployments/{dev,staging,prod}/cicd
   sudo chown -R your-user:your-group /opt/deployments
   ```

   This setup allows the CI/CD pipeline to sync files directly to environment-specific directories.

This setup ensures that:
- Each environment has its own isolated Docker Compose configuration
- Environment variables are set directly in the GitHub workflow
- The deployment workflow can target specific services in specific environments
- Files are synced directly to your homelab server using rsync

### Step 3: Configure Repository Dispatch Events

The repository dispatch mechanism allows one repository to trigger a workflow in another repository. This requires:

1. A GitHub Personal Access Token (PAT) with `repo` scope
2. The `peter-evans/repository-dispatch` action in your source repository workflows
3. A workflow in the deployment repository that listens for repository dispatch events

To create a PAT with the required permissions:
1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate a new token with the `repo` scope
3. Save this token securely and add it as the `CICD_PAT` secret in each source repository

The repository dispatch event in the source repository workflow should include:
```yaml
- name: Trigger CICD Deployment
  uses: peter-evans/repository-dispatch@v2
  with:
    token: ${{ secrets.CICD_PAT }}
    repository: myorg/cicd
    event-type: deploy-${{ env.SERVICE_NAME }}-${{ env.ENVIRONMENT }}
    client-payload: '{"service": "${{ env.SERVICE_NAME }}", "environment": "${{ env.ENVIRONMENT }}"}'
```

And the deployment repository workflow should listen for these events:
```yaml
on:
  repository_dispatch:
    types: 
      - deploy-frontend-dev
      - deploy-api-dev
      - deploy-backend-dev
      - deploy-frontend-staging
      - deploy-api-staging
      - deploy-backend-staging
      - deploy-frontend-prod
      - deploy-api-prod
      - deploy-backend-prod
```

## Benefits of This Approach

1. **Separation of Concerns**: Code development is separate from deployment configuration
2. **Environment Isolation**: Staging and production environments are clearly separated
3. **Secure Image Transfer**: Images are transferred securely via Tailscale
4. **Private Registry**: Images are stored privately in your homelab
5. **Centralized Deployment**: All deployments are managed from a single repository
6. **Automated Workflow**: Code changes automatically trigger builds and deployments
7. **Scalable Architecture**: Easy to add new services or environments

## Security Considerations

1. Use Tailscale for secure connections between GitHub Actions and your homelab
2. Store sensitive information in GitHub Secrets
3. Use a dedicated CI/CD user with limited permissions
4. Regularly rotate SSH keys and access tokens
5. Use the `tag:ci` option in Tailscale to identify and audit CI/CD connections

## Troubleshooting

### Image Push Failures

If you encounter issues pushing images to your local registry:

```bash
# Check if the registry is running
ssh cicd-bot@homelab "docker ps | grep registry"

# Verify registry connectivity
ssh cicd-bot@homelab "curl -X GET http://localhost:5000/v2/_catalog"

# Check Docker daemon logs
ssh cicd-bot@homelab "sudo journalctl -u docker"
```

### Deployment Failures

If deployments fail:

```bash
# Check Docker Compose logs
ssh cicd-bot@homelab "cd /opt/deployments && docker compose -f docker-compose.staging.yml logs"

# Verify environment variables
ssh cicd-bot@homelab "cd /opt/deployments && docker compose -f docker-compose.staging.yml config"

# Check container status
ssh cicd-bot@homelab "docker ps -a"
```

