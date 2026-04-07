---
name: alpine
description: Install and provision Alpine Linux on Proxmox VMs via swamp. Use when the user wants to PXE-install Alpine to disk with setup-alpine, package and deploy an apkovl overlay (lbu package) for diskless boot, or apk-install packages and enable OpenRC services on a running Alpine VM. Triggers on "install alpine", "alpine setup", "setup-alpine", "apkovl", "lbu package", "apk add", "rc-update", "alpine pxe", "alpine overlay", "provision alpine packages", "@swampadmin/alpine/install", "@swampadmin/alpine/overlay", "@swampadmin/alpine/packages", "create-stateful-vm", "deploy-apkovl", or "provision-packages".
---

# alpine

Swamp extension for installing and provisioning Alpine Linux on Proxmox VMs.
Provides three models plus three workflows that orchestrate them with the
`@swampadmin/proxmox` extension.

## Dependencies

- `@swampadmin/proxmox` — required. Provides the `keebDev02` (auth) and `fleet`
  (VM lifecycle) models that every alpine workflow depends on.

## Models

All three models share the same `globalArguments` schema and require SSH
reachability to the target VM.

### `@swampadmin/alpine/install`

PXE-install Alpine to a persistent disk via `setup-alpine`, then post-install
chroot setup (SSH keys, repos, qemu-guest-agent).

- **globalArguments**:
  - `sshHost` (string, required) — IP/hostname of the PXE-booted VM
  - `sshUser` (string, default `root`)
- **methods**:
  - `install(hostname, password, disk?)` — `disk` defaults to `/dev/sda`.
    Generates a `setup-alpine.conf` answer file, runs
    `ERASE_DISKS=<disk> setup-alpine -e -f /tmp/setup-alpine.conf`, then mounts
    `<disk>3` and `<disk>1` and chroots in to copy authorized_keys, rewrite
    `/etc/apk/repositories` to `latest-stable`, install qemu-guest-agent, and
    set the root password.
- **resources**: `result` (lifetime `infinite`, gc 10).

### `@swampadmin/alpine/overlay`

Package an Alpine LBU overlay on a "gold-image" VM and deploy it to a TFTP
server for diskless PXE boot.

- **globalArguments**: `sshHost`, `sshUser` (same shape as install).
- **methods**:
  - `deployApkovl(tftpHost, tftpPath)` — runs
    `lbu package /tmp/<hostname>.apkovl.tar.gz` on the VM, SCPs it down to local
    `/tmp`, then SCPs it up to
    `root@<tftpHost>:<tftpPath>/alpine.apkovl.tar.gz`. The destination filename
    is always `alpine.apkovl.tar.gz` regardless of the VM hostname.
- **resources**: `overlay` (lifetime `infinite`, gc 10).

### `@swampadmin/alpine/packages`

Install apk packages and optionally enable OpenRC services on a running Alpine
VM.

- **globalArguments**: `sshHost`, `sshUser`.
- **methods**:
  - `provision(packages[], services?[])` — runs `apk update && apk add <pkgs>`,
    then `rc-update add <svc> default && rc-service <svc> start` for each
    service.
- **resources**: `result` (lifetime `infinite`, gc 10).

## Workflows

### `@swampadmin/create-stateful-vm`

Inputs: `vmName`, `memory` (MB), `cores`, `diskSize` (GB).

Steps: `auth` (keebDev02) -> `fleet.create` -> `fleet.start` (PXE boot) ->
`alpineInstaller.install` -> `fleet.setBootOrder order=scsi0;net0` ->
`fleet.stop` -> `fleet.start` (boot from disk).

### `@swampadmin/deploy-apkovl`

No workflow inputs. Operates on a hard-coded `vmName: gold-image`. Steps: `auth`
-> `fleet.start gold-image` -> `goldImageOverlay.deployApkovl`.

### `@swampadmin/provision-packages`

Inputs: `vmName`, `packages[]`, optional `services[]`. Steps: `auth` ->
`fleet.start` -> `alpinePackages.provision`.

## Configuring model instances

Each workflow references model instance names (`alpineInstaller`,
`goldImageOverlay`, `alpinePackages`) — these must exist in the user's swamp
config and bind a `sshHost` value via CEL from the proxmox `fleet` output. The
canonical pattern:

```yaml
models:
  - name: alpineInstaller
    type: "@swampadmin/alpine/install"
    globalArguments:
      sshHost: ${{ models.fleet.vms[inputs.vmName].ip }}
      sshUser: root

  - name: alpinePackages
    type: "@swampadmin/alpine/packages"
    globalArguments:
      sshHost: ${{ models.fleet.vms[inputs.vmName].ip }}

  - name: goldImageOverlay
    type: "@swampadmin/alpine/overlay"
    globalArguments:
      sshHost: ${{ models.fleet.vms["gold-image"].ip }}
```

The `fleet.start` step must run before any alpine method so the VM has an IP the
CEL expression can resolve. Every model throws
`sshHost is required — VM must be running with an IP` if the value is empty.

## Gotchas

- **PXE kernel SCSI modules**: `install` explicitly `modprobe`s `virtio_scsi`
  and `sd_mod` and waits up to 60s for the disk node before invoking
  `setup-alpine`. The PXE-booted netboot kernel does not autoload them.
- **Disk partition layout is hardcoded**: post-install assumes `<disk>3` is root
  and `<disk>1` is `/boot` (default `setup-alpine -m sys` layout). Custom
  partition schemes will break the chroot mounts.
- **Authorized_keys source**: install copies `/root/.ssh/authorized_keys` from
  the _PXE environment_ into the new system. The PXE image must already have the
  operator's public key baked in.
- **Repos pinned to `latest-stable`**: install rewrites `/etc/apk/repositories`
  to `dl-cdn.alpinelinux.org/alpine/latest-stable` main
  - community. No version pinning is supported.
- **TFTP destination is fixed**: `deployApkovl` always writes
  `<tftpPath>/alpine.apkovl.tar.gz` on the TFTP host as `root` over SCP. The
  TFTP server must accept root SSH key auth from the swamp host.
- **No password sanitization**: the root password is interpolated directly into
  shell strings (`echo 'root:${password}' | chpasswd`). Avoid single quotes,
  newlines, or shell metacharacters in passwords — use vault references and
  alphanumerics.
- **`deploy-apkovl` is hard-coded to `gold-image`**: there are no inputs. To
  deploy a different overlay source, edit the workflow YAML.
- **SSH host key checking is disabled** (`StrictHostKeyChecking=no`,
  `UserKnownHostsFile=/dev/null`). Acceptable for ephemeral PXE VMs; treat any
  long-lived target host as untrusted.
- **Service enable runs `rc-service start` immediately** in `provision`. If a
  service requires config files that have not been laid down yet, start will
  fail and the whole step errors out.

## Running

```bash
# Provision a fresh stateful VM end-to-end
swamp workflow run @swampadmin/create-stateful-vm \
  --input vmName=db01 --input memory=4096 --input cores=2 --input diskSize=40

# Refresh the diskless gold-image overlay
swamp workflow run @swampadmin/deploy-apkovl

# Add packages to a running VM
swamp workflow run @swampadmin/provision-packages \
  --input vmName=db01 \
  --input 'packages=["postgresql","postgresql-contrib"]' \
  --input 'services=["postgresql"]'
```
