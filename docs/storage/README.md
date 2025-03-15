# Storage Management

This section covers the storage setup and management for the homelab, focusing on TrueNAS SCALE for network-attached storage (NAS) functionality.

## Table of Contents

- [Storage Architecture](#storage-architecture)
- [TrueNAS SCALE Setup](#truenas-scale-setup)
- [Disk Passthrough Configuration](#disk-passthrough-configuration)
- [Storage Pools and Datasets](#storage-pools-and-datasets)
- [Sharing Configuration](#sharing-configuration)
- [Backup Strategy](#backup-strategy)

## Storage Architecture

The homelab storage is designed with the following components:

```
┌─────────────────────────────────────────────────────────────┐
│                      Proxmox Host                           │
│                                                             │
│  ┌─────────────────┐       ┌───────────────────────────┐    │
│  │                 │       │                           │    │
│  │   System Drive  │       │     TrueNAS SCALE VM      │    │
│  │    (2TB SSD)    │       │                           │    │
│  │                 │       │   ┌─────────────────────┐ │    │
│  └─────────────────┘       │   │  ZFS Storage Pool   │ │    │
│                            │   │                     │ │    │
│                            │   │  ┌───────────────┐  │ │    │
│                            │   │  │   18TB HDD    │  │ │    │
│                            │   │  └───────────────┘  │ │    │
│                            │   │  ┌───────────────┐  │ │    │
│                            │   │  │   18TB HDD    │  │ │    │
│                            │   │  └───────────────┘  │ │    │
│                            │   └─────────────────────┘ │    │
│                            └───────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## TrueNAS SCALE Setup

TrueNAS SCALE is an open-source storage platform that combines the benefits of TrueNAS Core with the scalability of Linux containers and Kubernetes.

> **Note**: I followed the excellent guide from [Network Chuck on YouTube](https://www.youtube.com/watch?v=MkK-9_-2oko) for setting up TrueNAS SCALE. The steps below reflect my setup process, with some additional manual configurations I needed to make.

### Creating a TrueNAS SCALE VM in Proxmox

Follow these steps to set up a TrueNAS SCALE VM:

1. Download the latest TrueNAS SCALE ISO from the [official website](https://www.truenas.com/truenas-scale/)
2. Upload the ISO to Proxmox
3. Create a new VM with the following specifications:
   - General: Name `truenas`, VM ID of your choice
   - OS: Select the TrueNAS SCALE ISO
   - System: BIOS (SeaBIOS), SCSI Controller: VirtIO SCSI
   - Disks: Add a 50GB disk for the OS (local-lvm)
   - CPU: 4 cores, Type: host
   - Memory: 8GB minimum (16GB recommended)
   - Network: Bridge vmbr0, Model VirtIO
4. After creating the VM, you have two options for adding the HDDs:

   **Option 1: Using Virtual Disks (Simpler but less optimal)**
   - Select the VM > Hardware > Add > Hard Disk
   - Bus/Device: SCSI, Storage: local-lvm (or your preferred storage)
   - Disk size: 18TB (or the size of your physical disk)
   - Format: Raw disk image
   - Cache: Write back (or your preferred cache setting)
   - Repeat for the second HDD

   **Option 2: Using Disk Passthrough (Recommended for ZFS)**
   - Identify your physical disks (see Disk Passthrough Configuration section below)
   - Pass through the physical disks to the VM (see Adding Disks to TrueNAS VM below)
5. Start the VM and follow the TrueNAS SCALE installation wizard
6. After installation completes, remember to remove the ISO. You have two options:
   
   **Option 1: Edit the CD/DVD Drive**
   - Shut down the VM
   - Select the VM > Hardware > CD/DVD Drive
   - Click "Edit"
   - Select "Do not use any media"
   - Click "OK"
   - Start the VM again

   **Option 2: Remove the CD/DVD Drive Completely**
   - Shut down the VM
   - Select the VM > Hardware > CD/DVD Drive
   - Click "Remove"
   - Confirm the removal
   - Start the VM again

For detailed installation instructions, see [TrueNAS SCALE Installation](truenas-setup.md).

## Disk Passthrough Configuration

For optimal performance and to allow TrueNAS direct access to the physical disks, I used disk passthrough instead of virtual disks. This is crucial for ZFS to properly manage the disks and for features like SMART monitoring.

### Identifying Physical Disks

First, identify the physical disks in Proxmox:

```bash
# Install lshw if not already installed
apt install lshw

# List all disks and storage controllers
lshw -class disk -class storage
```

From the output, note the serial numbers and IDs of your drives. Here's what my output looked like:

```
disk:2
  description: ATA Disk
  product: ST18000NT001-3NF
  physical id: 2
  bus info: scsi@3:0.0.0
  logical name: /dev/sdc
  version: EN01
  serial: ZVTGH2PB
  size: 16TiB (18TB)
  capabilities: gpt-1.00 partitioned partitioned:gpt
  configuration: ansiversion=5 guid=dbb6b72b-b766-2c47-9db9-77cecdbe2c8f logicalsectorsize=512 sectorsize=4096

disk:3
  description: ATA Disk
  product: ST18000NT001-3NF
  physical id: 3
  bus info: scsi@4:0.0.0
  logical name: /dev/sdd
  version: EN01
  serial: ZVTGH1FM
  size: 16TiB (18TB)
  capabilities: gpt-1.00 partitioned partitioned:gpt
  configuration: ansiversion=5 guid=ab07dbf0-c666-204b-b976-857b1410ce4c logicalsectorsize=512 sectorsize=4096
```

You can also check the disk by-id paths:

```bash
ls -la /dev/disk/by-id/
```

From my system:
```
sdc    8:32   0 16.4T  0 disk   /dev/disk/by-id/ata-ST18000NT001-3NF101_ZVTGH2PB /dev/disk/by-id/wwn-0x5000c500e930a8e6
sdd    8:48   0 16.4T  0 disk   /dev/disk/by-id/wwn-0x5000c500e930c985 /dev/disk/by-id/ata-ST18000NT001-3NF101_ZVTGH1FM
```

### Adding Disks to TrueNAS VM

1. In the Proxmox web interface, select your TrueNAS VM
2. Go to Hardware > Add > Hard Disk
3. Select "Use physical disk"
4. Choose "Pass through disk" (not "Use volume")
5. Select the disk by its path, e.g., `/dev/disk/by-id/ata-ST18000NT001-3NF101_ZVTGH2PB`
6. Repeat for the second disk, e.g., `/dev/disk/by-id/ata-ST18000NT001-3NF101_ZVTGH1FM`

Using the disk by-id path ensures that the disk will always be identified correctly, even if the device order changes after a reboot.

## Storage Pools and Datasets

### Creating a ZFS Pool

1. In the TrueNAS web interface, go to Storage > Pools
2. Click "Add" to create a new pool
3. Select the two 18TB HDDs
4. Choose the desired RAID level:
   - Mirror (RAID1): Provides redundancy but only half the total capacity
   - RAIDZ-1 (RAID5): Requires at least 3 disks (not applicable with only 2 disks)
   - Stripe (RAID0): Uses full capacity but no redundancy (risky)
5. For two disks, Mirror (RAID1) is recommended for data safety
6. Name your pool (e.g., `tank`)
7. Click "Create"

### Creating Datasets

Datasets are used to organize and manage your data:

1. Go to Storage > Pools > [Your Pool Name]
2. Click "Add Dataset"
3. Name: `media`
4. Comments: "Media files including movies, TV shows, and music"
5. Sync: Standard
6. Compression: lz4 (good balance of performance and compression)
7. Enable Atime: Off (improves performance)
8. ZFS Deduplication: Off (high memory usage)
9. Case Sensitivity: Sensitive
10. Click "Save"

Repeat to create additional datasets as needed:

- `backups`: For system and data backups
- `documents`: For important documents
- `photos`: For photo collections
- `downloads`: For temporary downloads

## Sharing Configuration

### SMB (Windows) Shares

1. Go to Shares > Windows Shares (SMB)
2. Click "Add"
3. Path: Browse to your dataset (e.g., `/mnt/tank/media`)
4. Name: `media`
5. Description: "Media files share"
6. Access Control List: Select appropriate permissions
7. Enable ACL: On
8. Click "Save"

For connecting to the SMB share:
- Username: root (or your configured user)
- Password: your password
- Share path: `\\truenas-ip\share-name` (e.g., `\\192.168.1.10\prox-share`)

### NFS (Unix) Shares

1. Go to Shares > Unix Shares (NFS)
2. Click "Add"
3. Path: Browse to your dataset (e.g., `/mnt/tank/backups`)
4. Description: "Backups share"
5. Networks: Your local network (e.g., `192.168.1.0/24`)
6. Click "Save"

## Backup Strategy

### Snapshot Configuration

ZFS snapshots provide point-in-time copies of your data:

1. Go to Data Protection > Snapshots
2. Click "Add"
3. Dataset: Select your dataset
4. Naming Schema: `auto-%Y-%m-%d_%H-%M`
5. Schedule:
   - Hourly: Keep 24
   - Daily: Keep 7
   - Weekly: Keep 4
   - Monthly: Keep 12
6. Click "Save"

### Replication (Optional)

If you have an offsite backup location:

1. Go to Data Protection > Replication Tasks
2. Click "Add"
3. Source: Your local dataset
4. Destination: Remote dataset
5. Schedule: Daily or weekly
6. Click "Save"

### Cloud Backup (Optional)

For critical data, consider cloud backup solutions:

1. Go to System > Cloud Credentials
2. Add credentials for your cloud provider (e.g., AWS, Google Cloud)
3. Go to Tasks > Cloud Sync Tasks
4. Create a new task to sync important datasets to the cloud

## Performance Tuning

### ZFS Tuning

Optimize ZFS performance with these settings:

1. Go to System > Advanced
2. Set the following:
   - ZFS prefetch: Enabled
   - ZFS metadata cache max: 2GB (adjust based on available RAM)
   - ZFS arc max: 8GB (adjust based on available RAM)

### Network Tuning

Optimize network performance:

1. Go to Network > Global Configuration
2. Enable Jumbo Frames (MTU 9000) if your network supports it
3. Set appropriate TCP and UDP buffer sizes

## Next Steps

- [Configure services on TrueNAS SCALE](../services/truenas-services.md)
- [Mount TrueNAS shares in Proxmox and VMs](mounting-nas-proxmox.md)
- [Set up backup automation](../maintenance/backup.md)
- [Monitor storage health](../maintenance/monitoring.md) 