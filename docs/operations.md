# Операционное руководство и устранение неполадок

Этот раздел описывает предварительные требования, настройки окружения и типичные
«подводные камни» эксплуатации коллекции. Если что-то не работает — смотрите раздел
**Troubleshooting**.

**См. также:** [`README.md`](../README.md) (полный справочник), [`examples.md`](examples.md)
(примеры playbook'ов), [`overrides.md`](overrides.md) (переопределения и новая ОС),
[`ansible.cfg.example`](ansible.cfg.example) (рекомендуемый ansible.cfg).

---

## 1. Предварительные требования (Prerequisites)

### Control node (ваш хост Ansible)

- **Ansible ≥ 2.16.** Проверьте: `ansible --version`.
- **Python 3** с пакетом **`passlib`** — обязателен для рендеринга cloud-init шаблонов,
  потому что в них используется фильтр `password_hash('sha512')`.

  Установка (Ubuntu/Debian):
  ```bash
  sudo apt update && sudo apt install -y python3-passlib
  # или через pip:
  pip install passlib
  ```

  Проверка, что filter доступен на control node:
  ```bash
  ansible localhost -m debug -a "msg={{ 'secret' | password_hash('sha512') }}" \
    --connection=local
  ```

- **`ansible.cfg`** (рекомендуется):
  ```ini
  [defaults]
  host_key_checking = False
  retry_files_enabled = False
  vault_password_file = /home/you/.vault_pass   # опц., если не используете --ask-vault-pass
  forks = 10
  serial = 1                                   # роли сами ставят serial: 1 для clone_vm/create_templates
  ```

### Target node (Proxmox VE)

- **SSH-доступ** пользователем, у которого есть passwordless sudo (или прямой root).
  Ansible подключается по `ansible_user`/`vault_ansible_user`, а роли выполняют команды
  через `sudo qm …` / `pvesm status`.
- Установлены CLI **`qm`** и **`pvesm`**.
- Доступ к storage pool (см. раздел Storage).

---

## 2. Ansible Vault

Чувствительные переменные шифруются через vault:

| Переменная | Что хранится |
|---|---|
| `vault_proxmox_password` | Пароль Proxmox (`root@pam`) |
| `vault_ansible_user` | SSH-пользователь на target node |
| `vault_cloud_password` | Пароль cloud-init пользователя внутри ВМ |

```bash
# Зашифровать vault.yml (если он в репо/локально)
ansible-vault encrypt group_vars/all/vault.yml

# Запуск — один из способов:
ansible-playbook playbooks/create-vm-templates.yml --ask-vault-pass
ansible-playbook playbooks/clone-vm-from-template.yml --ask-vault-pass
```

Или укажите файл с паролем (тогда `--ask-vault-pass` не нужен):
```bash
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook playbooks/clone-vm-from-template.yml
```

---

## 3. SSH-ключ cloud-init пользователя

По умолчанию ключ берётся из `~/.ssh/id_rsa.pub`:
```yaml
ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

Переопределите, если нужно другое расположение/имя ключа:
```yaml
ssh_public_key: "{{ lookup('file', '/home/you/.ssh/mykey.pub') }}"
```

---

## 4. Storage и cloud-init snippets

- **`vm_storage_pool` / `storage_pool`** — storage pool для дисков ВМ.
  Дефолт в ролях консервативный (`local-lvm`) и **должен быть переопределён под вашу установку**.
  На вашем Proxmox это, например, `sd01`.

- **Snippets-хранилище** — `snippets_storage` (дефолт `local`). Хранилище должно поддерживать
  content-type `snippets`, а путь по умолчанию `/var/lib/vz/snippets`.
  Роль проверяет storage active и наличие snippets перед запуском.

---

## 5. Cloud-образы и зеркало

URL образов в `os_templates` (`defaults/main.yml`) берутся из **Yandex mirror**
(`mirror.yandex.ru`), а не из официальных источников.

По мере выхода новых релизов обновляйте пути:

| ОС | Что следить |
|---|---|
| Fedora | номер версии (43 → 45 и т.д.) в `image_url`/`vm_name` |
| AlmaLinux | версию `.qcow2` (9.7 → 9.8) |
| Ubuntu | series (`noble`, `resolute`) |

После скачивания зановго образа роль автоматически пересоздаёт статистiku, чтобы импорт
выполнился для нового файла.

---

## 6. Размер диска

- **`template_min_disk_size: "12G"`** — минимальный размер шаблона при создании.
  Образы из облака часто поставляются с крошечным root без свободного места после него, и тогда
  cloud-init `growpart` не сможет расширить корневой раздел при первом боте → ошибка
  «No space left on device». Ресайз до ≥12G оставляет место в конце диска; growpart затем
  расширяет root. **Не снижайте этот порог.**

- **Resize при клонировании — опционально:**
  - `clone_vm_disk_size: ""` (по умолчанию) → диск = размер шаблона, ресайз не делается.
  - Установите `"20G"` и включите мастер-переключатель `vm_resize_disk: true`, чтобы увеличить диск.
- **Слот диска** (`clone_vm_disk_slot`, по умолчанию пусто → автоопределение). Если переменная
  пуста, слот автоопределяется из конфига шаблона-источника (первый слот со `size=`).
  Если задана явно (например `scsi1`), используется как есть. Fallback при неудаче детекта — `scsi0`.
  Полезно для шаблонов с нестандартным слотом (sata0, virtio0 и т.д.).

---

## 7. Cloud-init snippets и первый запуск

- Файлы snippet'ов создаются как `ci-{vm_id}.yaml` в snippets-пути.
  **Они НЕ удаляются автоматически** — чистите вручную или по cron, если хотите.

- На всех клонах задан флаг **`--ciupgrade 0`** → `package_upgrade` отключён при первом боте
  (для скорости). Обновления пакетов выполняйте позже вручную.
- **Опциональное ожидание cloud-init** (`wait_for_cloud_init: true`): после старта ВМ роль
  поллит `qm guest exec ... cloud-init status` до состояния `done`/`error`. Требует:
  1. qemu-guest-agent в образе (есть во всех поддерживаемых);
  2. **включённую команду `guest-exec` в гостевом агенте**. По умолчанию в некоторых
     дистрибутивах/настройках Proxmox `guest-exec` заблокирован → `qm guest exec` вернёт
     «Command guest-exec has been disabled». В этом случае отключите опцию или разрешите
     `guest-exec` в конфиге гостевого агента (`/etc/qemu/qemu-ga.conf`, уберите `guest-exec`
     из `blacklist`) и перезапустите `qemu-guest-agent`.
  Параметры: `wait_for_cloud_init_retries` (60), `wait_for_cloud_init_delay` (5 с).
  По умолчанию выключено — роль завершается сразу после `qm start`.

---

## 8. Bulk-клонирование (несколько ВМ из одного шаблона)

### Основные переменные

| Переменная | Описание | Дефолт |
|---|---|---|
| `vm_count` | Сколько ВМ клонировать (`1` = backward compat) | `1` |
| `new_vm_ids` | Явный список ID (приоритет над `vm_count`) | `[]` |
| `vm_name_pattern` | Шаблон имени: `{index}` (с паддингом), `{index_raw}` (без) | `""` |
| `vm_name_index_padding` | `0` = без паддинга, `>0` = мин. цифр (авто-масштабируется до ширины count) | `2` |
| `vm_id_min` | Нижняя граница сгенерированных ID (`0` = без ограничения) | `0` |

### Генерация имён и ID

1. Если задан `new_vm_ids` → используем его (длина = кол-во ВМ).
2. Иначе: старт = `new_vm_id`, ID идут подряд `start .. start+N-1`.
3. Имена: при `vm_count=1` — просто `new_vm_name`; иначе `prefix-01 … prefix-NN`.

Примеры паддинга (при `vm_name_pattern: "k8s-{index}"`):

| vm_count | padding | Результат |
|---|---|---|
| 4 | 2 (default) | `k8s-01 .. k8s-04` |
| 15 | 2 | `k8s-01 .. k8s-15` |
| 120 | 2 (авто → 3 цифры) | `k8s-001 .. k8s-120` |
| 5 | 0 | `k8s-1 .. k8s-5` |

### Защита и безопасность ID

- **Коллизии** (при `recreate_vm: false`): если target ID занят — проверяется имя.
  - Имя совпадает → это наша ВМ, пропуск (идемпотентно).
  - Имя другое (чужая ВМ) → **fail**.
- **Защита шаблонов**: если target ID является шаблоном (`template: 1`) — **fail**, даже при
  `recreate_vm: true`. Шаблоны никогда не уничтожаются случайно.
- `recreate_vm: true` уничтожает только ВМ (не шаблоны) и пересоздаёт их.

### Пример bulk-клонирования

```yaml
vars:
  source_vm_id: 3001          # AlmaLinux template
  os_type: "almalinux"
  vm_count: 4                 # создать 4 ВМ
  new_vm_id: 8020             # старт ID → 8020, 8021, 8022, 8023
  vm_name_pattern: "k8s-{index}"   # k8s-01 .. k8s-04 (автопаддинг до ширины count)
```

### Явные ID (`new_vm_ids`)

```yaml
vars:
  new_vm_ids: [8020, 8025, 8030]   # произвольные ID (не обязаны быть подряд)
```

---

### Статические IP для bulk-клонирования

При `vm_count > 1` один `static_ip_address` применится ко **всем** ВМ (конфликт в сети).
Для bulk со статикой используйте список:

```yaml
vars:
  use_dhcp_ip: false
  static_ip_addresses: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
  static_subnet_mask: "24"
  static_gateway: "192.168.1.1"
```

Длина списка должна совпадать с числом клонируемых ВМ, иначе роль завершится ошибкой.

### Безопасность recreate

- `recreate_vm: true` уничтожает ВМ **по ID**. Роль проверяет, что существующее имя
  совпадает с ожидаемым (`new_vm_name`); если ID занят чужой ВМ — запуск завершится ошибкой.
- Шаблоны (`template: 1`) никогда не уничтожаются, даже при `recreate_vm: true`.
- `create_templates` с `recreate_templates: true` уничтожает шаблон только если у него нет
  зависимых ВМ (linked clones, определяемых по `parent:` в конфиге). Полные клоны
  независимы и не блокируют пересоздание.
- Частичный сбой bulk-клонирования оставляет уже созданные ВМ (rollback не выполняется);
  повторный прогон досоздаст недостающие (идемпотентность).

## 9. Идемпотентность и recreate

- Повторный прогон с теми же `new_vm_id`/именами → `changed=0`.
- `recreate_vm: false` по умолчанию — существующая ВМ не трогается (кроме шаблонов).
- `recreate_vm: true` — деструктивно: уничтожает и пересоздаёт указанные ВМ.

---

## 11. Частые ловушки (footguns)

- **`target_node` по умолчанию = `ansible_hostname`** (хост control node). Если не переопределить,
  роль попытается клонировать на несуществующий узел. Всегда задавайте `target_node: "{{ proxmox_node }}"`
  (или имя узла Proxmox).
- **`~/.ssh/id_rsa.pub` должен существовать** на control node — `lookup('file', ...)` падает на этапе
  парсинга playbook'а, до запуска. Создайте ключ или переопределите `ssh_public_key`.
- **`vault_password_file`** должен иметь права `600` (`chmod 600`), иначе секреты скомпрометированы.
- **`full_clone: false`** (linked clone) — клон зависит от шаблона: удаление/пересоздание шаблона
  ломает все linked-клоны. Используйте только если понимаете последствия; по умолчанию `true`.
- **`snippets_path` с trailing slash** — теперь нормализуется в коде, но лучше задавать без слэша.
- **`static_ip_address` формат** — ожидается CIDR-маска в `static_subnet_mask` (например `"24"`),
  не `"255.255.255.0"`. Иначе cloud-init отклонит конфигурацию.
- **Hostname** должен быть RFC 1123 (до 63 символов на label). Генерируемые имена
  валидируются автоматически; ручные — проверяйте сами.
- **Идемпотентность create_templates**: при `recreate_templates: false` (по умолчанию) повторный
  прогон ничего не делает. При `true` шаблон уничтожается и создаётся заново (включая скачивание
  образа, если он не кэширован).

## 10. Troubleshooting

| Симптом | Причина | Решение |
|---|---|---|
| `No space left on device` при cloud-init | Root не расширился — нет свободного места после него на диске шаблона | Увеличьте `template_min_disk_size` (≥12G) и пересоздайте шаблон; проверьте, что `growpart` отрабатывает |
| `VM ID 3004 is a template, not a VM` | Выбрали ID шаблона в качестве `new_vm_id` | Выберите другой `new_vm_id`, не совпадающий с шаблонами (обычно 8000+) |
| `VM ID X already exists with name Y (expected Z)` | Коллизия: target ID занят чужой ВМ (`recreate_vm: false`) | Установите `recreate_vm: true` или выберите другие ID |
| `password_hash filter requires passlib` | На control node нет пакета `passlib` | `apt install python3-passlib` (или `pip install passlib`) на control node |
| Storage не активен / snippets не поддерживаются | Storage pool не active или без content-type `snippets` | Проверьте `pvesm status`; убедитесь, что `snippets_storage` поддерживает snippets |
| Template не найден (`source_vm_id`) | Неправильный ID шаблона или шаблон удалён | Уточните `source_vm_id` и проверьте `qm config {id}` → `template: 1` |
| Timeout импорта образа | Образ большой / медленная сеть | Увеличьте `import_timeout_seconds` (дефолт 1200) |
| `VM ID X is owned by Y (expected Z)` | Попытка recreate чужой ВМ (`recreate_vm: true`) | Выберите другие ID или `recreate_vm: false` |
| `vm_name_pattern must contain {index}` | Паттерн без плейсхолдера при `vm_count > 1` | Добавьте `{index}` или `{index_raw}` в паттерн |
| `static_ip_addresses has N entries but M VMs` | Длина списка IP не совпадает с числом ВМ | Укажите один IP на ВМ или используйте DHCP |
| `Missing required variable(s)` (cloud_password/target_node) | Не задан пароль cloud-init или узел | Задайте `cloud_password` (непустой) и `target_node` |
| `Template X has dependent VMs` | `recreate_templates: true` при наличии linked clones | Удалите/мигрируйте клоны или `recreate_templates: false` |
| `~/.ssh/id_rsa.pub` not found | Файл ключа отсутствует на control node | Создайте ключ или переопределите `ssh_public_key` |
| `Command guest-exec has been disabled` | `wait_for_cloud_init: true`, но guest-exec заблокирован в qemu-ga | Отключите `wait_for_cloud_init` или разрешите `guest-exec` в `/etc/qemu/qemu-ga.conf` |

---

## 11. Безопасность

1. Используйте Ansible Vault для паролей (`vault_*`).
2. Предпочитайте SSH-ключи вместо паролей cloud-init.
3. Регулярно обновляйте образы в `os_templates` (безопасные патчи).
4. Не храните секреты в открытом виде в playbook'ах или репо.
