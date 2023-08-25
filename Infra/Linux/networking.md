# Networking
## Set static IP (Ubuntu/Debian-based systems)
Add the following configuration to `/etc/netplan/99_config.yaml`:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.10.10.2/24
      gateway4: 10.10.10.1
      nameservers:
          search: [mydomain, otherdomain]
          addresses: [10.10.10.1, 1.1.1.1]
```
Set the correct IP addresses and nameservers and call `sudo netplan apply`
