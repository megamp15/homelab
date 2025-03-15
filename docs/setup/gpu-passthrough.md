# GPU Passthrough to Virtual Machines

This guide covers how to pass through an NVIDIA GPU from your Proxmox host to a virtual machine, enabling GPU acceleration for applications like machine learning, video transcoding, or gaming.

## Prerequisites

- Proxmox VE installed and configured
- A CPU that supports IOMMU (Intel VT-d or AMD-Vi)
- An NVIDIA GPU (this guide uses a GeForce GTX 1660 Ti as an example)
- A VM already created in Proxmox

## Steps to Pass Through NVIDIA GPU to a VM

### 1. Blacklist Nouveau in Proxmox

The open-source Nouveau driver can interfere with NVIDIA's proprietary driver. Blacklist it on your Proxmox host:

```bash
sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

Add the following lines to the file:

```bash
blacklist nouveau
options nouveau modeset=0
```

Update the initramfs to apply the changes:

```bash
sudo update-initramfs -u
```

### 2. Enable VFIO and IOMMU in Proxmox

First, check if IOMMU is already enabled:

```bash
dmesg | grep -i iommu
```

If you don't see IOMMU enabled, modify the GRUB configuration:

```bash
sudo nano /etc/default/grub
```

For Intel CPUs, add or modify:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```

For AMD CPUs, add or modify:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"
```

Update GRUB and reboot:

```bash
sudo update-grub
sudo reboot
```

### 3. Find the NVIDIA GPU IDs

After rebooting, identify your GPU's PCI device IDs:

```bash
lspci -nn | grep -i nvidia
```

Example output:
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 Ti] [10de:2182] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation TU116 High Definition Audio Controller [10de:1aeb] (rev a1)
01:00.2 USB controller [0c03]: NVIDIA Corporation TU116 USB 3.1 Host Controller [10de:1aec] (rev a1)
01:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU116 USB Type-C UCSI Controller [10de:1aed] (rev a1)
```

Note the device IDs in square brackets (e.g., `10de:2182`, `10de:1aeb`, etc.).

### 4. Bind the GPU to VFIO

Create a VFIO configuration file to bind the GPU to the VFIO driver:

```bash
echo "options vfio-pci ids=10de:2182,10de:1aeb,10de:1aec,10de:1aed disable_vga=1" | sudo tee /etc/modprobe.d/vfio.conf
```

Replace the IDs with your actual device IDs from the previous step.

Load the VFIO modules at boot:

```bash
echo "vfio" | sudo tee -a /etc/modules
echo "vfio_iommu_type1" | sudo tee -a /etc/modules
echo "vfio_pci" | sudo tee -a /etc/modules
echo "vfio_virqfd" | sudo tee -a /etc/modules
```

Update initramfs and reboot:

```bash
sudo update-initramfs -u
sudo reboot
```

### 5. Add the GPU to the VM

In the Proxmox web interface:

1. Select your VM
2. Go to Hardware tab
3. Click "Add" > "PCI Device"
4. Select your GPU from the dropdown (e.g., `01:00.0`)
5. Check "All Functions" to include all related devices (audio, USB, etc.)
6. Click "Add"

### 6. Configure VM for GPU Passthrough

Edit the VM's configuration file for optimal GPU passthrough:

```bash
sudo nano /etc/pve/qemu-server/YOUR_VM_ID.conf
```

Add these lines at the end:

```
machine: q35
cpu: host,hidden=1,flags=+pcid
args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'
```

These settings help avoid the "Error 43" issue with NVIDIA GPUs in VMs.

### 7. Start the VM and Verify GPU Passthrough

Start your VM and verify that the GPU is properly passed through:

```bash
lspci -nnk | grep -A3 -i nvidia
```

### 8. Install NVIDIA Drivers in the VM

Install the appropriate NVIDIA drivers in your VM:

For Ubuntu/Debian:
```bash
sudo apt-get update
sudo apt-get install --reinstall nvidia-driver-535
```

For other distributions, follow their specific instructions for NVIDIA driver installation.

Verify the driver installation:
```bash
nvidia-smi
```

This should display information about your GPU, including the driver version and GPU utilization.

### 9. Install NVIDIA Container Toolkit (for Docker)

If you plan to use the GPU with Docker containers, install the NVIDIA Container Toolkit:

```bash
# Add the NVIDIA repository
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

# Install the NVIDIA Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-docker2
sudo systemctl restart docker
```

### 10. Verify Docker GPU Access

Test that Docker can access the GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi
```

If successful, this command should display the same GPU information as the `nvidia-smi` command.

## Troubleshooting

### Error 43 in Windows VMs

If you're passing through to a Windows VM and encounter "Error 43" in Device Manager:

1. Ensure you've added the `hidden=1` flag to the CPU line in the VM config
2. Add the `hv_vendor_id=NV43FIX` argument to hide the VM from NVIDIA's hypervisor detection
3. Make sure you're using the Q35 machine type

### GPU Not Showing in VM

If the GPU doesn't appear in the VM:

1. Verify that IOMMU is enabled: `dmesg | grep -i iommu`
2. Check that the GPU is bound to VFIO: `lspci -nnk | grep -A3 vfio`
3. Ensure all GPU functions (audio, USB, etc.) are passed through
4. Check the VM's logs for any errors: `qm showcmd YOUR_VM_ID`

### Performance Issues

If you experience performance issues:

1. Ensure CPU pinning is configured for optimal performance
2. Consider isolating CPU cores for the VM
3. Use the host CPU model in the VM configuration

## Next Steps

- [Configure GPU for transcoding in Plex](../services/plex-gpu-transcoding.md)
- [Set up machine learning containers with GPU acceleration](../services/ml-containers.md)
- [Optimize VM performance](vm-optimization.md) 