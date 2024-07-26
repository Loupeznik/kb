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

## Disk resize

### Partition + LVM

```shell
master@vm-kube-test-master-01:~$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0 111.9M  1 loop /snap/lxd/24322
loop1                       7:1    0    62M  1 loop /snap/core20/1587
loop2                       7:2    0 169.1M  1 loop /snap/microk8s/5643
loop3                       7:3    0  79.9M  1 loop /snap/lxd/22923
loop4                       7:4    0    47M  1 loop /snap/snapd/16292
loop5                       7:5    0  68.8M  1 loop /snap/powershell/242
loop6                       7:6    0  63.5M  1 loop /snap/core20/2015
sda                         8:0    0    20G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0   1.8G  0 part /boot
└─sda3                      8:3    0  18.2G  0 part # partition to resize
  └─ubuntu--vg-ubuntu--lv 253:0    0  18.2G  0 lvm  / # LVM to resize
sdb                         8:16   0   100G  0 disk
sr0                        11:0    1   1.4G  0 rom

master@vm-kube-test-master-01:~$ df -h
/dev/mapper/ubuntu--vg-ubuntu--lv   18G  5.5G   12G  33% /

master@vm-kube-test-master-01:~$ sudo apt install cloud-utils

# resize sda3
master@vm-kube-test-master-01:~$ sudo growpart /dev/sda 3

# resize lvm
master@vm-kube-test-master-01:~$ sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# extend lvm for the changes to take effect
master@vm-kube-test-master-01:~$ sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# reboot for good measure
master@vm-kube-test-master-01:~$ sudo reboot
```

## Mounting an SMB share

```shell
# Install dependencies
sudo apt install cifs-utils psmisc

# Mount the share
SHARE_NAME="myshare"
HOST="MY_IP_OR_HOSTNAME"

sudo mkdir /mnt/$SHARE_NAME

sudo mount -t cifs //$HOST/$SHARE_NAME /mnt/$SHARE_NAME -o username=my_user # specify share username, otherwise will fill current user

# Verify share mounted successfully
ls /mnt/$SHARE_NAME
```

## Get size of directories

Sorting by size from largest to smallest

```shell
DIR="/opt"

du -h --max-depth=1 $DIR | sort -hr
```
