# Multi-Repository CI/CD with Local Docker Registry

This guide explains how to set up a comprehensive CI/CD pipeline across multiple GitHub repositories using a local Docker registry in your homelab. This approach works with both Docker Compose and Docker Swarm deployments.

## Overview

The multi-repository CI/CD structure consists of:

1. **Source Repositories**: Individual project repositories that build Docker images
2. **Local Docker Registry**: A private registry in your homelab for storing images
3. **Deployment Repository**: A central repository that handles the actual deployment
4. **GitHub Actions**: Workflows that coordinate the build, push, and deployment process

This approach separates concerns between code development and infrastructure deployment, providing a clean and maintainable architecture suitable for any homelab setup.

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
                        │  Homelab Server │
                        │  (or Cluster)   │
                        │  - Docker       │
                        │    Registry     │
                        │  - Database     │
                        │  - Services     │
                        └────────┬────────┘
                                 │
                                 │ Repository Dispatch Event
                                 ▼
                        ┌─────────────────┐
                        │  Deployment     │
                        │  Repository     │
                        │  (Infrastructure│
                        │   Configuration)│
                        └────────┬────────┘
                                 │
                                 │ Deploy via Docker Compose or Swarm
                                 ▼
                        ┌─────────────────┐
                        │  Running        │
                        │  Applications   │
                        │  in Homelab     │
                        └─────────────────┘
```

## Components

### 1. Source Repositories

Each source repository contains:

- Application code for a specific service
- Dockerfile for building the service
- GitHub Actions workflow for CI/CD

Each workflow follows the same general pattern but is customized for the specific service:

1. **Checkout code** from the repository
2. **Set up the appropriate runtime environment** (Node.js for Frontend/API, Python for Backend)
3. **Install dependencies and run tests** specific to each service
4. **Build a Docker image** with the service-specific name
5. **Push the image** to your local Docker registry
6. **Trigger deployment** with the specific service name and environment

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

- Contains deployment configuration files for each environment
- Listens for repository dispatch events from source repositories
- Pulls the latest images from your local registry
- Deploys or updates services using Docker Compose or Docker Swarm

For detailed examples of deployment files for each environment, see:

- [Docker Compose Examples](scripts/compose-examples/) - Single-server deployments
- [Docker Swarm Examples](scripts/swarm-examples/) - Multi-server deployments

These deployment files follow a consistent pattern but with environment-specific configurations:

1. **Development**: Basic configuration for testing and development
2. **Staging**: Production-like environment for final testing
3. **Production**: Optimized configuration with resource limits and high availability

Each environment uses separate networks and volume names to ensure complete isolation.

### 4. GitHub Actions Workflow for Deployment

The deployment repository should have a workflow that listens for repository dispatch events.

Key points about the deployment workflow:

1. **Event Types**: The workflow listens for specific event types that include service name and environment
2. **Environment Extraction**: The workflow extracts the environment and service name from the event payload
3. **Deployment Method**: The workflow chooses between Docker Compose or Docker Swarm based on configuration
4. **Service Deployment**: The command deploys the specific service to the target environment

This approach allows you to maintain separate configuration files for each environment while using a single workflow to handle deployments.

## Setting Up the Multi-Repository CI/CD Pipeline

### Prerequisites

1. Multiple GitHub repositories for your services
2. A homelab server (or cluster) with Docker installed
3. A local Docker registry in your homelab
4. A dedicated deployment repository
5. GitHub Personal Access Token (PAT) with repository access

### Repository Structure

This CI/CD pipeline uses a multi-repository approach with clear separation of concerns:

1. **Source Repositories** - Where your application code lives

   - Each service (frontend, API, backend) has its own repository
   - Each repository contains its own Dockerfile and CI/CD workflow
   - The workflow builds and pushes Docker images, then triggers deployment

2. **Deployment Repository** - Where your deployment configuration lives
   - Contains deployment files for each environment
   - Contains a workflow that responds to deployment triggers
   - Handles the actual deployment to your homelab

### Service Types Explained

In our example, we use three common service types that represent a typical modern web application architecture:

1. **Frontend** - User interface layer

   - A web application that users interact with directly (React, Vue, Angular, etc.)
   - Runs in the user's browser and communicates with the API
   - Example: E-commerce website user interface with product listings, shopping cart, checkout pages
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

3. Add the GitHub Actions workflow file (`.github/workflows/build-push.yml`). Example workflow:

   ```yaml
   name: Build and Push to Homelab

   on:
     push:
       branches:
         - dev
         - staging
         - main
     workflow_dispatch:
       inputs:
         environment:
           description: "Environment to deploy to"
           required: true
           default: "dev"
           type: choice
           options:
             - dev
             - staging
             - prod

   env:
     REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
     PROJECT_NAME: my-project
     SERVICE_NAME: frontend # Change this for each service (frontend, api, backend)
     ENVIRONMENT: ${{ github.event.inputs.environment || (github.ref_name == 'main' && 'prod' || github.ref_name == 'staging' && 'staging' || 'dev') }}

   jobs:
     build-and-push:
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

         - name: Login to registry
           uses: docker/login-action@v3
           with:
             registry: ${{ env.REGISTRY_URL }}
             username: ${{ secrets.REGISTRY_USER }}
             password: ${{ secrets.REGISTRY_PASSWORD }}

         - name: Build and push image
           uses: docker/build-push-action@v5
           with:
             context: .
             file: Dockerfile
             push: true
             tags: |
               ${{ env.REGISTRY_URL }}/${{ env.PROJECT_NAME }}/${{ env.SERVICE_NAME }}:${{ env.ENVIRONMENT }}
             labels: |
               org.opencontainers.image.source=${{ github.repository }}
             cache-from: type=gha
             cache-to: type=gha,mode=max

         - name: Trigger Deployment
           uses: peter-evans/repository-dispatch@v2
           with:
             token: ${{ secrets.CICD_PAT }}
             repository: your-username/deployment-repo
             event-type: deploy-${{ env.ENVIRONMENT }}
             client-payload: '{"service": "${{ env.SERVICE_NAME }}", "environment": "${{ env.ENVIRONMENT }}", "action": "deploy"}'

         - name: Cleanup Tailscale
           if: always()
           run: |
             sudo tailscale logout
   ```

4. Configure repository secrets:
   - `REGISTRY_URL`: Your local Docker registry URL
   - `REGISTRY_USER`: Docker registry username
   - `REGISTRY_PASSWORD`: Docker registry password
   - `TAILSCALE_AUTHKEY`: Tailscale auth key
   - `CICD_PAT`: GitHub Personal Access Token for triggering the deployment repository

### Step 2: Set Up the Deployment Repository

1. Create a new GitHub repository for deployments (e.g., `homelab-deployment`)

2. Set up the repository structure:

   ```
   deployment-repo/
   ├── .github/
   │   └── workflows/
   │       └── deploy.yml
   ├── compose/
   │   ├── docker-compose.dev.yml
   │   ├── docker-compose.staging.yml
   │   └── docker-compose.prod.yml
   ├── swarm/
   │   ├── docker-stack.dev.yml
   │   ├── docker-stack.staging.yml
   │   └── docker-stack.prod.yml
   ├── scripts/
   │   ├── deploy-compose.sh
   │   └── deploy-swarm.sh
   └── README.md
   ```

3. Create deployment files for each environment:

   **Docker Compose Example (compose/docker-compose.dev.yml):**

   ```yaml
   version: "3.8"

   services:
     frontend-dev:
       image: ${REGISTRY}/my-project/frontend:dev
       ports:
         - "3000:80"
       environment:
         - NODE_ENV=development
         - API_URL=http://api-dev:3001
       depends_on:
         - api-dev
       restart: unless-stopped

     api-dev:
       image: ${REGISTRY}/my-project/api:dev
       ports:
         - "3001:3000"
       environment:
         - NODE_ENV=development
         - DATABASE_URL=postgresql://user:password@db-dev:5432/myapp_dev
       depends_on:
         - db-dev
       restart: unless-stopped

     backend-dev:
       image: ${REGISTRY}/my-project/backend:dev
       ports:
         - "4000:4000"
       environment:
         - ENV=development
         - DATABASE_URL=postgresql://user:password@db-dev:5432/myapp_dev
       depends_on:
         - db-dev
       restart: unless-stopped

     db-dev:
       image: postgres:13
       environment:
         - POSTGRES_DB=myapp_dev
         - POSTGRES_USER=user
         - POSTGRES_PASSWORD=password
       volumes:
         - postgres_dev_data:/var/lib/postgresql/data
       restart: unless-stopped

   volumes:
     postgres_dev_data:
   ```

   **Docker Swarm Example (swarm/docker-stack.prod.yml):**

   ```yaml
   version: "3.8"

   services:
     frontend-prod:
       image: ${REGISTRY}/my-project/frontend:prod
       ports:
         - "80:80"
       environment:
         - NODE_ENV=production
         - API_URL=http://api-prod:3000
       networks:
         - app-network
       deploy:
         replicas: 2
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
           order: start-first
         resources:
           limits:
             cpus: "0.5"
             memory: 512M
           reservations:
             cpus: "0.25"
             memory: 256M

     api-prod:
       image: ${REGISTRY}/my-project/api:prod
       ports:
         - "3000:3000"
       environment:
         - NODE_ENV=production
         - DATABASE_URL=postgresql://user:password@db-prod:5432/myapp_prod
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
             memory: 1G
           reservations:
             cpus: "0.5"
             memory: 512M

     backend-prod:
       image: ${REGISTRY}/my-project/backend:prod
       environment:
         - ENV=production
         - DATABASE_URL=postgresql://user:password@db-prod:5432/myapp_prod
       networks:
         - app-network
       deploy:
         replicas: 2
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
             memory: 1G
           reservations:
             cpus: "0.5"
             memory: 512M

     db-prod:
       image: postgres:13
       environment:
         - POSTGRES_DB=myapp_prod
         - POSTGRES_USER=user
         - POSTGRES_PASSWORD=password
       volumes:
         - postgres_prod_data:/var/lib/postgresql/data
       networks:
         - app-network
       deploy:
         replicas: 1
         placement:
           constraints:
             - node.labels.environment == production
         restart_policy:
           condition: on-failure
         resources:
           limits:
             memory: 1G
           reservations:
             memory: 512M

   networks:
     app-network:
       driver: overlay
       attachable: true

   volumes:
     postgres_prod_data:
       driver: local
   ```

4. Add the deployment workflow file (`.github/workflows/deploy.yml`):

   ```yaml
   name: Deploy to Homelab

   on:
     repository_dispatch:
       types:
         - deploy-dev
         - deploy-staging
         - deploy-prod

   env:
     ENVIRONMENT: ${{ github.event.client_payload.environment }}
     SERVICE: ${{ github.event.client_payload.service }}
     DEPLOYMENT_TYPE: ${{ vars.DEPLOYMENT_TYPE || 'compose' }} # Set as repository variable

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4

         - name: Setup Tailscale
           uses: tailscale/github-action@v3
           with:
             authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
             tags: tag:ci

         - name: Deploy to homelab
           uses: appleboy/ssh-action@v1.0.0
           with:
             host: ${{ secrets.HOMELAB_HOST }}
             username: ${{ secrets.HOMELAB_USER }}
             key: ${{ secrets.HOMELAB_SSH_KEY }}
             script: |
               cd /opt/homelab

               # Set environment variables
               export REGISTRY=${{ secrets.REGISTRY_URL }}
               export POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }}

               # Sync deployment files
               if [ "${{ env.DEPLOYMENT_TYPE }}" == "swarm" ]; then
                 curl -o deployments/docker-stack.${{ env.ENVIRONMENT }}.yml \
                   https://raw.githubusercontent.com/${{ github.repository }}/main/swarm/docker-stack.${{ env.ENVIRONMENT }}.yml
                 ./scripts/deploy-swarm.sh ${{ env.ENVIRONMENT }}
               else
                 curl -o deployments/docker-compose.${{ env.ENVIRONMENT }}.yml \
                   https://raw.githubusercontent.com/${{ github.repository }}/main/compose/docker-compose.${{ env.ENVIRONMENT }}.yml
                 ./scripts/deploy-compose.sh ${{ env.ENVIRONMENT }}
               fi

               echo "✅ Deployment to ${{ env.ENVIRONMENT }} completed successfully"

         - name: Cleanup Tailscale
           if: always()
           run: |
             sudo tailscale logout
   ```

5. Configure repository secrets:

   - `HOMELAB_HOST`: Your homelab server's Tailscale IP address
   - `HOMELAB_USER`: Your CI/CD user (e.g., cicd)
   - `HOMELAB_SSH_KEY`: SSH private key for the CI/CD user
   - `TAILSCALE_AUTHKEY`: Tailscale auth key
   - `REGISTRY_URL`: Your local Docker registry URL
   - `POSTGRES_PASSWORD`: Database password

6. Set up the deployment directories and files on your homelab server:

   ```bash
   # On your homelab server
   sudo mkdir -p /opt/homelab/{deployments,scripts}
   sudo chown -R cicd:cicd /opt/homelab

   # The GitHub Actions workflow will sync the deployment files here
   ```

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
- name: Trigger Deployment
  uses: peter-evans/repository-dispatch@v2
  with:
    token: ${{ secrets.CICD_PAT }}
    repository: your-username/deployment-repo
    event-type: deploy-${{ env.ENVIRONMENT }}
    client-payload: '{"service": "${{ env.SERVICE_NAME }}", "environment": "${{ env.ENVIRONMENT }}", "action": "deploy"}'
```

And the deployment repository workflow should listen for these events:

```yaml
on:
  repository_dispatch:
    types:
      - deploy-dev
      - deploy-staging
      - deploy-prod
```

## Benefits of This Multi-Repository Approach

1. **Separation of Concerns**: Code development is separate from deployment configuration
2. **Environment Isolation**: Different environments are clearly separated
3. **Deployment Flexibility**: Choose between Docker Compose or Docker Swarm based on your needs
4. **Secure Image Transfer**: Images are transferred securely via Tailscale
5. **Private Registry**: Images are stored privately in your homelab
6. **Centralized Deployment**: All deployments are managed from a single repository
7. **Automated Workflow**: Code changes automatically trigger builds and deployments
8. **Scalable Architecture**: Easy to add new services or environments

## Choosing Between Docker Compose and Docker Swarm

### Use Docker Compose When:

- You have a single server or simple setup
- You're learning CI/CD concepts
- You need quick deployments and easy debugging
- Your application doesn't require high availability

### Use Docker Swarm When:

- You have multiple servers available
- You need high availability and zero-downtime deployments
- You want built-in load balancing and scaling
- You're running production workloads

## Security Considerations

1. Use Tailscale for secure connections between GitHub Actions and your homelab
2. Store sensitive information in GitHub Secrets
3. Use a dedicated CI/CD user with limited permissions
4. Regularly rotate SSH keys and access tokens
5. Use environment-specific configurations
6. Implement proper network segmentation
7. Use Docker secrets for sensitive data in production

## Troubleshooting

### Common Issues

1. **Repository Dispatch Not Triggered**:

   - Verify the PAT token has correct permissions
   - Check the event type matches in both repositories
   - Ensure the repository path is correct

2. **Image Pull Failures**:

   - Verify registry authentication is working
   - Check network connectivity between servers
   - Ensure the image exists in the registry

3. **Deployment Failures**:

   - Check the deployment logs in GitHub Actions
   - Verify environment variables are set correctly
   - Ensure the deployment files are syntactically correct

4. **Service Communication Issues**:
   - Verify network configuration in Docker Compose/Swarm
   - Check service names match in configuration
   - Ensure ports are correctly exposed

This multi-repository approach provides a robust, scalable CI/CD pipeline that can handle complex deployments while maintaining clean separation between code and infrastructure concerns. It's suitable for both beginners starting with Docker Compose and advanced users leveraging Docker Swarm for production workloads.
