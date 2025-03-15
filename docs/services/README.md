# Self-Hosted Services

This section covers the services and applications currently hosted in my homelab environment.

## Table of Contents

- [Overview](#overview)
- [Current Service Architecture](#current-service-architecture)
- [Storage Services](#storage-services)
- [Container Management](#container-management)
- [Remote Access Solutions](#remote-access-solutions)
- [Planned Future Services](#planned-future-services)

## Overview

Self-hosting services in my homelab provides several benefits:
- Complete control over my data
- No subscription fees for many services
- Customization options not available in commercial solutions
- Learning opportunities for system administration and DevOps practices

## Current Service Architecture

My homelab services are currently organized across a few systems:

```
┌─────────────────────────────────────────────────────────────┐
│                      Proxmox Host                           │
│                      (Tailscale Installed)                  │
│                                                             │
│  ┌──────────────────┐       ┌───────────────────────────┐   │
│  │                  │       │                           │   │
│  │   TrueNAS SCALE  │       │     Ubuntu Server VMs     │   │
│  │  - SMB Shares    │       │     (Tailscale Installed) │   │
│  │  - NFS Shares    │       │                           │   │
│  │                  │       │  Docker Containers:       │   │
│  │                  │       │  - Portainer              │   │
│  │                  │       │                           │   │
│  └──────────────────┘       └───────────────────────────┘   │
│                                                             │
│  ┌─────────────────┐                                        │
│  │ Cloudflare      │                                        │
│  │ Tunnel          │                                        │
│  │                 │                                        │
│  │ Provides secure │                                        │
│  │ external access │                                        │
│  │ to select       │                                        │
│  │ services        │                                        │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

## Storage Services

### TrueNAS SCALE File Sharing

TrueNAS SCALE provides robust file sharing capabilities:

- **SMB Shares**: Windows-compatible file sharing
  - Used for: Documents, media, backups
  - Accessible from: Windows, macOS, Linux clients

- **NFS Shares**: Unix/Linux file sharing
  - Used for: VM storage, container volumes
  - Accessible from: Linux systems, Proxmox, Docker

## Container Management

### Docker with Portainer

I'm using Docker with Portainer for container management:

- **Docker**: Running containerized applications
  - [Docker Installation Guide](docker-install.md)
  - Installed on: Ubuntu Server VM
  - Benefits: Isolation, easy deployment, resource efficiency

- **Portainer**: Web UI for Docker management
  - Features:
    - Container deployment and management
    - Volume and network management
    - Resource monitoring
    - Simple web interface

### Docker Registry

I've set up a local Docker registry to store and manage Docker images:

- [Docker Registry Setup Guide](docker-registry.md)
- Benefits:
  - Store private images without uploading to public registries
  - Faster image pulls for local deployments
  - No rate limits or bandwidth constraints
  - Complete control over image storage

## CI/CD Pipeline

I've implemented a CI/CD pipeline for automated deployments:

- [CI/CD Pipeline Setup Guide](../maintenance/cicd-pipeline.md)
- Benefits:
  - Automated deployments from Git to homelab
  - Separation of development and production environments
  - Secure deployment process
  - Consistent deployment workflow

## Remote Access Solutions

I use two complementary solutions for remote access to my homelab:

### Tailscale

Tailscale provides secure access to my entire homelab network from anywhere:

- **Installation**: Installed directly on:
  - Proxmox host
  - Ubuntu Server VM
  - Other VMs and LXCs as needed

- **Key Benefits**:
  - Zero-config VPN based on WireGuard
  - Secure SSH access to all systems
  - Works across NATs and firewalls
  - No port forwarding required
  - End-to-end encryption
  - MagicDNS for easy hostname resolution

- **Use Case**:
  - Primary method for accessing and managing all homelab systems remotely
  - Secure SSH access from anywhere in the world
  - Access to internal web interfaces and services
  - Secure file transfers between devices on the tailnet

### Cloudflare Tunnels

Cloudflare Tunnels provides secure public access to specific services:

- **Purpose**: Expose select services to the internet with custom subdomains on pmahir.info

- **Key Benefits**:
  - No port forwarding required
  - DDoS protection
  - Automatic SSL certificates
  - Custom domain names (*.pmahir.info)
  - Enhanced security compared to direct port forwarding

- **Use Case**:
  - Selectively exposing specific services to the internet
  - Sharing services with others who don't have Tailscale
  - Public-facing services that need a domain name

### Tailscale vs. Cloudflare Tunnels

While both provide remote access, they serve different purposes:

| Feature | Tailscale | Cloudflare Tunnels |
|---------|-----------|-------------------|
| Primary Use | Private access to entire network | Public access to specific services |
| Access Control | Limited to your devices/users | Can be public or restricted |
| Setup Complexity | Very simple | Moderate |
| Custom Domains | Uses MagicDNS | Uses your Cloudflare domains |
| Best For | Administration, personal access | Sharing services with others |

## Planned Future Services

Services I plan to add to my homelab in the future:

- **Monitoring**: Prometheus and Grafana for system monitoring
- **Password Management**: Bitwarden for secure password storage
- **Document Management**: Paperless-ngx for document organization
- **Git Server**: Gitea for code repository hosting
- **Media Server**: Plex or Jellyfin for media streaming

## Deployment Methods

### Docker Compose

I deploy Docker services using Docker Compose for easy management:

```yaml
# Example docker-compose.yml for Portainer
version: '3'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./portainer-data:/data
    ports:
      - 9000:9000
```

## Next Steps

- [Set up monitoring with Prometheus and Grafana](../maintenance/monitoring.md)
- [Configure backup solutions](../maintenance/backup.md)
- [Implement security best practices](../maintenance/security.md) 