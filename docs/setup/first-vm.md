# Creating Your First Virtual Machine in Proxmox

This guide walks through the process of creating and configuring your first virtual machine (VM) in Proxmox VE.

## Prerequisites

- Proxmox VE installed and configured
- ISO image of your preferred operating system (e.g., Ubuntu Server)
- Network connection to download ISO (if not already available)

## Uploading an ISO Image

1. In the Proxmox web interface, select your node in the server view
2. Go to the "local" storage (or other storage configured for ISO images)
3. Select the "Content" tab
4. Click "Upload"
5. Click "Select File" and choose your ISO file
6. Click "Upload"

Alternatively, you can download an ISO directly from the web:

1. Select "ISO Images" under the local storage
2. Click "Download from URL"
3. Enter the URL of the ISO (e.g., `https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso`)
4. Click "Download"

## Creating a New VM

1. Click "Create VM" at the top right of the Proxmox web interface
2. General:
   - Node: your Proxmox node
   - VM ID: accept default or choose a number
   - Name: give your VM a descriptive name (e.g., `ubuntu-server`)
   - Click "Next"
3. OS:
   - Select the ISO image you uploaded
   - Type: Linux
   - Version: 6.x - 2.6 Kernel (for Ubuntu Server)
   - Click "Next"
4. System:
   - Graphics card: Default
   - SCSI Controller: VirtIO SCSI
   - Qemu Agent: Check this box
   - Click "Next"
5. Disks:
   - Storage: local-lvm (or your preferred storage)
   - Disk size: 32 GB (adjust as needed)
   - Format: QEMU image format
   - Cache: Default (No cache)
   - Click "Next"
6. CPU:
   - Cores: 2 (adjust based on your needs)
   - Type: host
   - Click "Next"
7. Memory:
   - Memory: 4096 MB (adjust based on your needs)
   - Minimum memory: leave unchecked
   - Click "Next"
8. Network:
   - Bridge: vmbr0
   - Model: VirtIO
   - Click "Next"
9. Confirm:
   - Review your settings
   - Check "Start after created" if you want to start the VM immediately
   - Click "Finish"

## Installing the Operating System

1. Once the VM is created and started, click on it in the server view
2. Select "Console" to access the VM's console
3. Follow the installation instructions for your chosen operating system
4. For Ubuntu Server:
   - Select language and keyboard layout
   - Configure network (DHCP is usually fine for initial setup)
   - Configure storage (use the entire disk for simplicity)
   - Set up user account and password
   - Install OpenSSH server if needed
   - Complete the installation and reboot

## Removing the Installation ISO

After the installation completes, you need to remove the ISO file from the virtual CD drive before rebooting. You have two options:

### Option 1: Edit the CD/DVD Drive (Recommended)

1. Shut down the VM if it's running:
   - From the Proxmox web interface, select the VM
   - Click "Shutdown" in the top menu
   - Wait for the VM to shut down completely

2. Edit the CD/DVD Drive:
   - Select the VM in the server view
   - Go to "Hardware"
   - Select the CD/DVD Drive (usually "IDE0")
   - Click "Edit"
   - Select "Do not use any media"
   - Click "OK"

3. Start the VM:
   - Click "Start" in the top menu
   - The VM will now boot from the installed operating system instead of the ISO

### Option 2: Remove the CD/DVD Drive Completely

1. Shut down the VM if it's running:
   - From the Proxmox web interface, select the VM
   - Click "Shutdown" in the top menu
   - Wait for the VM to shut down completely

2. Remove the CD/DVD Drive:
   - Select the VM in the server view
   - Go to "Hardware"
   - Select the CD/DVD Drive (usually "IDE0")
   - Click "Remove"
   - Confirm the removal

3. Start the VM:
   - Click "Start" in the top menu
   - The VM will now boot from the installed operating system

If you forget to remove or edit the ISO, the VM might boot back into the installation media instead of your newly installed system. If this happens, just shut down the VM and follow one of the options above.

## Post-Installation Configuration

### Installing Qemu Guest Agent

For better integration with Proxmox, install the QEMU guest agent:

```bash
# For Ubuntu/Debian
sudo apt update
sudo apt install qemu-guest-agent
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

### Network Configuration

For server VMs, a consistent IP address is important. Instead of configuring a static IP within the VM, I use my ASUS RT-AC86U router's "Manually Assigned IP" feature:

1. Find the MAC address of your VM:
   - In Proxmox, select the VM
   - Go to Hardware > Network Device
   - Note the MAC address
   - Or, from within the VM: `ip link show`

2. Configure the router:
   - Log in to your ASUS router admin panel (typically http://192.168.1.1)
   - Go to LAN > DHCP Server
   - Scroll down to "Manually Assigned IP around the DHCP list"
   - Click "+"
   - Enter the MAC address of your VM
   - Assign your desired IP address
   - Enter a name for the device (e.g., "ubuntu-server")
   - Click "Apply"

3. Restart networking on the VM or reboot it:
   ```bash
   sudo systemctl restart systemd-networkd
   # or
   sudo reboot
   ```

This approach has several advantages:
- Keeps all IP management centralized in the router
- Allows the VM to use DHCP (simpler configuration)
- Makes it easy to change IPs without reconfiguring VMs
- Provides a clear overview of all assigned IPs in your network

## VM Templates

Once you have a base VM configured, you can convert it to a template for easy deployment of multiple similar VMs:

1. Shut down the VM
2. Right-click on the VM in the server view
3. Select "Convert to Template"
4. To create a new VM from this template:
   - Right-click on the template
   - Select "Clone"
   - Choose between linked clone (faster, space-efficient) or full clone (independent)
   - Give the new VM a name and ID
   - Click "Clone"

## Next Steps

- [Set up TrueNAS SCALE VM](../storage/truenas-setup.md)
- [Configure Ubuntu Server for hosting services](../services/README.md)
- [Set up backup solutions](../maintenance/backup.md) 