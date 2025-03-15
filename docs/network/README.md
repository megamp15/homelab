# Network Configuration

This section covers the network setup for my homelab, including local networking and remote access solutions.

## Table of Contents

- [Overview](#overview)
- [Network Architecture](#network-architecture)
- [Remote Access Solutions](#remote-access-solutions)
  - [Tailscale Configuration](#tailscale-configuration)
  - [Cloudflare Tunnels Configuration](#cloudflare-tunnels-configuration)
- [DNS Configuration](#dns-configuration)
- [Security Considerations](#security-considerations)

## Overview

A well-designed network is the foundation of a reliable and secure homelab. This guide covers my network configuration with a focus on:

- Providing secure remote access to my homelab
- Avoiding port forwarding for enhanced security
- Accessing my homelab from anywhere in the world
- Selectively exposing services to the internet with custom domains

## Network Architecture

My homelab network is designed with security and accessibility in mind:

```
┌─────────────────────────────────────────────────────────────┐
│                      Internet                               │
│                          │                                  │
│                          ▼                                  │
│                    ┌──────────┐                             │
│                    │  Router  │                             │
│                    │  (No port│                             │
│                    │ forwarding)                            │
│                    └──────────┘                             │
│                          │                                  │
│                          ▼                                  │
│                    ┌──────────┐                             │
│                    │  Switch  │                             │
│                    └──────────┘                             │
│                          │                                  │
│                          ▼                                  │
│                    ┌───────────┐                            │
│                    │ Proxmox   │                            │
│                    │   Host    │◄────┐                      │
│                    │(Tailscale)│     │                      │
│                    └───────────┘     │                      │
│                          │           │                      │
│         ┌────────────────┼───────────┘                      │
│         │                │                                  │
│         ▼                ▼                                  │
│  ┌─────────────┐  ┌─────────────┐                           │
│  │  TrueNAS    │  │Ubuntu Server│                           │
│  │   SCALE     │  │     VM      │                           │
│  │ (SMB/NFS)   │  │ (Tailscale) │                           │
│  └─────────────┘  └─────────────┘                           │
│                          │                                  │
│                          ▼                                  │
│                   ┌──────────────┐                          │
│                   │  Cloudflare  │                          │
│                   │   Tunnels    │                          │
│                   │              │                          │
│                   └──────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

## Remote Access Solutions

I use two complementary solutions for remote access to my homelab:

1. **Tailscale**: Primary solution for secure access to my entire homelab network
2. **Cloudflare Tunnels**: For exposing specific services to the internet with custom domains

### Tailscale Configuration

Tailscale provides a zero-config VPN solution based on WireGuard that allows me to securely access my homelab from anywhere.

#### Why Tailscale?

- **Zero configuration**: Works across NATs and firewalls without port forwarding
- **End-to-end encryption**: All traffic is encrypted using WireGuard
- **MagicDNS**: Automatic DNS resolution for devices on the tailnet
- **Multi-platform**: Clients for Windows, macOS, Linux, iOS, Android
- **SSH access**: Direct SSH access to all systems on the tailnet

#### Installation Steps

1. Create a Tailscale account at [tailscale.com](https://tailscale.com/)

2. Install Tailscale on Linux systems (Proxmox host, Ubuntu Server VM, etc.):

```bash
# Simple one-line installation
curl -fsSL https://tailscale.com/install.sh | sh

# Start and authenticate
sudo tailscale up

# Check status
tailscale status
```

#### Using Tailscale

Once installed, I can:

- SSH directly to any machine on my tailnet: `ssh user@hostname`
- Access web interfaces using MagicDNS: `https://hostname`
- Transfer files securely between devices
- Access my homelab from anywhere in the world

### Cloudflare Tunnels Configuration

While Tailscale is perfect for personal access, Cloudflare Tunnels allows me to selectively expose specific services to the internet with custom subdomains on my domain (pmahir.info).

#### Why Cloudflare Tunnels?

- **No port forwarding**: Enhances security by eliminating the need to open ports on my router
- **DDoS protection**: Cloudflare's infrastructure protects against attacks
- **Custom domains**: Use subdomains of my domain (pmahir.info)
- **Automatic SSL**: Free SSL certificates for all services
- **Access control**: Can restrict access based on various factors

#### Setup Process

1. **Set Up Cloudflare for Your Domain**
   - Sign up at [Cloudflare](https://www.cloudflare.com/)
   - Add your domain (pmahir.info) to Cloudflare
   - Change your domain's nameservers on Namecheap to Cloudflare's nameservers
   - Ensure your domain is in proxied mode (orange cloud) for DNS

2. **Install Cloudflare Tunnel Client**

```bash
# Install dependencies
apt install -y curl gpg

# Add Cloudflare's GPG key
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null

# Add Cloudflare's repository
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main' | tee /etc/apt/sources.list.d/cloudflared.list

# Update package lists and install cloudflared
apt update && apt install -y cloudflared
```

3. **Authenticate with Cloudflare**

```bash
cloudflared tunnel login
```

This will open a browser to authorize with Cloudflare.

4. **Create a New Tunnel**

```bash
cloudflared tunnel create homelab-tunnel
```

This will generate a tunnel ID and create a credentials file.

5. **Configure the Tunnel**

Create or edit the configuration file:

```bash
sudo mkdir -p /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```

Add the following configuration:

```yaml
tunnel: <your-tunnel-id>
credentials-file: /root/.cloudflared/<your-tunnel-id>.json

ingress:
  # Portainer
  - hostname: portainer.pmahir.info
    service: http://192.168.1.100:9000
  
  # Other services as needed
  
  # Catch-all rule
  - service: http_status:404
```

Replace `<your-tunnel-id>` with your actual tunnel ID.

6. **Install and Start the Service**

```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

7. **Update Cloudflare DNS**

The tunnel automatically creates DNS records, but you can also manually set them up:

- Go to Cloudflare Dashboard → DNS
- Add CNAME records:
  - `portainer` → Points to `<your-tunnel-id>.cfargotunnel.com` (Proxied)

8. **Restart the Service After Config Changes**

If you make changes to the configuration:

```bash
sudo systemctl restart cloudflared.service
sudo systemctl status cloudflared.service
```

9. **Upgrade Cloudflared When Needed**

```bash
apt update && apt upgrade -y cloudflared
```

#### Using Cloudflare Tunnels

With Cloudflare Tunnels configured, I can:

- Access my Portainer instance at `https://portainer.pmahir.info`
- Share specific services with others who don't have Tailscale
- Maintain security without exposing my home IP or opening ports

## DNS Configuration

### Local DNS Resolution

For local DNS resolution, I rely on:

- **Router DNS**: Basic hostname resolution for local devices
- **Tailscale MagicDNS**: Automatic DNS resolution for devices on the tailnet

### External DNS Resolution

For external DNS resolution:

- **Cloudflare DNS**: Manages DNS records for my pmahir.info domain
- **Cloudflare Tunnels**: Automatically creates DNS records for tunneled services

## Security Considerations

My network security approach includes:

- **No port forwarding**: Neither Tailscale nor Cloudflare Tunnels require opening ports on my router
- **End-to-end encryption**: All remote access traffic is encrypted
- **Access control**: Tailscale limits access to authorized devices only
- **Regular updates**: Keeping all networking components updated
- **Monitoring**: Watching for unusual access patterns

## Next Steps

- [Configure backup solutions](../maintenance/backup.md)
- [Set up monitoring](../maintenance/monitoring.md)
- [Implement additional security measures](../maintenance/security.md) 