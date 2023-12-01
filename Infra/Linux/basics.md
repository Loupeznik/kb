# Linux management basics

## User accounts

### Create a user account

Sudo-enabled account:

```bash
sudo useradd -m -s /bin/bash -G sudo <username>
```

Non-sudo account:

```bash
sudo useradd -m -s /bin/bash <username>
```

### Add user to group

```bash
sudo usermod -a -G <group> <username>
```
