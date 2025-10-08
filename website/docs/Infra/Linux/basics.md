# Linux management basics

## User accounts

### Create a user account

Sudo-enabled account:

```shell
sudo useradd -m -s /bin/shell -G sudo <username>
```

Non-sudo account:

```shell
sudo useradd -m -s /bin/shell <username>
```

### Add user to group

```shell
sudo usermod -a -G <group> <username>
```

### Allow passwordless sudo

Edit the /etc/sudoers file or run `sudo visudo`:

```
# All commands for all users in the sudo group
%sudo ALL=(ALL) NOPASSWD:ALL

# Specific command for a specific user
<user> ALL=(ALL) NOPASSWD: <full_path_to_binary>
```

This will allow all users in the sudo group to run sudo commands without specifying a password.

### Create a cron job from the command line

This is useful for headless installs or dockerfiles.

```shell
cronfile="crontab-test-install"

mkdir -p /tmp/crontab-test-install && cd /tmp/crontab-test-install

echo "* * * * * curl https://google.com > /dev/null" >> crontab-test

crontab crontab-test
rm -rf /tmp/crontab-test-install
```

### Fix LF/CRLF issues

Using vim:

```shell
vim <filename> -c "set ff=unix" -c ":wq"
```

### Stop script execution if not running under root

```shell
if [[ $EUID -ne 0 ]]; then
   echo "[ERROR] This script must be run as root"
   exit 1
fi
```
