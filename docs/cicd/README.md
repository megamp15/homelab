# CI/CD Pipeline Documentation

This directory contains comprehensive documentation for setting up and managing CI/CD pipelines in your homelab environment.

## Overview

Continuous Integration and Continuous Deployment (CI/CD) is a critical part of modern development workflows. This documentation covers how to set up robust CI/CD pipelines for your homelab projects, leveraging GitHub Actions, local Docker registry, and secure deployment practices.

## Contents

- [Basic CI/CD Pipeline](basic-pipeline.md) - Simple automated deployment workflow
- [Multi-Repository CI/CD](multi-repo-cicd.md) - Advanced setup with multiple repositories
- [GitHub Actions Configuration](github-actions.md) - Detailed GitHub Actions setup
- [Security Best Practices](security.md) - Securing your CI/CD pipeline

## Example Scripts

The [scripts](scripts/) directory contains example files for implementing CI/CD pipelines:

- **Source Repository Examples** - [scripts/source-repos/](scripts/source-repos/)
  - Frontend, API, and Backend repository examples
  - Dockerfiles and GitHub Actions workflows for each service type

- **Deployment Repository Example** - [scripts/deployment-repo/](scripts/deployment-repo/)
  - Docker Compose files for different environments
  - Deployment workflow for handling repository dispatch events

These examples demonstrate the separation of concerns between source repositories (where code lives) and the deployment repository (where infrastructure configuration lives).

## Key Features

- **Automated Builds**: Automatically build Docker images from your code repositories
- **Secure Deployments**: Use Tailscale for secure connections to your homelab
- **Environment Separation**: Maintain distinct staging and production environments
- **Local Registry Integration**: Push and pull images from your private Docker registry
- **Cross-Repository Orchestration**: Coordinate deployments across multiple repositories

## Getting Started

If you're new to CI/CD in a homelab environment, start with the [Basic CI/CD Pipeline](basic-pipeline.md) guide. For more complex setups involving multiple services and repositories, refer to the [Multi-Repository CI/CD](multi-repo-cicd.md) documentation.

## Prerequisites

- GitHub repositories for your projects
- A homelab server running Docker
- A local Docker registry
- Tailscale set up on your homelab server
- Basic understanding of Docker and containerization 