# Install and configure Samba

## Install Samba

### Ubuntu

```shell
sudo apt update && sudo apt install -y samba
sudo groupadd smb
sudo useradd -m -g smb -s /bin/bash smb_myuser
sudo smbpasswd -a smb_myuser

# Change permissions on the shares
sudo chown -R smb_myuser:smb /mnt/storage1
```

### RHEL

```shell
sudo dnf update -y
sudo dnf install samba samba-common
sudo systemctl enable smb nmb
sudo systemctl start smb nmb
```

Then follow instructions as in the previous section.

## Configure Samba

Configure Samba by editing `/etc/samba/smb.conf`

```
[backups]
    comment = Backups
    path = /mnt/storage1/backups
    read only = no
    browsable = yes
    writable = yes
    guest ok = no
    valid users = @smb_myuser

[docs]
    comment = Shared documents
    path = /mnt/storage1/docs
    read only = no
    browsable = yes
    writable = yes
    guest ok = no
    valid users = @smb_myuser
```

Restart samba service: `sudo service smbd restart`
