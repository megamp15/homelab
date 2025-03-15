# Installing Docker on Ubuntu

This guide covers how to install Docker Engine on Ubuntu for use in your homelab environment. Docker is an essential tool for running containerized applications.

## Overview

Docker Engine is available in two editions:
- **Docker Engine**: The open-source containerization technology
- **Docker Desktop**: A developer-focused application with a GUI (not covered in this guide)

This guide focuses on installing Docker Engine on Ubuntu using the official Docker repository.

## Prerequisites

- A server or VM running Ubuntu (20.04 LTS, 22.04 LTS, or newer)
- A user with sudo privileges
- Internet access for downloading packages

## Installation Methods

There are several ways to install Docker on Ubuntu:

1. **Using the official Docker repository** (recommended)
2. Installing from a package
3. Using the convenience script

This guide focuses on the recommended method: installing from the official Docker repository.

## Installing Docker Engine

### 1. Set Up the Docker Repository

First, update your package index and install packages to allow apt to use a repository over HTTPS:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```

Add Docker's official GPG key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Set up the repository:

```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2. Install Docker Engine

Update the package index:

```bash
sudo apt-get update
```

Install the latest version of Docker Engine, containerd, and Docker Compose:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. Verify the Installation

Verify that Docker Engine is installed correctly by running the hello-world image:

```bash
sudo docker run hello-world
```

This command downloads a test image and runs it in a container. When the container runs, it prints a confirmation message and exits.

## Post-Installation Steps

### Run Docker as a Non-Root User

To avoid typing `sudo` whenever you run the `docker` command, add your user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

Log out and log back in for this to take effect. You can also run the following command to activate the changes to groups:

```bash
newgrp docker
```

Verify that you can run docker commands without sudo:

```bash
docker run hello-world
```

### Configure Docker to Start on Boot

Most current Linux distributions use systemd to manage which services start when the system boots. On Debian and Ubuntu, Docker service is configured to start on boot by default. To manually enable this, use:

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Using Docker Compose

Docker Compose is included as the `docker compose` plugin with the Docker Engine installation. You can use it to define and run multi-container Docker applications:

```bash
# Create a docker-compose.yml file
nano docker-compose.yml

# Run containers defined in docker-compose.yml
docker compose up -d
```

## Uninstalling Docker Engine

If you need to uninstall Docker Engine:

```bash
# Uninstall Docker Engine, CLI, containerd, and Docker Compose packages
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

# Delete all images, containers, and volumes
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd

# Remove source list and keyrings
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.gpg
```

## Troubleshooting

### Permission Denied

If you encounter `permission denied` errors, make sure you've added your user to the docker group and logged out and back in.

### Failed to Connect to Docker Daemon

If you see an error like `Cannot connect to the Docker daemon`, make sure the Docker service is running:

```bash
sudo systemctl status docker
sudo systemctl start docker
```

## Next Steps

- [Set up Portainer for Docker management](portainer-setup.md)
- [Create a local Docker registry](docker-registry.md)
- [Learn Docker Compose basics](docker-compose-basics.md)

## References

- [Official Docker Engine Installation Documentation](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Post-installation Steps](https://docs.docker.com/engine/install/linux-postinstall/) 