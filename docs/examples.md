# Пример использования коллекции Proxmox VM

**См. также:** [`README.md`](../README.md), [`operations.md`](operations.md)
(предварительные требования и troubleshooting), [`overrides.md`](overrides.md).

## Установка коллекции

```bash
# Установка коллекции из локального файла
ansible-galaxy collection install /path/to/opscores.proxmox_vm

# Или установка из репозитория
ansible-galaxy collection install git+https://github.com/username/ansible-collection-proxmox-vm.git
```

## Примеры плейбуков

### 1. Создание шаблонов ВМ

```yaml
---
- name: Create Proxmox VM Templates
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    vm_defaults:
      memory: 4096
      cores: 2
      network_bridge: "vmbr0"
    cleanup_downloaded_images: true
  tasks:
    - name: Create Templates
      include_role:
        name: create_templates
```

### 1.1. Переопределение параметров шаблонов

```yaml
---
- name: Create Proxmox VM Templates with custom parameters
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    recreate_templates: true
  tasks:
    - name: Create Templates
      include_role:
        name: create_templates
```

### 2. Клонирование ВМ из шаблона

```yaml
---
- name: Clone VM from Template
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001
    os_type: "almalinux"  # or 'ubuntu24', 'ubuntu26', 'fedora'
    new_vm_id: 8001
    new_vm_name: "web-server-01"
    cloud_user: "ansible"
    cloud_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      66386439653339373563626531313939393730636439383262303931343666643530306664393131
      3339383535613935323036316231393737316531373139340a353930306639343130363633383532
      33353332333364333436303662643333393737323566313035313036643231343131306439383839
      6433373130353436310a666234383734383439373731383732643239313035343434663739343033
      3561
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    target_node: "{{ proxmox_node }}"
    storage_pool: "{{ vm_storage_pool }}"
    full_clone: true
  tasks:
    - name: Clone VM
      include_role:
        name: clone_vm
```

### 2.1. Клонирование ВМ с переопределением параметров

```yaml
---
- name: Clone VM from Template with custom parameters
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3000  # Ubuntu 24.04 template
    os_type: "ubuntu24"  # or 'ubuntu26'
    new_vm_id: 8005
    new_vm_name: "custom-vm-01"
    cloud_user: "ansible"
    cloud_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      66386439653339373563626531313939393730636439383262303931343666643530306664393131
      3339383535613935323036316231393737316531373139340a353930306639343130363633383532
      33353332333364333436303662643333393737323566313035313036643231343131306439383839
      6433373130353436310a666234383734383439373731383732643239313035343434663739343033
      3561
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    target_node: "{{ proxmox_node }}"
    storage_pool: "{{ vm_storage_pool }}"
    full_clone: true

    # Переопределение параметров ВМ
    clone_vm_memory: 8192      # 8GB RAM
    clone_vm_cores: 4          # 4 CPU cores
    clone_vm_disk_size: "20G"  # optional: resize disk (default = template disk size)
    clone_vm_cpu: "host"       # Использовать CPU хоста
  tasks:
    - name: Clone VM
      include_role:
        name: clone_vm
```

### 3. Создание нескольких ВМ из шаблона

```yaml
---
- name: Clone Multiple VMs from Template
  hosts: proxmox
  collections:
    - opscores.proxmox_vm
  vars:
    vm_configs:
      - { source_id: 3001, new_id: 8001, name: "web-server-01", os_type: "almalinux" }
      - { source_id: 3001, new_id: 8002, name: "web-server-02", os_type: "almalinux" }
      - { source_id: 3000, new_id: 8003, name: "db-server-01", os_type: "ubuntu24" }
      - { source_id: 3002, new_id: 8004, name: "app-server-01", os_type: "fedora" }
  tasks:
    - name: Clone VMs in loop
      include_role:
        name: clone_vm
      vars:
        source_vm_id: "{{ item.source_id }}"
        new_vm_id: "{{ item.new_id }}"
        new_vm_name: "{{ item.name }}"
        os_type: "{{ item.os_type }}"
      loop: "{{ vm_configs }}"
```

## Примеры bulk-клонирования (несколько ВМ из одного шаблона)

### 3.1. Bulk-клонирование (`vm_count`) — N ВМ с падающими именами

Создаёт `vm_count` ВМ подряд по ID (`new_vm_id .. new_vm_id + count - 1`) и именам
с автоматическим паддингом:

```yaml
---
- name: Clone Multiple VMs from Template (bulk)
  hosts: proxmox
  serial: 1
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001          # AlmaLinux template
    os_type: "almalinux"        # or 'ubuntu24', 'ubuntu26', 'fedora'
    vm_count: 4                 # создать 4 ВМ
    new_vm_id: 8020             # старт ID -> 8020, 8021, 8022, 8023
    vm_name_pattern: "k8s-{index}"   # k8s-01 .. k8s-04 (автопаддинг до ширины count)
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tasks:
    - name: Clone VMs in bulk
      include_role:
        name: clone_vm
```

Результат: `8020 k8s-01`, `8021 k8s-02`, `8022 k8s-03`, `8023 k8s-04`.

### 3.2. Явные ID (`new_vm_ids`) — произвольные, не обязательно подряд

```yaml
---
- name: Clone VMs with explicit IDs
  hosts: proxmox
  serial: 1
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001
    os_type: "almalinux"
    new_vm_ids: [8020, 8025, 8030]   # произвольные ID (приоритет над vm_count)
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tasks:
    - name: Clone VMs with explicit IDs
      include_role:
        name: clone_vm
```

### 3.3. Без паддинга (`vm_name_index_padding: 0`)

```yaml
---
- name: Clone VMs without padding
  hosts: proxmox
  serial: 1
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001
    os_type: "almalinux"
    vm_count: 5
    new_vm_id: 8020
    vm_name_pattern: "web-{index_raw}"   # web-1 .. web-5 (без паддинга)
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tasks:
    - name: Clone VMs without padding
      include_role:
        name: clone_vm
```

### 3.4. Повторный запуск — идемпотентность

Повторный прогон с теми же `new_vm_id`/именами не создаёт дубликатов (`changed=0`).

### 3.5. Защита ID (коллизии и шаблоны)

- **Коллизия** (при `recreate_vm: false`): если target ID занят чужой ВМ с другим именем —
  запуск завершается ошибкой. Если имя совпадает — это наша ВМ, пропуск (идемпотентно).
- **Защита шаблонов**: если target ID является шаблоном (`template: 1`) — запуск завершается
  ошибкой даже при `recreate_vm: true`. Шаблоны никогда не уничтожаются случайно.

```yaml
---
- name: Clone VM (idempotent, safe against templates)
  hosts: proxmox
  serial: 1
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001
    os_type: "almalinux"
    new_vm_id: 8020
    new_vm_name: "k8s-01"
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tasks:
    - name: Clone VM
      include_role:
        name: clone_vm
```

> См. [docs/operations.md](docs/operations.md) для полного описания bulk-клонирования и
> устранения неполадок.

### 3.6. Bulk-клонирование со статическими IP

```yaml
---
- name: Clone VMs with per-VM static IPs
  hosts: proxmox
  serial: 1
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001
    os_type: "almalinux"
    vm_count: 3
    new_vm_id: 8020
    use_dhcp_ip: false
    static_ip_addresses: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
    static_subnet_mask: "24"
    static_gateway: "192.168.1.1"
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tasks:
    - name: Clone VMs with static IPs
      include_role:
        name: clone_vm
```

### 3.7. Ожидание завершения cloud-init

```yaml
---
- name: Clone VM and wait for cloud-init
  hosts: proxmox
  serial: 1
  collections:
    - opscores.proxmox_vm
  vars:
    source_vm_id: 3001
    os_type: "almalinux"
    new_vm_id: 8020
    new_vm_name: "wait-test"
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    wait_for_cloud_init: true
    wait_for_cloud_init_retries: 60
    wait_for_cloud_init_delay: 5
  tasks:
    - name: Clone VM and wait
      include_role:
        name: clone_vm
```

> Требует qemu-guest-agent в образе.

## Примеры команд запуска

```bash
# Запуск плейбука для создания шаблонов
ansible-playbook -i inventory playbooks/create-vm-templates.yml

# Запуск плейбука для клонирования ВМ
ansible-playbook -i inventory playbooks/clone-vm-from-template.yml

# Запуск с указанием конкретных переменных
ansible-playbook -i inventory playbooks/clone-vm-from-template.yml --extra-vars "new_vm_name=test-vm new_vm_id=105"
```

## Структура коллекции

```
opscores.proxmox_vm/
├── docs/
│   ├── README.md
│   ├── overrides.md
│   ├── examples.md
│   └── operations.md          # prerequisites, bulk clone, troubleshooting
├── playbooks/
│   ├── create-vm-templates.yml
│   ├── clone-vm-from-template.yml
│   ├── clone-vm-from-template-dhcp.yml
│   └── clone-vm-from-template-static.yml
├── roles/
│   ├── create_templates/
│   └── clone_vm/
├── plugins/
├── vars/
├── defaults/
├── meta/
└── tests/
```