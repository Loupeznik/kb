# Firewalld
## Set firewall rules
```shell
sudo firewall-cmd --zone=public --permanent --add-service=https
sudo firewall-cmd --zone=public --permanent --add-port=5432/tcp
```

## Reload firewalld
```shell
sudo firewall-cmd --reload
```
