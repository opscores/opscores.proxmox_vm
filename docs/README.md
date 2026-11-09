# Proxmox VM Collection Documentation

## Overview

This collection provides playbooks and roles for managing Proxmox virtual machines and templates. It includes functionality for creating VM templates from cloud images and cloning VMs from existing templates with cloud-init configuration.

## Installation

To install this collection, use the ansible-galaxy command:

```bash
ansible-galaxy collection install opscores.proxmox_vm
```

## Usage

### Creating VM Templates from Cloud Images

To create VM templates from cloud images, use the `create-vm-templates.yml` playbook:

```yaml
- name: Create Proxmox Templates from Cloud Images
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    vm_defaults:
      memory: 4096
      cores: 2
      disk_size: "12G"
      network_bridge: "vmbr0"
    cleanup_downloaded_images: false
  tasks:
    - name: Create Templates
      include_role:
        name: create_templates
```

### Cloning VMs from Templates

To clone VMs from existing templates, use the `clone-vm-from-template.yml` playbook:

```yaml
- name: Clone and Configure VM from Template
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    # Source Template Configuration
    source_vm_id: 3001      # ID of the source template (e.g., 3000 for Ubuntu 24, 3004 for Ubuntu 26, 3001 for AlmaLinux, 3002 for Fedora)
    os_type: "almalinux"      # Type of the OS, used to select the cloud-init template ('ubuntu24', 'ubuntu26', 'almalinux', or 'fedora')

    # New VM Configuration
    new_vm_id: 8001         # ID for the new VM
    new_vm_name: "test-almalinux" # Desired hostname for the new VM
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

This role creates VM templates from cloud images. It supports Ubuntu, AlmaLinux, and Fedora distributions.

#### Variables

**Global settings:**

- `vm_storage_pool`: Proxmox storage pool (default: "local-lvm")
- `snippets_path`: Path to Proxmox snippets directory (default: "/var/lib/vz/snippets")
- `tmp_min_free_gb`: Minimum free space in /tmp in GB (default: 5)
- `storage_min_free_gb`: Minimum free space in storage pool in GB (default: 10)
- `cleanup_downloaded_images`: Whether to remove downloaded images after import (default: false)
- `recreate_templates`: Whether to recreate existing templates (default: false)

**VM defaults (applied to all templates unless overridden):**

```yaml
vm_defaults:
  memory: 1024
  cores: 1
  disk_size: "12G"
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
    # memory: 2048, cores: 2, disk_size: "20G", network_bridge: "vmbr1", etc.
    # recreate: true, cleanup_image: true
  ubuntu26:
    template_id: 3004
    vm_name: "ubuntu-26-04-template"
    image_url: "https://mirror.yandex.ru/..."
    image_filename: "resolute-server-cloudimg-amd64.img"
    enabled: true
    cloud_init_template: ubuntu26
  almalinux:
    template_id: 3001
    vm_name: "almalinux-9-template"
    image_url: "https://mirror.yandex.ru/..."
    image_filename: "AlmaLinux-9-GenericCloud-9.7-20251118.x86_64.qcow2"
    enabled: true
    cloud_init_template: almalinux
  fedora:
    template_id: 3002
    vm_name: "fedora-43-template"
    image_url: "https://mirror.yandex.ru/..."
    image_filename: "Fedora-Cloud-Base-Generic-43-1.6.x86_64.qcow2"
    enabled: true
    cloud_init_template: fedora
```

See [docs/overrides.md](docs/overrides.md) for examples of overriding parameters and adding new OS.

### clone_vm

This role clones VMs from existing templates with cloud-init configuration.

The role includes the following checks:
- Verifies that the source template exists
- Confirms that the source VM is actually a template (template: 1)
- Validates that disk resize operation completes successfully (if enabled)

#### Variables

- `source_vm_id`: ID of the source template
- `new_vm_id`: ID for the new VM
- `new_vm_name`: Hostname for the new VM
- `os_type`: Type of the OS, used to select the cloud-init template ('ubuntu24', 'ubuntu26', 'almalinux', or 'fedora')
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
- `clone_vm_disk_size`: Disk size for the cloned VM (overrides default)
- `clone_vm_disk_slot`: Disk slot to resize (default: scsi0)
- `vm_resize_disk`: Whether to resize disk when cloning (default: true)
- `cloud_user`: Username for the cloud-init user
- `cloud_password`: Password for the cloud-init user
- `ssh_public_key`: SSH public key for the cloud-init user
- `dns_servers`: DNS servers for the VM
- `search_domains`: Search domains for the VM
- `use_dhcp_ip`: Whether to use DHCP instead of static IP (default: true)
- `static_ip_address`: Static IP address to assign to the VM (e.g., "192.168.1.100")
- `static_subnet_mask`: Subnet mask in CIDR format (default: "24")
- `static_gateway`: Static gateway address (e.g., "192.168.1.1")

## Security Considerations

For production use, consider the following security improvements:

1. Use Ansible Vault to encrypt sensitive variables like passwords
2. Use SSH key authentication instead of passwords
3. Implement proper network security policies
4. Regularly update cloud images to include security patches

## License

GPL-3.0-or-later
