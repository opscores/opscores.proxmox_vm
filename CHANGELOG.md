# Changelog

All notable changes to `opscores.proxmox_vm` are documented here.

## [1.1.0] - 2026-11-10

### Added
- Bulk cloning: `vm_count`, `new_vm_ids`, `vm_name_pattern` (`{index}`/`{index_raw}`), `vm_name_index_padding` (auto-scales), `vm_id_min`.
- Safety: template protection (refuse to destroy `template: 1`), foreign VM ID collision detection, recreate ownership check.
- `static_ip_addresses` list for bulk static IP assignment.
- `template_min_disk_size` (default 12G) to leave headroom for cloud-init growpart.
- `docs/operations.md`: prerequisites, vault, storage/snippets, disk sizing, bulk clone, idempotency, troubleshooting, footguns.
- `docs/examples.md`: bulk clone examples, static IP, recreate.
- `docs/overrides.md`: cloud-init differences by OS, supported OS matrix in README.
- `docs/ansible.cfg.example`, `docs/inventory.example` operational notes.

### Removed
- `wait_for_cloud_init` optional polling — removed because `guest-exec` is blocked by default in qemu-ga on most distributions; feature rarely worked out of the box. Users should verify VM readiness via SSH in their playbook.
- Auto-detection of disk slot — removed; `clone_vm_disk_slot` defaults to `scsi0` again. Set explicitly (e.g. `sata0`) if the template uses a non-standard slot.
- Template dependency check downgraded from hard fail to informational debug warning (only linked clones were detectable; full clones are independent).

### Changed
- Disk resize is now opt-in (`clone_vm_disk_size` default empty = template size; `vm_resize_disk` master toggle).
- `clone_vm` refactored: `main.yml` → orchestrator + `clone_one_vm.yml` per-VM flow.
- `create_templates`: refresh cloud image stat after download so import runs for newly downloaded images.
- Cloud images from Yandex mirror (`mirror.yandex.ru`).

### Fixed
- "No space left on device" on first boot (template disk resized to ≥12G).
- Hardware config command spacing (`--scsihw ... --agent ...`).
- Static IP value trimmed (no leading spaces from Jinja).
- Removed recursive self-reference in `cv_vm_index`.
- `ansible-lint` now passes (production profile).

### Security
- Recreate refuses to destroy templates and foreign VMs.
- Template dependency check (linked clones) before recreate in create_templates.
