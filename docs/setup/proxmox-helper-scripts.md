# Proxmox Helper-Scripts Guide

This guide explains how to use the community-maintained Proxmox VE Helper-Scripts to quickly deploy LXC containers in your Proxmox environment.

## What are Proxmox Helper-Scripts?

[Proxmox Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/) are a collection of bash scripts that simplify the creation and configuration of LXC containers in Proxmox VE. These scripts automate many common tasks, allowing you to deploy ready-to-use containers with minimal effort.

## Benefits

- **Quick Deployment**: Deploy containers in seconds with pre-configured applications
- **Consistency**: Ensure all containers are created with the same baseline configuration
- **Simplicity**: No need to remember complex LXC creation commands
- **Variety**: Scripts for many popular applications and Linux distributions

## Prerequisites

- Proxmox VE 6.x or newer
- Root access to your Proxmox host (via SSH or console)
- Internet connection on your Proxmox host

## Installation

There's no installation required. The scripts are designed to be run directly from the web.

## Usage

### Basic Syntax

```bash
bash -c "$(wget -qLO - https://community-scripts.github.io/ProxmoxVE/install/<script-name>.sh)"
```

Replace `<script-name>` with the name of the specific script you want to run.

### Example: Ubuntu 22.04 LXC

To create a basic Ubuntu 22.04 LXC container:

```bash
bash -c "$(wget -qLO - https://community-scripts.github.io/ProxmoxVE/install/ubuntu-22-04.sh)"
```

### Interactive Process

When you run a script, it will:

1. Ask for a container ID (or suggest one)
2. Ask for a hostname
3. Ask for a password
4. Ask for storage location
5. Ask for memory allocation
6. Ask for CPU cores
7. Ask for disk size
8. Ask for network configuration
9. Create and configure the container

For a complete list of available scripts, visit the [Proxmox VE Helper-Scripts website](https://community-scripts.github.io/ProxmoxVE/).

## Customizing Containers After Creation

After creating a container with a helper script, you can further customize it:

1. Start the container from the Proxmox web UI or with:
   ```bash
   pct start <container-id>
   ```

2. Access the container's shell:
   ```bash
   pct enter <container-id>
   ```

3. Make your desired changes (install packages, configure services, etc.)

## Tips for Using Helper Scripts

1. **Review the script before running**: You can examine any script by viewing it in your browser first
2. **Use unique container IDs**: Keep track of your container IDs to avoid conflicts
3. **Set appropriate resources**: Allocate CPU, RAM, and disk space based on the container's purpose
4. **Use descriptive hostnames**: Name containers based on their function for easier management
5. **Back up important containers**: Use Proxmox's backup features for containers with critical data

## Troubleshooting

If you encounter issues with the helper scripts:

1. **Check network connectivity**: Ensure your Proxmox host has internet access
2. **Verify Proxmox version**: Some scripts may require specific Proxmox versions
3. **Check storage space**: Ensure you have enough free space on your storage
4. **Review logs**: Check the console output for error messages
5. **Container fails to start**: Check the container's configuration and logs

## Security Considerations

- Always change default passwords immediately after container creation
- Update containers regularly with `apt update && apt upgrade` or equivalent
- Consider using private networks for containers that don't need external access
- Review the permissions and access controls for each container

## Additional Resources

- [Official Proxmox LXC Documentation](https://pve.proxmox.com/wiki/Linux_Container)
- [Proxmox VE Helper-Scripts GitHub Repository](https://github.com/community-scripts/ProxmoxVE)
- [LXC Project Website](https://linuxcontainers.org/) 