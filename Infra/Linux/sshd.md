# SSHd
## Setting ACLs
- To allowlist or blacklist IP addresses for sshd, edit `/etc/hosts.allow` or `/etc/hosts.deny`
- 1 IP address per row (not sure if it is possible to use CIDR as well)
```
sshd: 100.94.51.71
```

## Allow root SSH access to a machine
```shell
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config && service sshd restart
```
