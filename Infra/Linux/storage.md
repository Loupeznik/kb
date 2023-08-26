# Storage
## Mounting a new disk
```shell
# Find the new device
lsblk

# Partition and format the disk (use the device ID from the previous step)
sudo parted /dev/sdb
(parted) mklabel gpt
(parted) mkpart primary ext4 0% 100% # use 100% of the available space 
(parted) quit

sudo mkfs.ext4 /dev/sdb1

# [OPTIONAL] Check if the partition was created
lsblk

# Create a mount
sudo mkdir /mnt/storage
sudo mount /dev/sdb1 /mnt/storage

# Persist the mount
sudo vi /etc/fstab

# Add the following line
/dev/sdb1   /mnt/storage   ext4   defaults   0  2
```
