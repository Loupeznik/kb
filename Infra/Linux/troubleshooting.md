# Troubleshooting guides
## Troubleshooting RHEL package managers
```shell
# Problem
$ sudo rpmdb --rebuilddb -v
error: could not delete old database at /var/lib/rpmold.17138

# Solution
$ sudo rm -rf /var/lib/rpmold.17138

$ sudo rpmdb --rebuilddb -v

$ sudo dnf update --refresh
```
