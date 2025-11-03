# Ubuntu 24.04 Cloud Image Template Setup for Proxmox

This guide walks through creating a VM template from Ubuntu 24.04 cloud image in Proxmox.

## Prerequisites

- Proxmox VE installed and running
- SSH access to Proxmox host
- Internet connection to download cloud image

## Step 1: Download Ubuntu 24.04 Cloud Image

```bash
cd ~
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

## Step 2: Create a New VM

```bash
# Create VM with ID 9100 (adjust as needed)
qm create 9100 --name ubuntu-cloud-template --memory 4096 --net0 virtio,bridge=vmbr0
```

**Parameters:**
- `9100` - VM ID (use a high number for templates)
- `--name` - Template name
- `--memory` - RAM in MB (4GB recommended, adjust as needed)
- `--net0` - Network interface (change `vmbr0` to your bridge name)

## Step 3: Import the Cloud Image

```bash
qm importdisk 9100 noble-server-cloudimg-amd64.img local
```

**Note:** Replace `local` with your storage name if different (e.g., `local-lvm`, `local-zfs`)

The output will show something like:
```
unused0: successfully imported disk 'local:9100/vm-9100-disk-0.raw'
```

## Step 4: Attach the Disk to the VM

```bash
# Set SCSI controller and attach the imported disk
qm set 9100 --scsihw virtio-scsi-pci --scsi0 local:9100/vm-9100-disk-0.raw
```

**Important:** Use the exact path format from the import output: `storage:VMID/disk-name`

## Step 5: Resize the Disk

The cloud image is only 2-3.5GB. Resize it now so all cloned VMs have adequate space:

```bash
# Add 30GB to the disk (adjust as needed)
qm resize 9100 scsi0 +30G
```

**Note:** Cloud-init will automatically expand the filesystem on first boot. By resizing the template disk now, all cloned VMs will start with this larger disk size.

## Step 6: Add Cloud-Init Drive

```bash
# Add cloud-init drive
qm set 9100 --ide2 local:cloudinit
```

## Step 7: Configure Boot Settings

```bash
# Set the boot disk
qm set 9100 --boot c --bootdisk scsi0

# Add serial console (optional but recommended for troubleshooting)
qm set 9100 --serial0 socket --vga serial0
```

## Step 8: Configure Cloud-Init Defaults (Optional)

```bash
# Set default user
qm set 9100 --ciuser ubuntu

# Set network to DHCP
qm set 9100 --ipconfig0 ip=dhcp

# Add your SSH public key
qm set 9100 --sshkeys ~/.ssh/id_rsa.pub
```

**Alternative SSH key methods:**
```bash
# From a file
qm set 9100 --sshkeys /path/to/your/key.pub

# Or inline (URL encode newlines as %0A)
qm set 9100 --sshkeys "ssh-rsa AAAAB3NzaC1yc2E... user@host"
```

## Step 9: Convert to Template

```bash
qm template 9100
```

**Warning:** Once converted to a template, the VM cannot be started directly. You must clone it first.

## Using the Template

### Clone the Template

```bash
# Full clone (recommended)
qm clone 9100 200 --name my-ubuntu-vm --full

# Linked clone (faster, but dependent on template)
qm clone 9100 200 --name my-ubuntu-vm
```

### Resize the Disk (Optional)

If you need MORE space than what's in the template (32GB from our setup):

```bash
qm resize 200 scsi0 +20G
```

**Note:** Since we resized the disk in the template (Step 5), all cloned VMs already start with 32GB. You only need to resize if you need more than that for specific VMs.

### Customize Cloud-Init per VM

**Set Static IP Address:**
```bash
# Set static IP with gateway
qm set 200 --ipconfig0 ip=192.168.1.100/24,gw=192.168.1.1

# Set DNS servers
qm set 200 --nameserver "8.8.8.8 8.8.4.4"

# Set search domain
qm set 200 --searchdomain example.local
```

**Set User Password:**
```bash
# Set password for SSH and console access
qm set 200 --cipassword 'YourSecurePassword'
```

**Complete Example with Static IP and Password:**
```bash
# Clone the template
qm clone 9100 200 --name web-server-01 --full

# Configure network and credentials
qm set 200 --ciuser ubuntu
qm set 200 --cipassword 'MyPassword123!'
qm set 200 --ipconfig0 ip=192.168.1.100/24,gw=192.168.1.1
qm set 200 --nameserver "192.168.1.1"
qm set 200 --searchdomain lab.local
qm set 200 --sshkeys ~/.ssh/id_rsa.pub

# Start the VM
qm start 200
```

**Multiple Network Interfaces:**
```bash
# First interface - static IP
qm set 200 --ipconfig0 ip=192.168.1.100/24,gw=192.168.1.1

# Second interface - DHCP
qm set 200 --ipconfig1 ip=dhcp
```

**Important Notes:**
- **Set these BEFORE starting the VM** - cloud-init only runs on first boot
- Password should be in single quotes to avoid shell interpretation issues
- SSH keys are recommended over passwords for SSH access
- The password is useful for console access via Proxmox web interface

**Verify Configuration Before Starting:**
```bash
# View user configuration (including password - it will be hashed)
qm cloudinit dump 200 user

# View network configuration
qm cloudinit dump 200 network
```

### Start the VM

```bash
qm start 200
```

## Cloud-Init Configuration

On first boot, cloud-init will:
- Set the hostname
- Create the user account
- Install SSH keys
- Configure networking
- Run any user-data scripts

You can view cloud-init logs inside the VM:
```bash
cloud-init status --wait
cloud-init status --long
```

## Tips

### Storage Options

- **local** - Directory-based storage on host
- **local-lvm** - LVM thin provisioning (recommended for production)
- **local-zfs** - ZFS storage (great for snapshots and clones)

### Template Best Practices

1. **Use high VM IDs for templates** (9000+) to keep them separate from regular VMs
2. **Resize the disk in the template** (as shown in Step 5) so all clones have adequate space
3. **Install common packages before templating** (optional):
   ```bash
   # Start the VM once before templating
   qm start 9100
   
   # SSH in and install packages
   apt update && apt install -y qemu-guest-agent
   
   # Shutdown
   shutdown -h now
   
   # Then template it
   qm template 9100
   ```
4. **Keep templates updated** - Create new template versions periodically with updated images

### Customizing During Clone

You can also customize at clone time:
```bash
qm clone 9100 200 --name my-vm --full --storage local-lvm --cores 4 --memory 8192
```

## Troubleshooting

### Can't SSH to new VM

1. Check if cloud-init finished:
   ```bash
   # Inside the VM console
   cloud-init status
   ```

2. Verify SSH key was set:
   ```bash
   qm cloudinit dump 200 user
   ```

3. Check if SSH password authentication is enabled (if using passwords):
   ```bash
   # Inside the VM
   sudo grep PasswordAuthentication /etc/ssh/sshd_config
   ```
   
   By default, Ubuntu cloud images disable password authentication for SSH. To enable:
   ```bash
   sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
   sudo systemctl restart sshd
   ```

### Password doesn't work on console

Make sure you set the password BEFORE starting the VM for the first time. If you need to reset:

```bash
# Stop the VM
qm stop 200

# Update password
qm set 200 --cipassword 'NewPassword'

# Regenerate cloud-init
qm cloudinit pending 200

# Start again
qm start 200
```

### Disk size didn't expand

This should be automatic if you followed Step 5 (resizing in template). If the filesystem still isn't using the full disk, run inside the VM:

```bash
# Check current disk usage
df -h

# If needed, manually expand
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1

# Verify
df -h
```

### Network not working

Check cloud-init network config:
```bash
qm cloudinit dump 200 network
```

## Additional Resources

- [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/)
- [Proxmox Cloud-Init Documentation](https://pve.proxmox.com/wiki/Cloud-Init_Support)
- [Cloud-Init Documentation](https://cloudinit.readthedocs.io/)
