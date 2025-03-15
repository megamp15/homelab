# Mounting TrueNAS Shares in Proxmox

This guide covers how to mount TrueNAS SCALE shares in Proxmox and virtual machines, using both SMB and NFS protocols.

## Table of Contents

- [Mounting SMB Shares in Proxmox](#mounting-smb-shares-in-proxmox)
- [Mounting SMB Shares in VMs](#mounting-smb-shares-in-vms)
- [Mounting NFS Shares](#mounting-nfs-shares)
- [Troubleshooting](#troubleshooting)

## Mounting SMB Shares in Proxmox

After setting up TrueNAS SCALE and creating an SMB share (e.g., `prox-share`), follow these steps to mount it in Proxmox:

### 1. Create a Mount Point

Create a directory where the share will be mounted:

```bash
sudo mkdir /mnt/prox-share
```

### 2. Install Required Packages

Install CIFS utilities for SMB/CIFS support:

```bash
sudo apt install cifs-utils -y
```

### 3. Create Credentials File

Create a secure credentials file to store your SMB username and password:

```bash
sudo nano /etc/smbcredentials
```

Add your TrueNAS credentials:

```
username=prox
password=password
```

Secure the credentials file:

```bash
sudo chmod 600 /etc/smbcredentials
```

### 4. Configure Automatic Mounting

Edit the `/etc/fstab` file to automatically mount the share at boot:

```bash
sudo nano /etc/fstab
```

Add the following line:

```bash
//192.168.50.191/prox-share /mnt/prox-share cifs credentials=/etc/smbcredentials,iocharset=utf8,rw,vers=3.1.1,uid=1000,gid=1000,file_mode=0777,dir_mode=0777,noauto,x-systemd.automount 0 0
```

Replace `192.168.50.191` with your TrueNAS IP address.

### 5. Apply Changes

Reload the systemd daemon to apply the changes:

```bash
sudo systemctl daemon-reload
```

### 6. Test the Mount

Test that the share mounts correctly:

```bash
sudo mount -a
```

Verify the mount:

```bash
df -h | grep prox-share
```

## Mounting SMB Shares in VMs

The process for mounting SMB shares in VMs is similar to Proxmox, with a slight modification to the fstab entry.

### 1. Create a Mount Point

```bash
sudo mkdir /mnt/prox-share-nas
```

### 2. Install Required Packages

```bash
sudo apt install cifs-utils -y
```

### 3. Create Credentials File

```bash
sudo nano /etc/smbcredentials
```

Add your TrueNAS credentials:

```
username=prox
password=password
```

Secure the credentials file:

```bash
sudo chmod 600 /etc/smbcredentials
```

### 4. Configure Automatic Mounting

Edit the `/etc/fstab` file:

```bash
sudo nano /etc/fstab
```

Add the following line:

```bash
//192.168.50.191/prox-share /mnt/prox-share-nas cifs credentials=/etc/smbcredentials,iocharset=utf8,rw,vers=3.1.1,uid=1000,gid=1000,file_mode=0777,dir_mode=0777,mfsymlinks,x-systemd.automount 0 0
```

Note the addition of `mfsymlinks` which allows symbolic links to work properly in VMs.

### 5. Apply Changes and Test

```bash
sudo systemctl daemon-reload
sudo mount -a
```

## Mounting NFS Shares

NFS is often preferred for Linux-to-Linux file sharing due to better performance and Unix permission handling.

### 1. Install NFS Client

```bash
sudo apt update && sudo apt install -y nfs-common
```

### 2. Create a Mount Point

```bash
sudo mkdir /mnt/prox-share-nfs
```

### 3. Configure Automatic Mounting

Edit the `/etc/fstab` file:

```bash
sudo nano /etc/fstab
```

Add the following line:

```bash
192.168.50.191:/mnt/Prox-Pool/prox-share /mnt/prox-share-nfs nfs rw,sync,hard,intr 0 0
```

Replace `192.168.50.191` with your TrueNAS IP address and `/mnt/Prox-Pool/prox-share` with the actual path to your NFS share.

### 4. Apply Changes and Test

```bash
sudo systemctl daemon-reload
sudo mount -a
```

Verify the mount:

```bash
df -h | grep prox-share-nfs
```

## Troubleshooting

### Permission Issues

If you encounter permission issues:

1. Check the UID and GID in the fstab entry match the user who needs access
2. Verify the share permissions in TrueNAS
3. Try adding `noperm` to the mount options for SMB shares

### Connection Issues

If the share won't mount:

1. Verify the TrueNAS IP address is correct
2. Ensure the share name is correct
3. Check that the share service is running on TrueNAS
4. Test connectivity: `ping 192.168.50.191`
5. For SMB, test connection: `smbclient -L //192.168.50.191 -U prox`
6. For NFS, test connection: `showmount -e 192.168.50.191`

### Mount at Boot Issues

If the share doesn't mount at boot:

1. Try changing `noauto,x-systemd.automount` to just `auto`
2. Add `_netdev` option to ensure the network is available before mounting
3. Check system logs: `journalctl -u systemd-automount` 