# TrueNAS SCALE Installation and Setup

This guide provides detailed instructions for installing and configuring TrueNAS SCALE as a virtual machine in Proxmox.

## Prerequisites

- Proxmox VE installed and configured
- TrueNAS SCALE ISO downloaded from the [official website](https://www.truenas.com/truenas-scale/)
- Two 18TB HDDs available for storage

## VM Creation in Proxmox

### Step 1: Upload TrueNAS SCALE ISO

1. In the Proxmox web interface, select your node
2. Go to local storage (or other storage configured for ISO images)
3. Select the "Content" tab
4. Click "Upload"
5. Select the TrueNAS SCALE ISO file and upload it

### Step 2: Create the VM

1. Click "Create VM" at the top right
2. General:
   - Node: your Proxmox node
   - VM ID: accept default or choose a number
   - Name: `truenas`
   - Start at boot: Yes
   - Click "Next"
3. OS:
   - Use CD/DVD disc image file: Yes
   - Storage: local (or where you uploaded the ISO)
   - ISO image: Select the TrueNAS SCALE ISO
   - Type: Linux
   - Version: 5.x - 2.6 Kernel
   - Click "Next"
4. System:
   - Graphics card: Default
   - Machine: q35
   - BIOS: SeaBIOS
   - SCSI Controller: VirtIO SCSI
   - Qemu Agent: Checked
   - Click "Next"
5. Disks:
   - Storage: local-lvm (or your preferred storage)
   - Disk size: 50 GB (for TrueNAS OS)
   - Format: QEMU image format
   - Cache: Write back
   - Discard: Checked
   - Click "Next"
6. CPU:
   - Cores: 4 (adjust based on your needs)
   - Type: host
   - Click "Next"
7. Memory:
   - Memory: 16384 MB (16GB, adjust based on your needs, minimum 8GB)
   - Minimum memory: Unchecked
   - Click "Next"
8. Network:
   - Bridge: vmbr0
   - Model: VirtIO
   - Click "Next"
9. Confirm:
   - Review your settings
   - Click "Finish"

### Step 3: Add Storage Disks

1. Select the newly created TrueNAS VM
2. Click "Hardware"
3. Click "Add" > "Hard Disk"
4. Storage: local-lvm (or your preferred storage)
5. Disk size: 18 TB (or the size of your physical disk)
6. Format: Raw disk image
7. Cache: Write back
8. Click "Add"
9. Repeat steps 3-8 for the second 18TB HDD

### Step 4: Configure VM Options

1. Select the TrueNAS VM
2. Click "Options"
3. Double-click "Boot Order"
4. Ensure the boot order is:
   - CD-ROM
   - Disk 0 (the 50GB OS disk)
5. Click "OK"

## TrueNAS SCALE Installation

### Step 1: Start the Installation

1. Select the TrueNAS VM and click "Start"
2. Click "Console" to access the VM console
3. Wait for the TrueNAS SCALE boot menu to appear
4. Select "Install/Upgrade"

### Step 2: Installation Process

1. Select the destination disk (the 50GB disk, usually identified as `/dev/sda`)
2. Choose installation options:
   - Boot mode: BIOS
   - Pool type: Default
   - Swap size: 2GB (default)
3. Set the root password
4. Select "Install" and confirm
5. Wait for the installation to complete
6. When prompted, select "Reboot System"
7. Remove the installation media (in Proxmox, go to Hardware, select CD/DVD, click "Edit", and set "No media")

## Initial Configuration

### Step 1: Access the Web Interface

1. After the VM reboots, the console will display the TrueNAS IP address
2. Open a web browser and navigate to that IP address
3. Log in with:
   - Username: `root`
   - Password: the password you set during installation

### Step 2: Initial Setup Wizard

1. Complete the initial setup wizard:
   - Configure network settings (set a static IP if desired)
   - Set up directory services if needed
   - Configure storage (we'll do this in detail later)

### Step 3: Update the System

1. Go to System > Update
2. Click "Check for Updates"
3. If updates are available, click "Update" and follow the prompts

## Storage Configuration

### Step 1: Create a ZFS Pool

1. Go to Storage > Pools
2. Click "Add"
3. Select "Create new pool"
4. Name: `tank` (or your preferred name)
5. Select the two 18TB HDDs
6. Choose the RAID level:
   - For data redundancy: Mirror (RAID1)
   - For maximum capacity: Stripe (RAID0, but no redundancy)
7. Click "Create"

### Step 2: Create Datasets

1. Go to Storage > Pools > [Your Pool Name]
2. Click "Add Dataset"
3. Name: `media`
4. Comments: "Media files including movies, TV shows, and music"
5. Advanced Options:
   - Sync: Standard
   - Compression: lz4
   - Enable Atime: Off
   - ZFS Deduplication: Off
   - Case Sensitivity: Sensitive
6. Click "Save"

7. Create additional datasets as needed:
   - `backups`: For system and data backups
   - `documents`: For important documents
   - `photos`: For photo collections
   - `downloads`: For temporary downloads

## Sharing Configuration

### Step 1: Configure SMB Shares

1. Go to Shares > Windows Shares (SMB)
2. Click "Add"
3. Path: Browse to your dataset (e.g., `/mnt/tank/media`)
4. Name: `media`
5. Description: "Media files share"
6. Click "Advanced Options"
7. Enable ACL: On
8. Click "Save"

### Step 2: Configure NFS Shares (Optional)

1. Go to Shares > Unix Shares (NFS)
2. Click "Add"
3. Path: Browse to your dataset (e.g., `/mnt/tank/backups`)
4. Description: "Backups share"
5. Networks: Your local network (e.g., `192.168.1.0/24`)
6. Click "Save"

## User and Permission Setup

### Step 1: Create a User

1. Go to Credentials > Local Users
2. Click "Add"
3. Full Name: Your name
4. Username: Your preferred username
5. Password: Set a strong password
6. Primary Group: Create a new group (e.g., `homelab`)
7. Home Directory: `/mnt/tank/homes/username`
8. Click "Save"

### Step 2: Set Permissions

1. Go to Storage > Pools > [Your Pool Name]
2. Click the three dots next to a dataset
3. Select "Edit Permissions"
4. Owner: Select your user
5. Owner Group: Select your group
6. Access Mode: Set appropriate permissions (e.g., 755)
7. Click "Save"

## System Tuning

### Step 1: Configure Email Alerts

1. Go to System > Email
2. Configure your email settings for system alerts
3. Click "Send Test Email" to verify
4. Click "Save"

### Step 2: Configure SMART Tests

1. Go to Data Protection > SMART Tests
2. Click "Add"
3. Disks: Select all disks
4. Type: Long
5. Description: "Monthly SMART Test"
6. Schedule: Monthly
7. Click "Save"
8. Add another test with Type: Short and Schedule: Weekly

### Step 3: Configure Power Management

1. Go to System > Advanced
2. Scroll to Power Management
3. Configure power settings based on your needs
4. Click "Save"

## Applications (Optional)

TrueNAS SCALE includes a built-in application system based on Kubernetes:

1. Go to Apps
2. Click "Discover Apps"
3. Browse available applications
4. Install applications as needed (e.g., Plex Media Server, NextCloud)

## Next Steps

- [Configure backup solutions](../maintenance/backup.md)
- [Set up monitoring](../maintenance/monitoring.md)
- [Explore TrueNAS SCALE applications](../services/truenas-services.md) 