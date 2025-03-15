# Homelab Setup Guide

This section covers the initial setup and configuration of the homelab, focusing on the installation of Proxmox VE as the hypervisor platform.

## Table of Contents

- [Hardware Preparation](#hardware-preparation)
- [Proxmox VE Installation](#proxmox-ve-installation)
- [Post-Installation Configuration](#post-installation-configuration)
- [Network Setup](#network-setup)
- [Storage Configuration](#storage-configuration)

## Hardware Preparation

Before installing any software, ensure your hardware is properly set up:

1. Clean the computer case and components of dust
2. Ensure all components are properly seated and connected
3. Connect the system to a UPS if available
4. Connect to your network via Ethernet
5. Prepare a USB drive (at least 8GB) for Proxmox installation

## Proxmox VE Installation

### Prerequisites

- Download the latest Proxmox VE ISO from the [official website](https://www.proxmox.com/en/downloads)
- Create a bootable USB using tools like [Rufus](https://rufus.ie/) (Windows) or [Etcher](https://www.balena.io/etcher/) (cross-platform)

### Installation Steps

1. Boot from the USB drive
2. Select "Install Proxmox VE"
3. Accept the EULA
4. Select the target disk for installation (2TB SSD)
5. Configure location and timezone
6. Set up the root password and email
7. Configure network settings:
   - Hostname: `proxmox.local`
   - IP Address: `192.168.1.100` (or your preferred static IP)
   - Netmask: `255.255.255.0`
   - Gateway: `192.168.1.1` (your router's IP)
   - DNS Server: `192.168.1.1` (or your preferred DNS)
8. Review the settings and confirm installation
9. After installation, remove the USB drive and reboot

## Post-Installation Configuration

### Accessing the Web Interface

1. Open a web browser and navigate to `https://192.168.1.100:8006` (replace with your IP)
2. Log in with username `root` and the password you set during installation
3. Accept the self-signed certificate warning

### Removing Subscription Notice

To properly configure Proxmox VE without a subscription, follow these guides:

1. For updating repositories and disabling enterprise version:
   - [Guide: PVE Node Updates & Repository Configuration](https://www.youtube.com/watch?v=TWX3iWcka_0&t=333s)

2. For removing the subscription popup:
   - [Guide: Removing Subscription Popup](https://www.youtube.com/watch?v=mQGB4hL2_ZQ)

Alternatively, you can use the community post-installation script which automates these steps:
[Proxmox VE Post-Install Script](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install)

### Updating Proxmox

```bash
apt update
apt dist-upgrade -y
```

## Guides

- [Proxmox Installation](proxmox-installation.md)
- [First Virtual Machine](first-vm.md)
- [GPU Passthrough](gpu-passthrough.md)
- [LXC Containers](lxc-containers.md)

- [Set up TrueNAS SCALE](../storage/truenas-setup.md)
- [Configure network and VPN](../network/README.md) 