# Proxmox - cloud init and VM templates

## Create a VM template from an image

```shell
# If the tools haven't been installed yet
apt update && apt install -y libguestfs-tools

# Download the image
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2

# Customize the image
virt-customize -a debian-12-generic-amd64.qcow2 --install qemu-guest-agent --root-password password:securePassword123 \
--append-line "/etc/ssh/sshd_config.d/rootlogin.conf":"PermitRootLogin yes" \
--append-line "/etc/ssh/sshd_config.d/rootlogin.conf":"PasswordAuthentication no" \
--append-line "/etc/ssh/sshd_config.d/rootlogin.conf":"AuthorizedKeysFile .ssh/authorized_keys" \
--append-line "/etc/ssh/sshd_config.d/rootlogin.conf":"PubkeyAuthentication yes" \
--firstboot-command 'ssh-keygen -A && systemctl restart sshd'

# Create and setup the VM
NAME="debian12-tpl" # name of the VM
VMID="10000" # desired VM ID
DISKID="ssd-storage-1" # name of the disk where the VM files will be stored

qm create $VMID --name $NAME --memory 1024 --cores 1 --net0 virtio,bridge=vmbr0
qm importdisk $VMID debian-12-generic-amd64-tpl.qcow2 $DISKID
qm set $VMID --scsihw virtio-scsi-pci --scsi0 $DISKID:$VMID/vm-$VMID-disk-0.raw
qm set $VMID --boot c --bootdisk scsi0
qm set $VMID --ide2 $DISKID:cloudinit
qm set $VMID --agent enabled=1

# Convert the VM to template
qm template $VMID

```
