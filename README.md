# Proxmox VM Collection

An Ansible collection for managing Proxmox virtual machines and templates.

## Description

This collection provides playbooks and roles for managing Proxmox virtual machines and templates. It includes functionality for creating VM templates from cloud images and cloning VMs from existing templates with cloud-init configuration.

## Requirements

- **Ansible ≥ 2.16** (see `meta/runtime.yml`).
- **Proxmox VE 9.x** (developed and tested against). The collection only uses generic
  `qm`/`pvesm` CLI commands with no version-specific flags, so other Proxmox VE
  versions may work, but have not been verified.

### Control node prerequisites

- Python 3 with the **`passlib`** package — required for cloud-init template rendering
  (`password_hash('sha512')`). Install it on the control node:
  ```bash
  sudo apt install -y python3-passlib   # or: pip install passlib
  ```

### Target node prerequisites

- SSH access as a user with passwordless `sudo` (or root). Roles run all commands via
  `sudo qm …` / `pvesm status`.
- `qm` and `pvesm` CLI tools installed.
- An active storage pool supporting cloud-init snippets (`snippets_storage`, default `local`).

> For a full list of operational nuances, prerequisites and troubleshooting, see
> [docs/operations.md](docs/operations.md).

## Installation

To install this collection, use the ansible-galaxy command:

```bash
ansible-galaxy collection install opscores.proxmox_vm
```

## Inventory Setup

Minimal inventory targeting a Proxmox host:

```yaml
all:
  children:
    proxmox:
      hosts:
        pve01:
          ansible_host: 192.168.160.200
          ansible_user: "{{ vault_ansible_user | default('ansible') }}"
          ansible_become: true
          ansible_become_method: sudo
          ansible_become_user: root
          proxmox_host: "192.168.160.200"
          proxmox_node: "pve01"
          proxmox_user: "root@pam"
          proxmox_password: "{{ vault_proxmox_password }}"
      vars:
        ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

Sensitive variables (`vault_proxmox_password`, `vault_ansible_user`, `vault_cloud_password`) go in `group_vars/all/vault.yml` — see [docs/inventory.example](docs/inventory.example) for the full annotated version and vault setup instructions.

## Usage

### Creating VM Templates from Cloud Images

```yaml
- name: Create Proxmox Templates from Cloud Images
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    vm_defaults:
      memory: 4096
      cores: 2
      network_bridge: "vmbr0"
    cleanup_downloaded_images: false
  tasks:
    - name: Create Templates
      include_role:
        name: create_templates
```

### Cloning VMs from Templates

```yaml
- name: Clone and Configure VM from Template
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    # Source Template Configuration
    source_vm_id: 3002      # ID of the source template (e.g., 3000 for Ubuntu 24, 3004 for Ubuntu 26, 3001 for AlmaLinux, 3002 for Fedora)
    os_type: "fedora"      # Type of the OS, used to select the cloud-init template ('ubuntu24', 'ubuntu26', 'almalinux', or 'fedora')

    # New VM Configuration
    new_vm_id: 8001         # ID for the new VM
    new_vm_name: "test-fedora" # Desired hostname for the new VM
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password | default('changeme') }}"  # Use ansible-vault
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}" # Path to your public SSH key

    # Proxmox Environment (values from inventory)
    target_node: "{{ proxmox_node }}"
    storage_pool: "{{ vm_storage_pool }}"
    full_clone: true
  tasks:
    - name: Clone VM
      include_role:
        name: clone_vm
```

## Roles

### create_templates

This role creates VM templates from cloud images. It supports Ubuntu, AlmaLinux, Fedora, and any additional OS you add.

#### Supported operating systems

| os_type | Template ID | Template name | Cloud image |
|---|---|---|---|
| `ubuntu24` | 3000 | ubuntu-24-04-template | Ubuntu 24.04 (noble) cloud image |
| `ubuntu26` | 3004 | ubuntu-26-04-template | Ubuntu 26.04 (resolute) cloud image |
| `almalinux` | 3001 | almalinux-9.8-template | AlmaLinux 9.8 GenericCloud |
| `fedora` | 3002 | fedora-45-template | Fedora 45 Cloud Base |

Template IDs and image URLs are defined in `roles/create_templates/defaults/main.yml` (`os_templates`).
Image URLs point at the Yandex mirror (`mirror.yandex.ru`) and must be updated as releases age.

#### Variables

**Global settings:**

- `vm_storage_pool`: Proxmox storage pool (default: "local-lvm")
- `snippets_path`: Path to Proxmox snippets directory (default: "/var/lib/vz/snippets")
- `cleanup_downloaded_images`: Whether to remove downloaded images after import (default: false)
- `recreate_templates`: Whether to recreate existing templates (default: false)
- `tmp_min_free_gb`: Minimum free space in /tmp in GB (default: 5)
- `storage_min_free_gb`: Minimum free space in storage pool in GB (default: 10)
- `template_min_disk_size`: Minimum disk size for created templates, e.g. "12G" (default). Cloud images often ship a tiny root with no free space after it, so cloud-init growpart cannot expand the root filesystem on first boot ("No space left on device"). Resizing the template disk up here leaves headroom at the end; growpart then grows root to fill it. Do not lower this below ~12G.

**VM defaults (applied to all templates unless overridden):**

```yaml
vm_defaults:
  memory: 1024
  cores: 1
  network_bridge: "vmbr0"
  cpu_type: "host"
  ostype: "l26"
  machine: "q35"
  bios: "seabios"
  scsihw: "virtio-scsi-pci"
  agent: "enabled=1"
```

**OS templates (single source of truth):**

```yaml
os_templates:
  ubuntu24:
    template_id: 3000
    vm_name: "ubuntu-24-04-template"
    image_url: "https://mirror.yandex.ru/..."
    image_filename: "noble-server-cloudimg-amd64.img"
    enabled: true
    cloud_init_template: ubuntu24
    # Optional per-OS overrides:
    # memory: 2048
    # cores: 2
    # scsi_disk_slot: sata0   # disk slot to attach/resize (default: scsi0)
```

See [docs/overrides.md](docs/overrides.md) for examples of overriding parameters and adding new OS.

### clone_vm

This role clones VMs from existing templates with cloud-init configuration.

The role includes the following checks:
- Verifies that the source template exists
- Confirms that the source VM is actually a template (template: 1)
- Validates that disk resize operation completes successfully (if enabled)
- **Validates that cloud-init template exists for os_type**

#### Variables

- `source_vm_id`: ID of the source template
- `new_vm_id`: ID for the new VM
- `new_vm_name`: Hostname for the new VM
- `os_type`: Type of the OS ('ubuntu24', 'ubuntu26', 'almalinux', 'fedora', or custom)
- `target_node`: Proxmox node to create VM on
- `storage_pool`: Storage pool to use for VM
- `full_clone`: Whether to perform a full clone
- `recreate_vm`: Whether to recreate the VM if it already exists (default: false)
- `snippets_path`: Path to Proxmox snippets directory (default: "/var/lib/vz/snippets")
- `snippets_storage`: Proxmox storage name for cloud-init snippets (default: "local")
- `timezone`: Timezone for the cloned VM (default: "Europe/Moscow")
- `clone_vm_memory`: Memory size for the cloned VM (overrides default)
- `clone_vm_cores`: Number of CPU cores for the cloned VM (overrides default)
- `clone_vm_ostype`: OS type for the cloned VM (overrides default)
- `clone_vm_machine`: Machine type for the cloned VM (overrides default)
- `clone_vm_bios`: BIOS type for the cloned VM (overrides default)
- `clone_vm_cpu`: CPU type for the cloned VM (overrides default)
- `clone_vm_scsihw`: SCSI hardware type for the cloned VM (overrides default)
- `clone_vm_agent`: Guest agent configuration for the cloned VM (overrides default)
- `clone_vm_disk_size`: Disk size for the cloned VM (empty by default -> keeps template disk size)
- `vm_resize_disk`: Master toggle for disk resize when cloning (default: true). Set to false to skip resizing even if `clone_vm_disk_size` is set
- `clone_vm_disk_slot`: Disk slot to resize (default: scsi0). Set explicitly if the template uses a different slot (e.g. sata0, virtio0).
- `update_vm_config`: Reconcile memory/cores/cpu on skip without recreating the VM (default: false). Cloud-init changes still require `recreate_vm: true`.
- **Bulk clone** — create N VMs from one template:
  - `vm_count`: number of VMs to clone (default 1 = backward compat). VM IDs are contiguous starting at `new_vm_id`.
  - `new_vm_ids`: explicit list of VM IDs (takes priority over `vm_count`).
  - `vm_name_pattern`: name template with `{index}` (padded) / `{index_raw}` (raw) placeholders.
  - `vm_name_index_padding`: padding width (`0` = none, `>0` = min digits; auto-scales up for large counts).
  - `vm_id_min`: optional lower bound for generated IDs (`0` = no limit).
- **Safety**: with `recreate_vm: false`, cloning fails on a foreign VM ID collision (same ID, different name) and always refuses to destroy templates. See [docs/operations.md](docs/operations.md#8-bulk-clone).
- `vm_resize_disk`: Whether to resize disk when cloning (default: true)
- `cloud_user`: Username for the cloud-init user
- `cloud_password`: Password for the cloud-init user
- `ssh_public_key`: SSH public key for the cloud-init user
- `dns_servers`: DNS servers for the VM
- `search_domains`: Search domains for the VM
- `use_dhcp_ip`: Whether to use DHCP instead of static IP (default: true)
- `static_ip_address`: Static IP address to assign to the VM (e.g., "192.168.1.100")
- `static_ip_addresses`: Per-VM static IPs for bulk clone (list; takes priority over `static_ip_address` when non-empty). Length must match the number of cloned VMs.
- `static_subnet_mask`: Subnet mask in CIDR format (default: "24")
- `static_gateway`: Static gateway address (e.g., "192.168.1.1")

## Safety & Idempotency Notes

- **Template protection**: `create_templates` refuses to recreate an existing VM that is not a template (`template: 1` guard). `clone_vm` always refuses to destroy templates.
- **Idempotency (limitation)**: on a repeated run with the same parameters the existing VM is skipped (changed=0) **without updating its configuration** (memory, cores, …). To change a VM's config use `update_vm_config: true` (reconciles memory/cores/cpu without recreating) or `recreate_vm: true` (destroys and recreates).
- **`vm_id_min`**: validation only — the role checks that generated IDs are not below `vm_id_min`, but does not shift IDs automatically. Set `new_vm_id` ≥ `vm_id_min` from the start.
- **`full_clone: false`** (linked clone): the clone depends on the source template — recreating the template breaks all linked clones. Default is `true`.
- **`clone_vm_disk_slot` / `scsi_disk_slot`**: default `scsi0`. Set explicitly (e.g. `sata0`, `virtio0`) if the template uses a non-standard slot.

## Security Considerations

For production use, consider the following security improvements:

1. Use Ansible Vault to encrypt sensitive variables like passwords
2. Use SSH key authentication instead of passwords
3. Implement proper network security policies

## License

GPL-3.0-or-later
