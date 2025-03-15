# Troubleshooting Guide

This section provides solutions for common issues you might encounter with your homelab setup.

## Table of Contents

- [Proxmox Issues](#proxmox-issues)
- [TrueNAS SCALE Issues](#truenas-scale-issues)
- [Network Issues](#network-issues)
- [Docker and Container Issues](#docker-and-container-issues)
- [Storage Issues](#storage-issues)
- [Performance Issues](#performance-issues)
- [Remote Access Issues](#remote-access-issues)

## Proxmox Issues

### VM Won't Start

**Symptoms**: Virtual machine fails to start, showing error in the Proxmox web interface.

**Possible Solutions**:

1. **Check resource availability**:
   ```bash
   # Check CPU and memory usage
   top
   # Check disk space
   df -h
   ```

2. **Check VM configuration**:
   - Ensure VM has adequate resources allocated
   - Verify storage is accessible
   - Check VM configuration file for errors

3. **Check Proxmox logs**:
   ```bash
   # Check system logs
   tail -n 100 /var/log/syslog
   # Check Proxmox logs
   tail -n 100 /var/log/pve/tasks/
   ```

### Proxmox Web Interface Not Accessible

**Symptoms**: Cannot access the Proxmox web interface at https://your-proxmox-ip:8006

**Possible Solutions**:

1. **Check if Proxmox services are running**:
   ```bash
   systemctl status pveproxy
   systemctl status pvedaemon
   ```

2. **Restart Proxmox services**:
   ```bash
   systemctl restart pveproxy
   systemctl restart pvedaemon
   ```

3. **Check firewall settings**:
   ```bash
   # Check if port 8006 is open
   iptables -L -n | grep 8006
   ```

### Subscription Notice Won't Go Away

**Symptoms**: Subscription notice appears even after applying the fix.

**Solution**:

1. Re-apply the fix and ensure correct file path:
   ```bash
   sed -i.backup "s/data.status !== 'Active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
   ```

2. Clear browser cache or try a different browser

3. Restart the pveproxy service:
   ```bash
   systemctl restart pveproxy.service
   ```

## TrueNAS SCALE Issues

### Web Interface Not Accessible

**Symptoms**: Cannot access the TrueNAS web interface.

**Possible Solutions**:

1. **Check if TrueNAS VM is running** in Proxmox

2. **Check network configuration** of the TrueNAS VM

3. **Access the TrueNAS console** through Proxmox and check network settings:
   ```bash
   # Show network interfaces
   ifconfig
   # Check if web service is running
   service nginx status
   ```

### ZFS Pool Issues

**Symptoms**: ZFS pool shows degraded status or errors.

**Possible Solutions**:

1. **Check pool status**:
   ```bash
   zpool status
   ```

2. **Scrub the pool** to check for and fix errors:
   ```bash
   zpool scrub tank
   ```

3. **Replace failed disk** if necessary:
   ```bash
   zpool replace tank old_disk new_disk
   ```

### Dataset Permission Issues

**Symptoms**: Cannot access or write to datasets.

**Possible Solutions**:

1. **Check permissions**:
   ```bash
   ls -la /mnt/tank/dataset
   ```

2. **Fix permissions** through the TrueNAS web interface:
   - Go to Storage > Pools
   - Click the three dots next to the dataset
   - Select "Edit Permissions"
   - Set appropriate owner, group, and permissions

## Network Issues

### Cannot Connect to VMs

**Symptoms**: Unable to reach VMs on the network.

**Possible Solutions**:

1. **Check VM network configuration**:
   - Verify IP address configuration
   - Ensure VM is connected to the correct bridge

2. **Check Proxmox network configuration**:
   ```bash
   # Show network interfaces
   ip addr show
   # Check bridge status
   brctl show
   ```

3. **Check firewall settings** on both Proxmox and the VM

### Tailscale Connectivity Issues

**Symptoms**: Cannot connect to homelab through Tailscale.

**Possible Solutions**:

1. **Check Tailscale status**:
   ```bash
   tailscale status
   ```

2. **Restart Tailscale**:
   ```bash
   sudo systemctl restart tailscaled
   ```

3. **Re-authenticate Tailscale**:
   ```bash
   sudo tailscale up
   ```

4. **Check subnet routing** if trying to access other devices:
   ```bash
   sudo tailscale up --advertise-routes=192.168.1.0/24
   ```

### Cloudflare Tunnel Issues

**Symptoms**: Cannot access services through Cloudflare Tunnel.

**Possible Solutions**:

1. **Check tunnel status**:
   ```bash
   cloudflared tunnel info <tunnel-id>
   ```

2. **Check tunnel logs**:
   ```bash
   journalctl -u cloudflared
   ```

3. **Restart the tunnel**:
   ```bash
   sudo systemctl restart cloudflared
   ```

4. **Verify DNS configuration** in Cloudflare dashboard

## Docker and Container Issues

### Container Won't Start

**Symptoms**: Docker container fails to start.

**Possible Solutions**:

1. **Check container logs**:
   ```bash
   docker logs container_name
   ```

2. **Check container configuration**:
   ```bash
   docker inspect container_name
   ```

3. **Check resource availability**:
   ```bash
   # Check disk space
   df -h
   # Check memory usage
   free -h
   ```

### Docker Compose Issues

**Symptoms**: `docker-compose up` fails.

**Possible Solutions**:

1. **Check Docker Compose file syntax**:
   ```bash
   docker-compose config
   ```

2. **Check environment variables**:
   ```bash
   cat .env
   ```

3. **Check network configuration**:
   ```bash
   docker network ls
   ```

### Container Networking Issues

**Symptoms**: Containers cannot communicate with each other or the host.

**Possible Solutions**:

1. **Check Docker networks**:
   ```bash
   docker network ls
   docker network inspect bridge
   ```

2. **Check container network settings**:
   ```bash
   docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name
   ```

3. **Restart Docker service**:
   ```bash
   sudo systemctl restart docker
   ```

## Storage Issues

### Disk Space Running Out

**Symptoms**: System reports low disk space.

**Possible Solutions**:

1. **Check disk usage**:
   ```bash
   df -h
   ```

2. **Find large files**:
   ```bash
   find / -type f -size +100M -exec ls -lh {} \; | sort -k5 -rh
   ```

3. **Clean up Docker**:
   ```bash
   # Remove unused containers
   docker container prune
   # Remove unused images
   docker image prune
   # Remove unused volumes
   docker volume prune
   ```

### Slow Disk Performance

**Symptoms**: File operations are slow, high disk I/O wait times.

**Possible Solutions**:

1. **Check disk I/O**:
   ```bash
   iostat -x 1
   ```

2. **Check for disk errors**:
   ```bash
   smartctl -a /dev/sda
   ```

3. **Optimize ZFS settings** if using ZFS:
   ```bash
   # Check current ARC size
   arc_summary
   # Adjust ARC size if needed
   echo "options zfs zfs_arc_max=4294967296" > /etc/modprobe.d/zfs.conf
   ```

## Performance Issues

### High CPU Usage

**Symptoms**: System is slow, high CPU usage reported.

**Possible Solutions**:

1. **Identify CPU-intensive processes**:
   ```bash
   top
   ```

2. **Check for runaway processes**:
   ```bash
   ps aux | sort -rk 3,3 | head -n 10
   ```

3. **Adjust VM resource allocation** in Proxmox if necessary

### Memory Issues

**Symptoms**: System is slow, high memory usage or swapping.

**Possible Solutions**:

1. **Check memory usage**:
   ```bash
   free -h
   ```

2. **Identify memory-intensive processes**:
   ```bash
   ps aux | sort -rk 4,4 | head -n 10
   ```

3. **Adjust VM memory allocation** in Proxmox if necessary

### Network Performance Issues

**Symptoms**: Slow network transfers, high latency.

**Possible Solutions**:

1. **Check network usage**:
   ```bash
   iftop
   ```

2. **Test network speed**:
   ```bash
   iperf3 -c server_ip
   ```

3. **Check for network bottlenecks**:
   - Network interface configuration
   - Switch/router performance
   - Cable quality

## Remote Access Issues

### VPN Connection Problems

**Symptoms**: Cannot establish VPN connection to homelab.

**Possible Solutions**:

1. **Check VPN service status**:
   ```bash
   # For WireGuard
   sudo wg show
   # For OpenVPN
   sudo systemctl status openvpn
   ```

2. **Check firewall settings**:
   ```bash
   sudo iptables -L
   ```

3. **Check VPN logs**:
   ```bash
   # For WireGuard
   journalctl -u wg-quick@wg0
   # For OpenVPN
   journalctl -u openvpn
   ```

### SSH Connection Issues

**Symptoms**: Cannot SSH into servers.

**Possible Solutions**:

1. **Check SSH service status**:
   ```bash
   sudo systemctl status sshd
   ```

2. **Check SSH configuration**:
   ```bash
   cat /etc/ssh/sshd_config
   ```

3. **Check firewall settings**:
   ```bash
   sudo iptables -L | grep 22
   ```

4. **Try verbose connection** for more information:
   ```bash
   ssh -vvv user@server
   ```

## Diagnostic Tools

### System Information

```bash
# General system information
uname -a
# Hardware information
lshw
# PCI devices
lspci
# USB devices
lsusb
# Block devices
lsblk
```

### Network Diagnostics

```bash
# Network interfaces
ip addr
# Routing table
ip route
# DNS resolution
dig example.com
# Network connectivity
ping -c 4 8.8.8.8
# Trace route
traceroute example.com
```

### Log Analysis

```bash
# System logs
journalctl -xe
# Specific service logs
journalctl -u service_name
# Kernel messages
dmesg
```

## Next Steps

- [Set up monitoring to detect issues early](../maintenance/monitoring.md)
- [Implement backup solutions for disaster recovery](../maintenance/backup.md)
- [Configure alerting for critical issues](../maintenance/monitoring.md#alerting) 