---
title: Publishing snaps from macOS (multipass + snapcraft)
sidebar_position: 10
---

# Publishing snaps from macOS (multipass + snapcraft)

Snap packages can only be built on Linux — `snapcraft` needs `snapd`, which needs a Linux kernel. On macOS the official path is to spin up a Linux VM via [multipass](https://multipass.run/). This page collects the sharp edges you'll hit the first time, especially around:

- Multipass networking on macOS (the "No route to host" problem even when `ping` works).
- The snap-store chicken-and-egg where a registered-but-never-published name refuses `snapcraft upload`.
- Authentication via exported store credentials instead of interactive login.

## Multipass: "No route to host" on `multipass exec` / `multipass transfer`

### Symptoms

```text
$ multipass exec snap-bootstrap -- true
exec failed: ssh connection failed: 'Failed to connect: No route to host'

$ multipass transfer ./somefile snap-bootstrap:/home/ubuntu/
ssh connection failed: 'Failed to connect: No route to host'
```

Even though:

```text
$ multipass list
Name              State     IPv4          Image
snap-bootstrap    Running   192.168.2.2   Ubuntu 24.04 LTS

$ ping -c 3 192.168.2.2
3 packets transmitted, 3 packets received, 0.0% packet loss

$ nc -vz 192.168.2.2 22
Connection to 192.168.2.2 port 22 [tcp/ssh] succeeded!
```

The VM is up, the network is fine, sshd is listening — only `multipassd`'s internal SSH channel is refusing to connect. This is a **multipass daemon bug**, not a networking problem.

### Fixes, in order of effort

#### 1. Restart the multipass daemon

Usually the fastest fix:

```sh
sudo launchctl kickstart -k system/com.canonical.multipassd
sleep 3
multipass exec snap-bootstrap -- true
```

#### 2. Stop and start the VM

If the daemon restart alone isn't enough:

```sh
multipass stop snap-bootstrap
multipass start snap-bootstrap
sleep 10
multipass exec snap-bootstrap -- true
```

#### 3. Bypass `multipass exec` entirely — SSH directly

When `ping` and `nc -vz 22` both succeed but `multipass exec` still fails, you can skip the daemon's channel and SSH straight to the VM using multipass's own private key:

```sh
sudo cat "/var/root/Library/Application Support/multipassd/ssh-keys/id_rsa" > /tmp/mp.key
chmod 600 /tmp/mp.key
ssh -i /tmp/mp.key -o StrictHostKeyChecking=no ubuntu@192.168.2.2
```

The `ubuntu` user is multipass's default. Once inside, you have a normal shell and can do anything `multipass exec` would have done. For one-off file transfers while the daemon is wedged:

```sh
# Copy INTO the VM via ssh
scp -i /tmp/mp.key ./localfile ubuntu@192.168.2.2:/home/ubuntu/

# Or pipe
cat localfile | ssh -i /tmp/mp.key ubuntu@192.168.2.2 'cat > /home/ubuntu/file'
```

Cleanup the key when done:

```sh
rm /tmp/mp.key
```

#### 4. Last resort — purge and relaunch

If none of the above works the VM may be irrecoverably wedged:

```sh
multipass delete snap-bootstrap && multipass purge
multipass launch --name snap-bootstrap --cpus 2 --memory 4G --disk 10G 24.04
```

Fresh VMs almost always work cleanly on the first `multipass exec` after `launch`.

### Alternative workflow: use the repo mount

Before fighting multipass-exec, remember that `multipass launch` auto-mounts your home directory (or you can `multipass mount` an arbitrary path). Once the mount exists, host-side writes are immediately visible inside the VM without needing `multipass transfer`:

```sh
multipass mount "$(pwd)" snap-bootstrap:/home/ubuntu/project
# put whatever you need on the host side:
cp /tmp/secret.txt ./.temp/secret.txt
# and reference it inside the VM:
ssh -i /tmp/mp.key ubuntu@192.168.2.2 'ls /home/ubuntu/project/.temp/secret.txt'
```

Gitignore the staging path so it doesn't leak into source control.

## Snap Store: registered-but-never-published names

### Symptoms

Uploading a brand-new snap fails with:

```text
Store operation failed:
- resource-not-found: Snap not found for name=<your-snap-name>
```

This happens on both host-side (`snapcraft upload` after `snapcraft login`) and inside CI (goreleaser's `snapcraft upload --release=edge,beta …`).

### Cause

A snap name goes through two distinct states in the store:

1. **Registered** — reserved via `snapcraft register` or the dashboard. The name is yours, but no revision has ever been uploaded.
2. **Published** — at least one revision exists. The name is now a real snap with a metadata page, channel map, etc.

The `snapcraft upload` endpoint (and the ACLs that credentials mint against) treat these differently. A registered-only name can't be the target of `upload` from any client — the store returns `resource-not-found`. You need to make the first upload through a path that transitions the name from registered to published.

The dashboard's "Publish to this name" link next to the registered name just opens the snapcraft publishing docs — it is **not** a wizard that activates the name. Ignore that button.

### Fix: one manual bootstrap upload from Linux

Any other upload attempt (CLI, CI, goreleaser) will keep failing with the same error until a first revision exists. Do it exactly once, from a real Linux host.

1. **Export store credentials on the host** (the browser flow works where the docker image's email/password flow doesn't):

    ```sh
    snapcraft export-login \
      --acls=package_access,package_push,package_update,package_release \
      --expires=2030-01-01 /tmp/snap-credentials.txt
    ```

    Omit `--snaps=<name>` — the scoping requires the snap to already be published. Omit `--channels` so the credential works for any channel.

2. **Get the credentials + a built `.snap` into a Linux VM.** Multipass is the common choice on macOS:

    ```sh
    multipass launch --name snap-bootstrap --cpus 2 --memory 4G --disk 10G 24.04
    multipass mount "$(pwd)" snap-bootstrap:/home/ubuntu/project
    cp /tmp/snap-credentials.txt ./.temp/snap-credentials.txt
    ```

3. **Inside the VM, build and upload:**

    ```sh
    sudo snap install snapcraft --classic
    sudo snap install go --classic
    sudo snap install goreleaser --classic --edge   # or wget the binary

    export SNAPCRAFT_STORE_CREDENTIALS="$(cat /home/ubuntu/project/.temp/snap-credentials.txt)"

    cd /home/ubuntu/project
    git checkout <your-tag>
    goreleaser release --snapshot --clean --skip=publish   # produces dist/*.snap
    snapcraft upload --release=edge dist/<name>_*_linux_amd64.snap
    ```

    Expected output: `Revision 1 of '<name>' created` and `Released to edge`.

4. **Done.** The name is now published. Subsequent `snapcraft upload` calls (from CI, from your laptop, from wherever) will succeed because the name is in the published state.

### Caveats

- **Classic confinement gates `stable`, not `edge`/`beta`.** If your snap uses `confinement: classic`, the first upload enqueues a one-time manual review by the snap store team before the snap can be released to the `stable` channel. Until that clears, only `edge`/`beta`/`candidate` work. Request review at [forum.snapcraft.io/c/store-requests/19](https://forum.snapcraft.io/c/store-requests/19).
- **`grade: devel`** also blocks `candidate` and `stable` channels — devel revisions only release to `edge` and `beta`. Use a goreleaser template to flip grade at the right semver cut-over:

    ```yaml
    grade: '{{ if eq .Major 0 }}devel{{ else }}stable{{ end }}'
    ```

- **snapcraft in docker (`snapcore/snapcraft:stable`)** can't complete the browser-based 2FA login flow. Use multipass, a cloud Linux VM, or GitHub Actions for the bootstrap instead of the container.

## goreleaser + snapcraft: useful non-obvious flags

- `snapshot.version_template: "{{ incpatch .Version }}-next"` in `.goreleaser.yaml` is why `goreleaser release --snapshot` produces `0.10.3-next` and not the current tag. Tweak the template or pass `--snapshot` from the actual tag commit to get the version you want in artifact filenames.
- `goreleaser release --snapshot --clean --skip=publish` builds everything including `.snap` without uploading — good for local dry-runs and the bootstrap flow above.
- `publish: false` under `snapcrafts:` builds the `.snap` but doesn't try to upload to the store. Useful while you're waiting on name activation or classic-confinement review so unrelated release channels still succeed.

## Verifying a successful snap publish

```sh
snap info <your-snap-name> --color=never | head -30
# Should show "snap-id", "publisher", and a channels section with the revision.

# From any Linux:
sudo snap install <your-snap-name> --classic --edge   # or --stable once grade flips

# From the snap store API:
curl -s "https://api.snapcraft.io/v2/snaps/info/<your-snap-name>" \
  -H 'Snap-Device-Series: 16' | jq '.channel-map[] | {channel, revision, version}'
```
