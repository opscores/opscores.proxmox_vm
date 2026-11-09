# Переопределение параметров

## Переопределение через inventory

Ansible поддерживает глубокое слияние словарей (deep merge). Определите `os_templates` в `group_vars` или `host_vars` для переопределения параметров конкретных ОС.

### Отключить создание шаблона Fedora

```yaml
# group_vars/proxmox.yml
os_templates:
  fedora:
    enabled: false
```

### Своё зеркало для Ubuntu 24

```yaml
# group_vars/proxmox.yml
os_templates:
  ubuntu24:
    image_url: "https://internal-mirror.corp/noble-server-cloudimg-amd64.img"
```

### Изменить template_id

```yaml
# group_vars/proxmox.yml
os_templates:
  almalinux:
    template_id: 9001
```

### Переопределить параметры VM для всех ОС

```yaml
# group_vars/proxmox.yml
vm_defaults:
  memory: 2048
  cores: 2
  disk_size: "20G"
  network_bridge: "vmbr1"
```

### Переопределить параметры VM для конкретной ОС

```yaml
# group_vars/proxmox.yml
os_templates:
  ubuntu24:
    memory: 4096
    cores: 4
```

## Per-OS recreate и cleanup

По умолчанию флаги `recreate_templates` и `cleanup_downloaded_images` применяются глобально ко всем ОС. Можно переопределить для конкретной ОС:

### Включить recreate только для Fedora

```yaml
# group_vars/proxmox.yml
recreate_templates: false  # глобально выключено

os_templates:
  fedora:
    recreate: true  # только Fedora будет пересоздаваться
```

### Включить cleanup только для Ubuntu 24

```yaml
# group_vars/proxmox.yml
cleanup_downloaded_images: false  # глобально выключено

os_templates:
  ubuntu24:
    cleanup_image: true  # только для Ubuntu 24 удалять образ после импорта
```

### Полный пример с per-OS настройками

```yaml
# group_vars/proxmox.yml
recreate_templates: false
cleanup_downloaded_images: false

os_templates:
  ubuntu24:
    enabled: true
    template_id: 3000
    recreate: false
    cleanup_image: true
  ubuntu26:
    enabled: true
    template_id: 3004
    recreate: false
    cleanup_image: false
  almalinux:
    enabled: true
    template_id: 3001
    recreate: true      # пересоздавать при каждом запуске
    cleanup_image: true  # удалять образ после импорта
  fedora:
    enabled: true
    template_id: 3002
    recreate: false
    cleanup_image: false
```

## Добавление новой ОС

1. Добавьте запись в `os_templates` в `defaults/main.yml`
2. Создайте файл `cloud-init-<os_name>.yaml.j2` в `clone_vm/templates/`
3. Готово — задачи обнаружат новую ОС автоматически

### Пример: Debian 12

```yaml
# defaults/main.yml → os_templates
os_templates:
  # ... существующие ОС ...
  debian12:
    template_id: 3005
    vm_name: "debian-12-template"
    image_url: "https://mirror.yandex.ru/debian-cloud-images/bookworm/current/amd64/archive/debian-12-genericcloud-amd64.qcow2"
    image_filename: "debian-12-genericcloud-amd64.qcow2"
    enabled: true
    cloud_init_template: debian12
    recreate: false
    cleanup_image: false
```

```yaml
# clone_vm/templates/cloud-init-debian12.yaml.j2
#cloud-config
# Debian 12-specific cloud-init configuration

hostname: "{{ new_vm_name }}"
manage_etc_hosts: false
preserve_hostname: false

timezone: "{{ timezone | default('Europe/Moscow') }}"
locale: en_US.UTF-8

resize_rootfs: true
growpart:
  mode: auto
  devices: ['/']

package_update: false
package_upgrade: false
packages:
  - qemu-guest-agent
  - curl
  - wget
  - htop
  - net-tools
  - sudo
  - openssh-server

users:
  - name: {{ cloud_user }}
    primary_group: {{ cloud_user }}
    groups: [sudo, adm, systemd-journal]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
    lock_passwd: false
    passwd: "{{ cloud_password | password_hash('sha512') }}"
    ssh_authorized_keys:
      - {{ ssh_public_key }}

ssh_pwauth: true
disable_root: true

bootcmd:
  - [systemd-machine-id-setup]

runcmd:
  - [hostnamectl, set-hostname, "{{ new_vm_name }}"]
  - [timedatectl, set-timezone, "{{ timezone | default('Europe/Moscow') }}"]
  - [systemctl, enable, qemu-guest-agent]
  - [systemctl, start, qemu-guest-agent]
  - [sed, -i, 's/^#?PubkeyAuthentication.*/PubkeyAuthentication yes/', /etc/ssh/sshd_config]
  - [sed, -i, 's/#PermitRootLogin.*/PermitRootLogin no/', /etc/ssh/sshd_config]
  - [systemctl, restart, sshd]
  - [chown, -R, '{{ cloud_user }}:{{ cloud_user }}', '/home/{{ cloud_user }}/.ssh']
  - ['chmod', '700', '/home/{{ cloud_user }}/.ssh']
  - ['chmod', '600', '/home/{{ cloud_user }}/.ssh/authorized_keys']

final_message: "Debian 12 cloud-init completed at $UPTIME"
```

## Клонирование по имени ОС

Для клонирования по имени ОС (а не по VMID) используйте `os_templates` из inventory:

```yaml
# playbooks/clone_ubuntu24.yml
- hosts: proxmox
  vars:
    src_vm_id: "{{ os_templates['ubuntu24']['template_id'] | default(3000) }}"
    os_type: "ubuntu24"
  roles:
    - clone_vm
```

## Обязательные переменные clone_vm

При использовании роли `clone_vm` необходимо определить следующие переменные в playbook или inventory:

### Параметры VM (обязательные)
| Переменная | Описание | Пример |
|---|---|---|
| `source_vm_id` | ID шаблона-источника | `3000` |
| `new_vm_id` | ID новой VM | `8001` |
| `new_vm_name` | Hostname новой VM | `"test-vm"` |
| `os_type` | Тип ОС (должен совпадать с cloud-init шаблоном) | `"ubuntu24"` |
| `target_node` | Целевой Proxmox узел | `"pve-node"` |
| `storage_pool` | Хранилище для диска | `"local-lvm"` |
| `full_clone` | Полное клонирование | `true` |

### Cloud-init параметры (обязательные)
| Переменная | Описание | Пример |
|---|---|---|
| `cloud_user` | Имя пользователя cloud-init | `"ansible"` |
| `cloud_password` | Пароль (используйте ansible-vault!) | `"{{ vault_cloud_password }}"` |
| `ssh_public_key` | SSH публичный ключ | `"{{ lookup('file', '~/.ssh/id_rsa.pub') }}"` |
| `timezone` | Часовой пояс | `"Europe/Moscow"` |

### Пример playbook

```yaml
# playbooks/clone_vm.yml
- hosts: proxmox
  vars:
    source_vm_id: 3000
    new_vm_id: 8001
    new_vm_name: "test-ubuntu24"
    os_type: "ubuntu24"
    target_node: "pve-node"
    storage_pool: "local-lvm"
    full_clone: true
    cloud_user: "ansible"
    cloud_password: "{{ vault_cloud_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  roles:
    - clone_vm
```

**Важно:** `cloud_password` должен быть зашифрован через `ansible-vault`. Никогда не храните пароли в открытом виде.
