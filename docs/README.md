# Proxmox VM Collection — Documentation Index

The full collection documentation lives in the root [`README.md`](../README.md).
This directory contains focused guides:

| Document | Description |
|---|---|
| [`README.md`](../README.md) | Full reference: requirements, inventory, usage, roles, variables, security |
| [`operations.md`](operations.md) | Prerequisites (passlib, vault, storage/snippets), disk sizing, bulk clone, idempotency, troubleshooting |
| [`examples.md`](examples.md) | Playbook examples: templates, single clone, bulk clone, static/dhcp, safety |
| [`overrides.md`](overrides.md) | Overriding parameters, per-OS settings, adding a new OS |
| [`inventory.example`](inventory.example) | Annotated inventory with operational notes |

## Quick links

- **Install:** `ansible-galaxy collection install opscores.proxmox_vm`
- **Vault:** `ansible-vault encrypt group_vars/all/vault.yml` → run with `--ask-vault-pass`
- **Control node:** requires `python3-passlib` (`apt install python3-passlib`)
- **Templates:** IDs 3000 (Ubuntu 24), 3004 (Ubuntu 26), 3001 (AlmaLinux 9.8), 3002 (Fedora 45) — all 12G
- **VMs:** typically 8000-range (8001+)
- **Bulk clone:** `vm_count`, `new_vm_ids`, `vm_name_pattern` (`{index}`/`{index_raw}`)

See [`operations.md`](operations.md) for the complete operational guide and troubleshooting table.
