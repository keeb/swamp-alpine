# @keeb/alpine

[Swamp](https://github.com/systeminit/swamp) extension for Alpine Linux disk installation and overlay packaging.

## Models

### `alpine/install`

Install Alpine Linux to disk on a PXE-booted VM using `setup-alpine` with chroot-based post-install.

| Method | Description |
|--------|-------------|
| `install` | Run setup-alpine, configure disk, install packages, enable services |

### `alpine/overlay`

Package and deploy Alpine overlay files (apkovl) for diskless PXE boot.

| Method | Description |
|--------|-------------|
| `deployApkovl` | Run `lbu package` on gold-image VM and SCP result to TFTP/HTTP server |

## Workflows

| Workflow | Description |
|----------|-------------|
| `create-stateful-vm` | Full provisioning: create VM, PXE boot, install Alpine to disk, set boot order, reboot |
| `deploy-apkovl` | Package gold-image overlay and deploy to TFTP server |

## Dependencies

- [@keeb/proxmox](https://github.com/keeb/swamp-proxmox) — VM creation, start, boot order, fleet sync

## Install

```bash
swamp extension pull @keeb/alpine
```

## Example

Install Alpine to disk on a PXE-booted VM, then install and enable services:

```yaml
models:
  - name: install
    type: "@keeb/alpine/install"
    globalArguments:
      sshHost: "10.0.0.42"
      sshUser: "root"
  - name: packages
    type: "@keeb/alpine/packages"
    globalArguments:
      sshHost: "10.0.0.42"
      sshUser: "root"

jobs:
  - name: bootstrap
    steps:
      - model: install
        method: install
        inputs:
          hostname: "web01"
          password: "${vault.alpine.rootPassword}"
          disk: "/dev/sda"
      - model: packages
        method: provision
        inputs:
          packages: ["docker", "docker-openrc", "curl"]
          services: ["docker"]
```

## License

MIT
