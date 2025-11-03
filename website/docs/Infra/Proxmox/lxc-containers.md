# Running LXC containers

LXC (Linux Containers) is a lightweight virtualization technology that allows you to run multiple isolated Linux systems (containers) on a single host. Proxmox VE supports LXC containers, making it easy to create and manage them.

## Running AlmaLinux containers

The base AlmaLinux image is available in the Proxmox VE template repository. You can create a new LXC container using this template. However the image does not have any basic software installed, so the installation will need to be manual from the Proxmox web interface or CLI.

### Setting up SSH

Run the following:

```bash
dnf update
dnf install -y openssh-server openssh-clients
systemctl enable sshd
systemctl start sshd

# Add public key to authorized keys
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCyv..." > /root/.ssh/authorized_keys
```

### Setting up firewalld

Run the following:

```bash
dnf install -y firewalld
systemctl enable firewalld
systemctl start firewalld

# Allow SSH
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# Check status
firewall-cmd --list-all
```
