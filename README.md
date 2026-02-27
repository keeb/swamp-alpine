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

## License

MIT
