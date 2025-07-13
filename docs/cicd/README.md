# CI/CD Pipeline Documentation for Homelab

This directory contains comprehensive documentation for setting up and managing CI/CD pipelines in your homelab environment. Whether you're new to homelabbing or looking to implement automated deployments, this guide will help you build a robust CI/CD pipeline.

## Overview

Continuous Integration and Continuous Deployment (CI/CD) is a critical part of modern development workflows. This documentation covers how to set up CI/CD pipelines for your homelab projects, leveraging GitHub Actions, local Docker registry, and secure deployment practices.

## What You'll Learn

- How to set up automated deployments to your homelab
- Secure remote access using Tailscale
- Local Docker registry for private image storage
- Multiple deployment strategies (Docker Compose and Docker Swarm)
- Multi-repository CI/CD workflows
- Environment separation (development, staging, production)

## Architecture Options

This documentation covers two main deployment architectures:

### Option 1: Single Server with Docker Compose

- **Best for**: Beginners, small projects, learning
- **Requirements**: 1 server, Docker, basic setup
- **Benefits**: Simple, easy to understand, quick to set up

### Option 2: Multi-Server with Docker Swarm

- **Best for**: Advanced users, production workloads, high availability
- **Requirements**: 4+ servers, Docker Swarm cluster
- **Benefits**: High availability, rolling updates, load balancing, scalability

## Contents

### Getting Started

- [Basic CI/CD Pipeline](basic-pipeline.md) - Simple single-server setup with Docker Compose
- [CI/CD Pipeline Setup](cicd-pipeline.md) - Complete setup guide for both architectures

### Advanced Topics

- [Multi-Repository CI/CD](multi-repo-cicd.md) - Managing multiple services and repositories
- [Docker Swarm Setup](docker-swarm-setup.md) - Setting up a Docker Swarm cluster
- [Docker Swarm Commands](docker-swarm-commands.md) - Quick reference for Swarm operations

## Example Scripts

The [scripts](scripts/) directory contains example files for implementing CI/CD pipelines:

- **Source Repository Examples** - [scripts/source-repos/](scripts/source-repos/)

  - Frontend, API, and Backend repository examples
  - Dockerfiles and GitHub Actions workflows for each service type
  - Examples for both Docker Compose and Docker Swarm deployments

- **Deployment Examples** - [scripts/deployments/](scripts/deployments/)
  - Docker Compose files for single-server deployments
  - Docker Swarm stack files for multi-server deployments
  - Environment-specific configurations

These examples demonstrate best practices for organizing your CI/CD pipeline code and infrastructure configuration.

## Key Features

### CI/CD Pipeline Features

- **Automated Builds**: Automatically build Docker images from your code repositories
- **Secure Deployments**: Use Tailscale for secure connections to your homelab
- **Environment Separation**: Maintain distinct development, staging, and production environments
- **Local Registry Integration**: Push and pull images from your private Docker registry
- **Cross-Repository Orchestration**: Coordinate deployments across multiple repositories

### Deployment Options

#### Docker Compose Benefits

- **Simplicity**: Easy to understand and manage
- **Quick Setup**: Get started in minutes
- **Perfect for Learning**: Great for understanding CI/CD concepts
- **Resource Efficient**: Works well on a single server

#### Docker Swarm Benefits

- **High Availability**: Automatic service failover and recovery
- **Rolling Updates**: Zero-downtime deployments with rollback capabilities
- **Service Scaling**: Easy horizontal scaling of services across nodes
- **Load Balancing**: Built-in load balancing and service discovery
- **Multi-Environment Support**: Deploy to different environments using node labels

## Getting Started

### For Beginners

If you're new to CI/CD in a homelab environment, start with the [Basic CI/CD Pipeline](basic-pipeline.md) guide. This will walk you through setting up a simple single-server deployment using Docker Compose.

### For Advanced Users

If you're ready for a production-ready setup with high availability, check out the [Docker Swarm Setup](docker-swarm-setup.md) guide and then the [Multi-Repository CI/CD](multi-repo-cicd.md) documentation.

## Prerequisites

### Basic Setup (Docker Compose)

- 1 homelab server running Docker
- GitHub repositories for your projects
- Basic understanding of Docker and containerization

### Advanced Setup (Docker Swarm)

- 4+ homelab servers (or VMs) for manager and worker nodes
- Docker Swarm cluster initialized
- A local Docker registry running on the management server
- Tailscale set up on all servers for secure communication
- Understanding of Docker Swarm concepts

## Common Use Cases

### Personal Projects

- Portfolio websites
- API services
- Development environments
- Testing deployments

### Small Teams

- Collaborative development
- Staging environments
- Code review workflows
- Automated testing

### Production Workloads

- High-availability services
- Load-balanced applications
- Blue-green deployments
- Disaster recovery

## Next Steps

1. **Choose Your Architecture**: Decide between Docker Compose (simple) or Docker Swarm (advanced)
2. **Set Up Prerequisites**: Install Docker, Tailscale, and prepare your servers
3. **Follow the Guides**: Start with the appropriate setup guide based on your choice
4. **Implement Examples**: Use the example scripts to set up your first CI/CD pipeline
5. **Customize**: Adapt the examples to your specific needs and projects

This documentation is designed to grow with you - start simple and expand as your needs become more complex.
