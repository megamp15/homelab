# CI/CD Scripts

This directory contains example scripts and configuration files for setting up a multi-repository CI/CD pipeline for your homelab.

## Directory Structure

```
scripts/
├── source-repos/                  # Example files for source repositories
│   ├── frontend/                  # Frontend application repository
│   │   ├── Dockerfile             # Docker build instructions for frontend
│   │   └── .github/workflows/
│   │       └── build-push.yml     # CI/CD workflow for frontend
│   ├── api/                       # API service repository
│   │   ├── Dockerfile             # Docker build instructions for API
│   │   └── .github/workflows/
│   │       └── build-push.yml     # CI/CD workflow for API
│   └── backend/                   # Backend service repository
│       ├── Dockerfile             # Docker build instructions for backend
│       └── .github/workflows/
│           └── build-push.yml     # CI/CD workflow for backend
├── deployment-repo/               # Central deployment repository
│   ├── docker-compose.dev.yml     # Development environment configuration
│   ├── docker-compose.staging.yml # Staging environment configuration
│   ├── docker-compose.prod.yml    # Production environment configuration
│   └── .github/workflows/
│       └── deploy.yml             # Deployment workflow
└── README.md                      # This file
```

## Repository Organization

This structure demonstrates the separation between:

1. **Source Repositories** - Where your application code lives
   - Each source repository contains its own Dockerfile and CI/CD workflow
   - The workflow builds and pushes Docker images, then triggers deployment

2. **Deployment Repository** - Where your deployment configuration lives
   - Contains Docker Compose files for each environment
   - Contains a workflow that responds to deployment triggers
   - Handles the actual deployment to your homelab

## Usage

These files are examples that you can use as a starting point for your own CI/CD pipeline. You'll need to customize them to fit your specific needs.

### Source Repository Files

For each of your source repositories (frontend, API, backend, etc.):

1. Copy the appropriate `Dockerfile` from the matching example in `source-repos/`
2. Create a `.github/workflows/` directory and copy the `build-push.yml` file
3. Customize the workflow to match your specific service requirements

### Deployment Repository Files

For your central deployment repository:

1. Copy the Docker Compose files from `deployment-repo/`
2. Create a `.github/workflows/` directory and copy the `deploy.yml` file
3. Customize the Docker Compose files for your specific services and environments

## Getting Started

1. Set up your source repositories with the appropriate Dockerfiles and workflows
2. Set up your deployment repository with Docker Compose files and the deployment workflow
3. Configure the secrets in your GitHub repositories as described in the main documentation
4. Customize the files to fit your specific needs

For more detailed instructions, see the [Multi-Repository CI/CD Pipeline](../multi-repo-cicd.md) documentation. 