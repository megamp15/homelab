# My Homelab (2025)

![License](https://img.shields.io/github/license/megamp15/homelab)
![Last Commit](https://img.shields.io/github/last-commit/megamp15/homelab)

> A comprehensive documentation of my homelab setup, configurations, and learnings from building a home server infrastructure in 2025.

## What is a Homelab?

A homelab is a personal laboratory at home where you can experiment with new technologies, self-host applications, and learn about system administration, networking, and more in a controlled environment.

## Hardware Specifications

| Component | Details |
|-----------|---------|
| CPU | Intel i7-6700K (4 cores, 8 threads) |
| RAM | 16GB DDR4 |
| GPU | NVIDIA GeForce GTX 1660 Ti |
| Storage | 1× 2TB SSD, 2× 18TB HDDs |
| Network | 1Gbps Ethernet |
| Original Purpose | Gaming PC (2016) |

## Core Technologies

| Technology | Purpose |
|------------|---------|
| [Proxmox VE](https://www.proxmox.com/en/proxmox-ve) | Virtualization platform |
| [TrueNAS SCALE](https://www.truenas.com/truenas-scale/) | Network Attached Storage |
| [Ubuntu Server](https://ubuntu.com/server) | VM operating system |
| [Docker](https://www.docker.com/) | Container platform |
| [Tailscale](https://tailscale.com/) | Private mesh network & secure remote access |
| [Cloudflare Tunnels](https://www.cloudflare.com/products/tunnel/) | Secure public-facing services without port forwarding |

## Documentation

- [Setup Guide](docs/setup/README.md) - Initial installation and configuration
- [Network Configuration](docs/network/README.md) - Network setup and VPN configuration
- [Storage Management](docs/storage/README.md) - TrueNAS setup and data management
- [Services](docs/services/README.md) - Self-hosted applications and services
  - [Docker Installation](docs/services/docker-install.md) - Setting up Docker Engine
  - [Docker Registry](docs/services/docker-registry.md) - Private container registry
- [Maintenance](docs/maintenance/README.md) - Backup strategies and system maintenance
  - [CI/CD Pipeline](docs/maintenance/cicd-pipeline.md) - Automated deployment workflow
- [Troubleshooting](docs/troubleshooting/README.md) - Common issues and solutions

## Project Structure

```
homelab/
├── docs/
│   ├── setup/
│   ├── network/
│   ├── storage/
│   ├── services/
│   │   ├── docker-install.md
│   │   └── docker-registry.md
│   ├── maintenance/
│   │   └── cicd-pipeline.md
│   └── troubleshooting/
└── README.md
```

## Features

- **Virtualization**: Running multiple VMs on Proxmox for service isolation
- **Containerization**: Docker for efficient application deployment
- **Private Registry**: Local Docker registry for storing custom images
- **Automated Deployments**: CI/CD pipeline for streamlined updates
- **Secure Remote Access**: VPN and tunneling for accessing services from anywhere
- **Persistent Storage**: NAS integration for reliable data storage
- **Backup Solutions**: Comprehensive backup strategy for data protection

## Goals

- Document the entire homelab setup process for future reference
- Create a reproducible configuration for disaster recovery
- Share knowledge and experiences with the homelab community
- Continuously improve and expand the homelab capabilities

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgements

- [khuedoan/homelab](https://github.com/khuedoan/homelab) - Inspiration for this repository structure
- [r/homelab](https://www.reddit.com/r/homelab/) - The homelab community for ideas and support