<div align="center">

# 🧊 Proxmox Cheat Sheet — шпаргалка по Proxmox VE

**Шпаргалка по Proxmox VE на русском:** теория простым языком (KVM, LXC, pmxcfs, кластер, quorum), управление виртуальными машинами (`qm`) и контейнерами (`pct`), хранилища, снапшоты, бэкапы и PBS, кластер и HA, сеть, файрвол, пользователи и API, ZFS и Ceph — с примерами. Продвинутый уровень: passthrough, hooks, SDN, диагностика. И практикум: пошаговые сценарии и задачи с решениями.

![Proxmox](https://img.shields.io/badge/OS-Proxmox_VE_8/9-E57000?logo=proxmox&logoColor=white)
![Bash](https://img.shields.io/badge/shell-bash-4EAA25?logo=gnubash&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

</div>

---

**Для кого:** начинающие пользователи Proxmox, homelab-энтузиасты, системные администраторы, DevOps/SRE-инженеры, все, кто переезжает с VMware/Hyper-V. Актуально для **Proxmox VE 8.x и 9.x**.

### Легенда

| Значок | Значение |
|:---:|---|
| 🟢 | База — нужно знать каждому |
| 🟡 | Продвинутый уровень |
| 🔴 | Опасно: может удалить данные или сломать систему |
| 💡 | Совет / лайфхак |

---

## 📚 Содержание

0. [🧠 Теория: как устроен Proxmox VE](#-0-теория-как-устроен-proxmox-ve)
1. [Первые шаги и информация о системе](#-1-первые-шаги-и-информация-о-системе)
2. [Виртуальные машины: qm](#-2-виртуальные-машины-qm)
3. [Контейнеры LXC: pct и pveam](#-3-контейнеры-lxc-pct-и-pveam)
4. [Диски, образы и импорт](#-4-диски-образы-и-импорт)
5. [Снапшоты и клонирование](#-5-снапшоты-и-клонирование)
6. [Хранилища: pvesm](#-6-хранилища-pvesm)
7. [Бэкапы: vzdump и восстановление](#-7-бэкапы-vzdump-и-восстановление)
8. [Proxmox Backup Server](#-8-proxmox-backup-server)
9. [Cloud-Init](#-9-cloud-init)
10. [Кластер: pvecm и quorum](#-10-кластер-pvecm-и-quorum)
11. [Миграция VM и контейнеров](#-11-миграция-vm-и-контейнеров)
12. [Высокая доступность: ha-manager](#-12-высокая-доступность-ha-manager)
13. [Сеть: мосты, bond, SDN](#-13-сеть-мосты-bond-sdn)
14. [Файрвол: pve-firewall](#-14-файрвол-pve-firewall)
15. [Пользователи, ACL и API-токены: pveum](#-15-пользователи-acl-и-api-токены-pveum)
16. [API и pvesh](#-16-api-и-pvesh)
17. [ZFS на узле Proxmox](#-17-zfs-на-узле-proxmox)
18. [Ceph (кратко)](#-18-ceph-кратко)
19. [Логи и задачи](#-19-логи-и-задачи)
20. [Обслуживание узла и обновления](#-20-обслуживание-узла-и-обновления)
21. [🟡 Для профи: passthrough, hooks, тонкая настройка](#-21-для-профи-passthrough-hooks-тонкая-настройка)
22. [🔴 Опасные команды](#️-22-опасные-команды-)
23. [🩺 Диагностика: что делать, если…](#-23-диагностика-что-делать-если)
24. [🎯 Вопросы с собеседований](#-24-вопросы-с-собеседований)
25. [🧰 Практические сценарии](#-25-практические-сценарии)
26. [🎓 Практикум: задачи с решениями](#-26-практикум-задачи-с-решениями)
27. [Глоссарий](#-глоссарий)
28. [Полезные ресурсы](#-полезные-ресурсы)

---

## 🧠 0. Теория: как устроен Proxmox VE

> Команды забываются, а понимание остаётся. Proxmox — это не «ещё один гипервизор», а Debian + KVM/QEMU + LXC, склеенные веб-интерфейсом и кластерной ФС.

### 0.1. Из чего состоит

```
┌──────────────────────────────────────────────┐
│  Веб-интерфейс (:8006) / pvesh / API          │
├──────────────────────────────────────────────┤
│  Утилиты: qm (VM), pct (LXC), pvesm, vzdump, │
│           pvecm, ha-manager, pveum, pvesh    │
├──────────────────────────────────────────────┤
│  KVM/QEMU (полная виртуализация VM)           │
│  LXC (контейнеры, общее ядро)                 │
├──────────────────────────────────────────────┤
│  pmxcfs — кластерная ФС конфигурации          │
├──────────────────────────────────────────────┤
│  Debian + ядро Linux                          │
└──────────────────────────────────────────────┘
```

| Термин | Что это |
|---|---|
| **PVE** | Proxmox Virtual Environment — сам продукт |
| **Узел (node)** | Физический сервер с установленным PVE |
| **VMID / CTID** | Числовой ID виртуальной машины или контейнера (100–999999999) |
| **KVM/QEMU** | Полная виртуализация: у каждой VM своё ядро, можно Windows/BSD |
| **LXC** | Контейнеры: общее ядро хоста, быстрее и «легче», но только Linux |
| **pmxcfs** | Кластерная ФС (`/etc/pve`), хранит конфиги, реплицируется между узлами через corosync |
| **Corosync** | Синхронизация конфигурации и quorum в кластере |
| **PBS** | Proxmox Backup Server — отдельный продукт для дедуплицированных бэкапов |

### 0.2. VM или LXC — что выбрать

| Критерий | VM (KVM) | LXC-контейнер |
|---|---|---|
| Ядро | Своё | Общее с хостом |
| ОС | Любая (Windows, BSD…) | Только Linux |
| Накладные расходы | Выше | Минимальные |
| Изоляция | Аппаратная | Ограниченная (общее ядро!) |
| Старт | Секунды–десятки секунд | Почти мгновенный |
| Когда выбирать | Windows, проприетарное ПО, максимальная изоляция | Linux-сервисы, высокий density |

> 💡 Правило: «если можно в LXC — делай LXC, если сомневаешься — VM». Контейнер с root-доступом общается с ядром хоста — для недоверенного кода используй VM.

### 0.3. Конфигурация: /etc/pve

Все конфиги — в `/etc/pve`, это не обычная папка, а **pmxcfs**:

| Путь | Что лежит |
|---|---|
| `/etc/pve/qemu-server/<VMID>.conf` | Конфиг VM |
| `/etc/pve/lxc/<CTID>.conf` | Конфиг контейнера |
| `/etc/pve/storage.cfg` | Хранилища |
| `/etc/pve/corosync.conf` | Кластер |
| `/etc/pve/user.cfg` | Пользователи и права |
| `/etc/pve/firewall/*.fw` | Правила файрвола |
| `/etc/pve/priv/` | Ключи, токены, пароли |

> 🔴 `/etc/pve` меняется утилитами (`qm set`, `pct set`) или напрямую файлом, но **не редактируй** его на неработающем кластере без понимания: pmxcfs хранит себя в `/var/lib/pve-cluster/config.db`, и полная потеря этого файла = потеря всех конфигов VM. Регулярно бэкапь: скопируй `/var/lib/pve-cluster/config.db` и `/etc/pve/*.cfg`.

### 0.4. Веб-интерфейс

- `https://IP-узла:8006` — вход root (или PAM) / любой realm из user.cfg
- Пункты: **Datacenter** (кластер, storage, backups, HA, файрвол, SDN) → **Node** (сеть, диски, обновления) → **VM/CT**
- 💡 Консоль: noVNC работает из браузера; `qm terminal <VMID>` — последовательная консоль (нужен guest-агент или serial-конфиг)

---

## 🟢 1. Первые шаги и информация о системе

```bash
pveversion -v                 # версия PVE и всех пакетов
pveperf                        # быстрый бенчмарк диска /var/lib/vz
pvecm status                   # статус кластера (даже если узел один)
hostname -f                    # FQDN узла — важно для кластера
ip a                          # IP-адреса
df -h                         # место
free -h                       # память
lscpu | grep -E 'Model name|NUMA|Socket|Core'   # CPU
cat /proc/cpuinfo | grep -c processor            # потоков
```

### Репозитории пакетов

```bash
# Посмотреть подключённые репозитории PVE
ls /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/*.sources 2>/dev/null

# Для бесплатного использования: pve-no-subscription вместо enterprise
# (PVE 8+: файлы .sources с компонентом pve-no-subscription)
apt update
apt dist-upgrade              # обновление узла PVE — только так
```

> 💡 Enterprise-репозиторий требует платную подписку: без неё `apt update` будет ругаться на 401. Для homelab переключись на `pve-no-subscription` (и `pve-enterprise` для Ceph — на `ceph-no-subscription`).

---

## 🟢 2. Виртуальные машины: qm

`qm` — универсальный инструмент управления VM. Почти всё, что делает веб-интерфейс, делает `qm`.

### Жизненный цикл

| Команда | Что делает |
|---|---|
| `qm list` | Список всех VM |
| `qm status 100` | Статус VM 100 (добавь `--verbose` для деталей) |
| `qm config 100` | Конфигурация VM |
| `qm start 100` / `qm stop 100` | Запустить / выключить (как выдернуть питание) |
| `qm shutdown 100` | Корректное выключение (через guest-агент или ACPI) |
| `qm reboot 100` / `qm reset 100` | Перезагрузка / жёсткий сброс 🔴 |
| `qm suspend 100` / `qm resume 100` | Пауза с сохранением в память |
| `qm destroy 100` | 🔴 Удалить VM; по умолчанию удаляются и её диски, на которые никто больше не ссылается (`--destroy-unreferenced-disks 0` — оставить диски; `--purge` — вычистить и из HA/бэкапов конфигов) |
| `qm pending 100` | Показать применённые и ожидающие применения изменения |

### Создание VM из CLI

```bash
qm create 100 --name web-01 --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:vm-100-disk-0,size=20G \
  --ostype l26 --agent enabled=1
# l26 = Linux 2.6+/3.x/4.x+ (general Linux)
qm set 100 --ide2 local:iso/debian-12.iso,media=cdrom   # прицепить ISO
qm start 100
```

### Изменение на лету

```bash
qm set 100 --memory 4096          # память (уменьшение = ballooning, нужен агент)
qm set 100 --cores 4              # CPU
qm set 100 --onboot 1             # автозапуск при загрузке узла
qm set 100 --boot order=scsi0     # порядок загрузки
qm set 100 --delete net1          # удалить параметр из конфига
qm set 100 --description "Сервер сайта"
```

### QEMU Guest Agent

```bash
qm set 100 --agent enabled=1     # включить в конфиге VM
# Внутри гостя:
apt install qemu-guest-agent && systemctl enable --now qemu-guest-agent
qm agent 100 ping                # проверить связь с агентом
qm agent 100 get-osinfo          # IP-адреса, ОС гостя (видны в веб-интерфейсе!)
qm agent 100 fsfreeze-freeze     # заморозить ФС (используется при бэкапе)
```

> 💡 Всегда ставь guest-агент: без него корректный shutdown из веб-интерфейса часто не работает, а IP-адрес гостя не виден. Для Windows — virtio-win-guest-tools.

### QEMU Monitor

```bash
qm monitor 100        # интерактивный монитор QEMU (info block, info status…)
qm status 100 --verbose | grep -E 'memory|cpus|maxmem'
```

---

## 🟢 3. Контейнеры LXC: pct и pveam

### Шаблоны

```bash
pveam update                          # обновить список шаблонов
pveam available --section system      # что можно скачать
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
pveam list local                      # скачанные шаблоны
```

### Жизненный цикл

| Команда | Что делает |
|---|---|
| `pct list` | Список контейнеров |
| `pct status 101` / `pct config 101` | Статус / конфиг |
| `pct create 101 local:vztmpl/debian-12-*.tar.zst --hostname web-lxc --memory 512 --rootfs local-lvm:8 --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1` | Создать контейнер |
| `pct start 101` / `pct stop 101` / `pct shutdown 101` / `pct reboot 101` | Управление питанием |
| `pct enter 101` | Заходить внутрь контейнера как root (без пароля!) |
| `pct exec 101 -- apt update` | Выполнить команду внутри |
| `pct set 101 --memory 1024` | Изменить конфиг |
| `pct destroy 101` | 🔴 Удалить контейнер |
| `pct push 101 file.txt /tmp/` / `pct pull 101 /etc/hostname .` | Копировать файлы туда/обратно |

> 💡 `--unprivileged 1` — по умолчанию и почти всегда правильно: UID маппится (root в CT = 100000 на хосте). Privileged-контейнер с root внутри = почти root на хосте, используй только когда объёмный доступ к устройствам действительно нужен.

### Полезные опции create

```bash
pct create 102 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname app \
  --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:10 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.50/24,gw=192.168.1.1 \
  --unprivileged 1 \
  --features nesting=1 \      # nesting=1 нужен для Docker внутри LXC
  --onboot 1 --start 1
```

> 💡 Docker в LXC: нужен `--features nesting=1` (а для некоторых случаев `keyctl=1`). Но помни: docker в непривилегированном LXC работает, в privileged — риск безопасности выше.

---

## 🟢 4. Диски, образы и импорт

### Управление дисками VM

```bash
qm set 100 --scsi1 local-lvm:20          # добавить диск 20G
qm resize 100 scsi0 +10G                # увеличить диск (гостевая ФС расширяется отдельно)
qm move-disk 100 scsi0 storage-nvme --delete   # переместить диск на другое хранилище
qm unlink 100 --idlist scsi1             # отвязать диск без удаления
qm importdisk 100 ubuntu.qcow2 local-lvm     # импорт образа (PVE 8.x; в PVE 9 — qm disk import)
```

### qemu-img — работа с образами

```bash
qemu-img info disk.qcow2                  # размер, формат, снапшоты внутри
qemu-img convert -O raw src.vmdk dst.raw  # конвертировать формат
qemu-img create -f qcow2 data.qcow2 50G   # пустой образ
```

### Монтирование диска VM на хосте (осторожно)

```bash
# LVM-том VM: vg = pve, имя тома видно в qm config
# 🔴 НИКОГДА не монтируй диск работающей VM — повреждение ФС
qm shutdown 100
lvs                       # найти vm-100-disk-0
mount /dev/pve/vm-100-disk-0 /mnt/vm-disk   # 🔴 только у остановленной VM
# ... работы ...
umount /mnt/vm-disk && qm start 100
```

---

## 🟢 5. Снапшоты и клонирование

```bash
# VM
qm snapshot 100 before-upgrade --description "перед апгрейдом БД"
qm listsnapshot 100
qm rollback 100 before-upgrade        # 🔴 откат: всё после снапшота ПОТЕРЯЕТСЯ
qm delsnapshot 100 before-upgrade
qm snapshot 100 snap1 --vmstate 1      # 🟡 сохранить и память (пауза)

# LXC
pct snapshot 101 before-upgrade
pct rollback 101 before-upgrade
pct listsnapshot 101
```

### Клонирование

```bash
qm clone 100 200 --name web-02 --full 1 --storage local-lvm   # полная копия
qm clone 100 201 --linked 1        # 🟡 linked clone: быстрые, но привязаны к снапшоту-родителю
pct clone 101 202
```

> 💡 Linked clone занимает секунды и почти не ест место, но удалить родительский снапшот нельзя, пока жив клоны. Для продакшена — full clone или шаблоны (правый клик → Convert to template; `qm template 100`).

---

## 🟢 6. Хранилища: pvesm

| Тип | Что даёт | Кому |
|---|---|---|
| `dir` | Каталог на диске | Просто и понятно |
| `lvmthin` | LVM-thin (по умолчанию `local-lvm`) | VM-диски, снапшоты |
| `zfspool` | Пул ZFS | Снапшоты, надёжность |
| `nfs` / `cifs` | Сетевые | ISO, бэкапы, общие диски |
| `pbs` | Proxmox Backup Server | Дедуплицированные бэкапы |
| `rbd` | Ceph | Кластер с общими дисками |

```bash
pvesm status                         # все хранилища и занятость
pvesm list local-lvm                 # содержимое хранилища
pvesm list local --content iso       # только ISO
pvesm add dir mydata --path /mnt/data --content images
pvesm add nfs backups --server 192.168.1.10 --export /share/pve --content backup
pvesm remove mydata
pvesm set local-lvm --content images,rootdir   # поменять типы контента
```

> 💡 `--content`: `images` (диски VM), `rootdir` (диски LXC), `iso`, `vztmpl`, `backup`. Хранилище видно в Datacenter → Storage в веб-интерфейсе.

---

## 🟢 7. Бэкапы: vzdump и восстановление

```bash
# Бэкап VM/CT в файл .vma/.tar на хранилище
vzdump 100 --mode snapshot --compress zstd --storage local
vzdump 100 101 102 --storage backups          # несколько сразу
vzdump --all --exclude 999 --storage backups  # все, кроме 999

# Режимы:
#   snapshot — на живой системе (FS-freeze через агент), рекомендуется
#   suspend  — пауза на время (для корректности БД без агента)
#   stop     — полное выключение, максимальная консистентность

ls -lh /var/lib/vz/dump/               # где лежат бэкапы (storage local)

# Восстановление
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2026_10_31-12_00_00.vma.zst 100 --force  # 🔴 перезапишет VM 100
qmrestore vzdump-qemu-100-*.vma.zst 300           # восстановить как новую VM 300
pct restore 101 /var/lib/vz/dump/vzdump-lxc-101-*.tar.zst --storage local-lvm
```

### Тест восстановления (обязательно!)

```bash
# Бэкап, который не восстанавливали, — это не бэкап
qmrestore vzdump-qemu-100-*.vma.zst 999 --unique
qm start 999 && qm agent 999 ping     # проверить загрузку и сервисы
qm destroy 999                        # убрать тестовую копию
```

> 💡 Планировщик бэкапов: Datacenter → Backup или файл `/etc/pve/jobs.cfg`. Ротация — параметр `prune-backups` (PVE 8+) или `keep-all/keep-last`.

---

## 🟡 8. Proxmox Backup Server

PBS — отдельный сервер (свой ISO или репозиторий на любом Debian) с дедупликацией: одинаковые блоки хранятся один раз, инкрементальные бэкапы каждый раз «полные» логически.

```bash
# На узле PVE: добавить PBS как хранилище
pvesm add pbs pbs1 --server 192.168.1.20 --datastore main \
  --username root@pam --password 'пароль' --fingerprint <SHA256-отпечаток>

# Логи, статус
pvesm status pbs1
proxmox-backup-client status --repository root@pam@192.168.1.20:main   # 🟡 с клиента

# Проверка бэкапа на PBS (на самом PBS)
proxmox-backup-client verify main/vm/100/2026-10-31T12:00:00Z
```

> 💡 Логика «полных» бэкапов при дедупликации: каждое задание выглядит полной копией, но по сети и на диск уходят только изменённые чанки. GC на PBS (`proxmox-backup-manager garbage-collection start`) освобождает место от удалённых бэкапов.

---

## 🟢 9. Cloud-Init

Cloud-Init создаёт VM из «чистого» образа (например, Ubuntu cloud image) с уже заданными пользователем, ключом и IP.

```bash
# 1. Скачать cloud-образ и импортировать
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qm create 150 --name ci-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 150 noble-server-cloudimg-amd64.img local-lvm
qm set 150 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-150-disk-0
qm set 150 --ide2 local-lvm:cloudinit            # cloudinit-диск
qm set 150 --boot order=scsi0 --serial0 socket --vga serial0
qm set 150 --ciuser admin --sshkeys ~/.ssh/id_ed25519.pub
qm set 150 --ipconfig0 ip=192.168.1.60/24,gw=192.168.1.1
qm template 150                                 # превратить в шаблон

# 2. Штамповать VM из шаблона
qm clone 150 151 --name web-ci
qm resize 151 scsi0 +10G
qm start 151 && ssh admin@192.168.1.60
```

> 💡 `qm cloudinit 151 update` — перегенерировать cloud-init диск после изменений параметров (гостю нужен cloud-init внутри).

---

## 🟡 10. Кластер: pvecm и quorum

> Кластер = общая pmxcfs + миграция + HA. Минимум для «честного» кластера — 3 узла (или 2 + QDevice), иначе при падении одного узла не будет quorum и оставшийся «замрёт».

```bash
# Узел 1 — создать кластер
pvecm create mycluster

# Узлы 2+ — присоединиться (нужен root-SSH между узлами)
pvecm add IP-первого-узла

# Информация
pvecm status            # quorum, узлы, ID
pvecm nodes             # список узлов
pvecm expected 1        # 🔴 ЭКСТРЕННО: остаться без кластера (только для диагностики!)
```

### Quorum и QDevice

| Узлов | Выживание при отказе |
|:---:|---|
| 1 | — |
| 2 | ❌ нет quorum при отказе любого |
| 2 + QDevice | ✅ |
| 3 | ✅ отказ 1 узла |

> 💡 Никогда не добавляй в кластер узлы с одинаковыми именами и не разноси узлы по WAN: corosync хочет низкие задержки (<2–5 мс). Для кластера на 2 узла ставь [QDevice](https://pve.proxmox.com/wiki/Cluster_Manager#_qdevice_installation) на третьей машине.

---

## 🟢 11. Миграция VM и контейнеров

```bash
qm migrate 100 node2                # онлайн-миграция (общее хранилище или --with-local-disks)
qm migrate 100 node2 --with-local-disks --targetstorage local-lvm
qm migrate 100 node2 --online 0    # офлайн-миграция (VM выключена)
pct migrate 101 node2              # контейнеры мигрируют только офлайн (--restart для запуска после)
```

> 💡 Онлайн-миграция VM с локальным диском работает (NBD-стриминг), но дольше. VM с общим хранилищем (Ceph/NFS) мигрируются за секунды.

---

## 🟡 12. Высокая доступность: ha-manager

> HA = автоматический перезапуск VM/CT на другом узле кластера при падении узла. Требует кластера с quorum и общего хранилища (иначе данные не увидит второй узел).

```bash
ha-manager status                       # обзор HA-ресурсов
ha-manager add vm:100 --max_restart 3   # добавить VM под HA
ha-manager add ct:101
ha-manager remove vm:100                # снять с HA (VM остаётся)
ha-manager migrate vm:100 node2         # ручной переезд с HA-параметрами
ha-manager relocate vm:100 node2        # 🟡 relocate без учёта состояния (техобслуживание)
# Плановый вывод узла: Maintenance mode в GUI (PVE 8.2+) или ручная миграция ha-manager migrate
```

> 🔴 Если узлов меньше трёх и нет QDevice, при падении узла кластер потеряет quorum, и HA **не сработает** на оставшихся узлах. Quorum — фундамент HA.

---

## 🟢 13. Сеть: мосты, bond, SDN

### Мост vmbr0 — основа

```bash
cat /etc/network/interfaces
```

```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.5/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

```bash
ifreload -a          # применить изменения сети без перезагрузки (пакет ifupdown2)
ip -br link          # обзор интерфейсов
bridge link          # кто в мосте
```

> 🔴 Меняя сеть узла по SSH, держи открытым `tmux` и не выходи, пока не проверишь `ifreload -a` и доступность. Ошибка в мосте = потерянный узел.

### Bond (объединение NIC)

```
auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2
    bond-miimon 100
    bond-mode 802.3ad        # LACP — нужен коммутатор; active-backup проще

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.5/24
    gateway 192.168.1.1
    bridge-ports bond0
```

### VLAN и прочее

```bash
qm set 100 --net0 virtio,bridge=vmbr0,tag=10            # VLAN 10 у VM
qm set 100 --net0 virtio,bridge=vmbr0,trunks=10;20;30     # 🟡 транк: список VLAN в гостя
```

> 🟡 **SDN** (PVE 8+): зоны VLAN/VXLAN/QinQ в Datacenter → SDN — отдельные виртуальные сети с тегами для групп VM/CT. Настройка через веб-интерфейс, конфиги в `/etc/pve/sdn/`.

---

## 🟢 14. Файрвол: pve-firewall

Три уровня правил: **Datacenter** → **Node** → **VM/CT**. По умолчанию выключен на всех уровнях — включение на узле без правила для SSH = потеря доступа.

```bash
pve-firewall status        # статус (показывает и MAC/IP-filter)
pve-firewall log           # смотреть лог в CLI
pve-firewall reload        # перечитать конфиги
cat /etc/pve/firewall/cluster.fw    # правила уровня datacenter
cat /etc/pve/nodes/pve1/host.fw     # узла
cat /etc/pve/firewall/100.fw        # VM 100
```

Пример `/etc/pve/firewall/100.fw`:

```
[OPTIONS]
# enable: 1

[RULES]
IN SSH(ACCEPT) -source 192.168.1.0/24
IN ACCEPT -proto tcp -dport 443
```

> 💡 Сначала правило для SSH/интерфейса, потом `enable: 1`. IP-filter у контейнеров по умолчанию запрещает spoofing — не отключай без причины.

---

## 🟢 15. Пользователи, ACL и API-токены: pveum

```bash
pveum user add ivan@pam --password      # пользователь (realm: pam — системный, pve — свой)
pveum user list
pveum group add ops --comment "Ops team"
pveum user modify ivan@pam --groups ops

# Роли и ACL
pveum acl modify /vms/100 --users ivan@pam --roles PVEVMUser
pveum acl modify / --groups ops --roles PVEVMAdmin   # 🟡 осторожно с корнем "/"
pveum role list                        # встроенные роли: PVEAdmin, PVEVMUser, Auditor…

# API-токены (для скриптов/CI — вместо логина-пароля)
pveum user token add ivan@pam ci-bot --privsep 0 --expire 0   # выведет secret — сохрани!
```

> 💡 Токен = отдельный «пароль» с собственными правами: `ivan@pam!ci-bot`. `privsep 0` — токен наследует права владельца, `1` — свои через ACL.

---

## 🟡 16. API и pvesh

`pvesh` — обёртка над REST API: всё, что умеет веб-интерфейс.

```bash
pvesh get /version
pvesh get /nodes
pvesh get /nodes/pve1/qemu            # список VM узла
pvesh get /nodes/pve1/qemu/100/status/current
pvesh create /nodes/pve1/qemu/100/status/start
pvesh get /cluster/resources          # все ресурсы кластера (VM, CT, storage, node)

# REST API напрямую (токен):
curl -k -H "Authorization: PVEAPIToken=ivan@pam!ci-bot=<secret>" \
  https://pve1:8006/api2/json/cluster/resources
```

> 💡 Формат: `pvesh get|set|create|delete <path>` — пути те же, что в URL `/api2/json/...`. Отличный способ автоматизировать управление без GUI.

---

## 🟡 17. ZFS на узле Proxmox

```bash
zpool status                    # состояние пулов (следи за error'ами и SMART)
zpool list                      # пулы и занятость
zfs list                        # датасеты
zpool scrub tank                # проверка целостности — делай по cron раз в месяц
zpool iostat -v 1               # активность дисков в реальном времени
zfs create tank/vmdata         # датасет
zfs set compression=lz4 tank/vmdata    # сжатие (обычно включают сразу)
zfs snapshot tank/vmdata@before-upgrade
zfs send tank/vmdata@snap | ssh root@backup zfs recv tank2/vmdata@snap   # репликация
```

> 💡 Снапшоты ZFS бесплатны по месту «до изменений». ARCoM (авто-замена дисков) и зеркалирование — читай [Proxmox wiki: ZFS](https://pve.proxmox.com/wiki/ZFS_on_Linux). ARC кэш ест память: ограничивай его параметром модуля `zfs_arc_max` (например: `echo 8G > /sys/module/zfs/parameters/zfs_arc_max`), чтобы хост не swap'ал.

---

## 🟡 18. Ceph (кратко)

> Ceph — распределённое хранилище прямо на узлах PVE: диски узлов = общее блочное хранилище `rbd` для VM. Стартово нужно ≥3 узла и ≥3 диска (OSD).

```bash
pveceph status           # health: HEALTH_OK?
ceph -s                  # общий статус кластера Ceph
ceph osd status          # OSD (диски)
ceph df                  # занятость
pveceph install          # установка Ceph на узел
pveceph createmon         # первый monitor
pveceph createosd /dev/sdb    # добавить диск как OSD (🔴 сотрёт данные с диска!)
```

> 💡 Для homelab Ceph часто избыточен: 3 узла × чистые диски + быстрый сеть 10G — иначе NFS/ZFS-репликация проще и предсказуемее.

---

## 🟢 19. Логи и задачи

```bash
journalctl -u pvedaemon           # API-демон
journalctl -u pveproxy           # веб-интерфейс (8006)
journalctl -u pve-cluster        # pmxcfs/corosync
journalctl -u qmeventd
grep -R "ERROR" /var/log/pve/tasks/       # 🔴 последняя задача
cat /var/log/pve/tasks/active            # что выполняется прямо сейчас
ls -lt /var/log/pve/tasks/ | head         # свежие логи задач (бэкапы, миграции)
dmesg -T | grep -i kvm                    # события KVM
grep -i 'killed process' /var/log/kern.log # OOM-killer на хосте
```

> 💡 В веб-интерфейсе: Datacenter → Tasks — все задачи кластера с логами. Каждое действие GUI/CLI пишется туда.

---

## 🟢 20. Обслуживание узла и обновления

```bash
apt update && apt dist-upgrade          # единственный правильный способ обновлять PVE
proxmox-boot-tool status               # как настроена загрузка (systemd-boot/ZFS)
proxmox-boot-tool refresh              # после обновления ядра (на EFI/ZFS системах)
pveversion                             # проверить, что стало после обновления
reboot                                  # после обновления ядра — перезагрузка узла

# Плановое обслуживание узла в кластере:
# На кластере из 3 узлов после потери одного quorum сохраняется сам (2 из 3) — pvecm expected менять не нужно
qm migrate 100 node2  # увести ресурсы
```

> 💡 Обновляй узлы кластера по одному: живой узел держит quorum, остальные — обновляются. В PVE 8 веб-интерфейс сам подсказывает доступные обновления (Changelog читай перед major-версиями).

---

### 🟡 Обслуживание ФС и хранилищ: LVM-thin, ZFS, ext4, TRIM, SMART

#### TRIM и discard (не давать тонким пулам «раздуваться»)

```bash
# В VM: диск должен быть virtio-scsi с discard=on
qm set 100 --scsi0 local-lvm:vm-100-disk-0,discard=on
# Внутри гостя Linux: периодический fstrim (обычно уже есть fstrim.timer)
systemctl status fstrim.timer && sudo fstrim -av

# LXC: rootfs тоже поддерживает discard
pct set 101 --rootfs local-lvm:8,discard=on

# LVM-thin на хосте: manual trim тонкого пула
fstrim -av                  # на хосте — смонтированные thin-тома
lvs -a                      # Data% тонкого пула — контролируй заполнение!
```

> 💡 Если Data% тонкого пула подходит к 100%, пул **зависает** (не просто «место кончилось»). Держи ниже 80% и следи за `lvs` — у тонкого пула есть и метаданные (Meta%).

#### ZFS-пул: регулярное обслуживание

```bash
zpool status -v                       # ошибки, resilver в процессе
zpool scrub tank                      # раз в месяц (systemd-таймеры zfs-scrub-weekly@tank / monthly)
zpool trim tank                       # TRIM на SSD-пулах; постоянный: zpool set autotrim=on tank
arcstat 1                             # ARC-кэш: hit-rate
# Ограничить ARC (по умолчанию до 50% RAM), постоянно:
echo 'options zfs zfs_arc_max=8589934592' | sudo tee /etc/modprobe.d/zfs.conf
update-initramfs -u -k all
```

> 💡 В веб-интерфейсе: Node → Disks — SMART и статус дисков. Уведомления о scrub-ошибках приходят через ZED (mail/Notification, PVE 8+: Datacenter → Notification).

#### ext4/загрузка

```bash
smartctl -a /dev/sda                  # SMART диска (пакет smartmontools)
sudo e2fsck -f /dev/pve/root          # 🔴 только с размонтированной ФС (загрузка с live/rescue)
# Расширение root после увеличения диска узла:
growpart /dev/sda 1 && resize2fs /dev/sda1
```

> 💡 Дефрагментация на хосте Proxmox не нужна ни для ext4 (extent-аллокатор), ни для ZFS (CoW). «Тормозит» — ищи в IO-задержке: `iostat -x 1` (await), а не во фрагментации.

---

## 🟡 21. Для профи: passthrough, hooks, тонкая настройка

### PCI passthrough (GPU, HBA, NIC в гостя)

```bash
# 1. Включить IOMMU в GRUB: intel_iommu=on (AMD — iommu уже включён)
#    /etc/kernel/cmdline (PVE 8) → proxmox-boot-tool refresh
dmesg | grep -e DMAR -e IOMMU      # проверить, что включился
pvesh get /nodes/pve1/hardware/pci --pci-class-blacklist ""   # список PCI-устройств
qm set 100 -hostpci0 0000:01:00.0  # передать GPU (все функции: pcie=1 см. wiki)
qm set 100 -hostpci0 0000:01:00.0,pcie=1,x-vga=1   # 🟡 как основная видеокарта
```

### CPU и память

```bash
qm set 100 --cpu host              # тип CPU: host = как у хоста (нужен для GNS3/nested-virt)
qm set 100 --numa 1                # NUMA-топология для больших VM
qm set 100 --balloon 1024          # ballooning: минимальная память в живую (1024 из 4096)
qm set 100 --hotplug memory,disk,cpu,net   # горячее подключение ресурсов
qm set 100 --tags prod,web         # теги (PVE 8+)
```

### Hookscripts (действия вокруг бэкапа)

```bash
qm set 100 --hookscript local:snippets/hook-stop-fw.sh
```

```bash
#!/usr/bin/env bash
# Пауза приложения перед бэкапом для консистентности БД
case "$2" in
  backup-start) systemctl stop myapp ;;
  backup-end)   systemctl start myapp ;;
esac
```

> 💡 `--hookscript` со стадиями `pre-start`, `post-stop`, `backup-start`, `backup-end`, `log-end` — гибкий механизм для консистентных бэкапов БД без полноценных агентов.

---

## ⚠️ 22. Опасные команды 🔴

Никогда не запускай их, не понимая, что они делают, — особенно если их прислали в чате или на форуме.

| Команда | Что произойдёт |
|---|---|
| `qm destroy <VMID> --purge` | Удалит VM **и все** её диски без подтверждения |
| `pct destroy <CTID>` | Удалит контейнер и его диск |
| `pvecm delnode <node>` | 🔴 Выкинет узел из кластера (без `--force` — только с полным пониманием последствий) |
| `rm -rf /etc/pve/*` | Уничтожит конфигурацию всех VM (pmxcfs) |
| `mkfs.ext4 /dev/sdX` на диске из ZFS/Ceph | Данные пулов будут потеряны |
| `zpool destroy tank` | Удалит весь пул ZFS |
| `pveceph createosd /dev/sdX` | 🔴 Сотрёт диск под OSD |
| `pvecm expected 1` на рабочем кластере | 🟡 «Разрешит» одиночное выживание — риск split-brain |
| `vzdump --all --storage local` без места | Переполнит диск узла — упадут VM и кластер |
| `apt upgrade` (не dist-upgrade) | Может рассинхронизировать версии ядра и пакетов PVE |

> 💡 Перед опасной операцией: `qm list`/`pvesm status` (проверить ID), свежий бэкап, `qm snapshot` и — главное — тестовое восстановление.

---

## 🩺 23. Диагностика: что делать, если…

| Симптом | Что делать |
|---|---|
| VM не стартует: `TASK ERROR: … lock` | `qm list` → статус `locked`? `cat /var/log/pve/tasks/active`, часто — незавершённая миграция/бэкап: перезапусти задачу или `systemctl restart pvedaemon` |
| Нет quorum после падения узла | `pvecm status` → меньше половины голосов → срочно чини узел или (временно, на свой риск) `pvecm expected 1` |
| Веб-интерфейс недоступен | `systemctl status pveproxy` → `journalctl -u pveproxy`, `ss -tlnp \| grep 8006` |
| Бэкап не вылез по NFS | `showmount -e server`, `pvesm status`, `mount \| grep backup`, проверь экспорт-опции `no_root_squash` |
| Диск VM переполнен внутри гостя | `qm resize 100 scsi0 +10G`, потом внутри гостя `growpart`/`resize2fs` (или `lvextend` в LXC) |
| Высокий load на хосте | `top` → какой KVM-процесс? → внутри VM; `qm status <id> --verbose \| grep cpu`; `iostat -x 1` (disk-IO?) |
| OOM-killer убивает процессы хоста | `dmesg -T \| grep -i 'killed process'` → уменьшить ballooning `--balloon`, выключить KSM для точности; следи за ARC (ZFS) |
| Corosync: кластер «дрожит» | `journalctl -u corosync`, задержки (`ping` между узлами), дубликаты ring-адресов? — `corosync-cfgtool -t` |
| IP VM не виден в GUI | Guest-агент не запущен или `--agent enabled=0` → включи и проверь `qm agent <id> ping` |
| Контейнер не стартует после обновления PVE | `pct config <id>` — нет `rootfs`-параметра? Восстанови конфиг из `/var/lib/lxc`/backup; смотри `/var/log/pve/tasks/` |

---

## 🎯 24. Вопросы с собеседований

<details>
<summary><b>Чем LXC отличается от VM (KVM)?</b></summary>

VM эмулирует железо, у неё собственное ядро; LXC — контейнер на общем ядре хоста с изоляцией namespaces/cgroups: быстрее старт, меньше накладных расходов, но гости только Linux и изоляция слабее. Непривилегированный LXC маппит UID (root в CT → 100000 на хосте).
</details>

<details>
<summary><b>Что такое quorum и зачем он?</b></summary>

Quorum — большинство голосов в кластере (n/2+1). Он предотвращает split-brain: при потере связи две «половины» кластера не могут обе считать себя активными — выживает только часть с quorum. Без quorum pmxcfs замораживает изменения, HA не выполняет действия. Для 2 узлов нужен QDevice — третий голос.
</details>

<details>
<summary><b>Чем vzdump-snapshot отличается от suspend/stop, и когда что использовать?</b></summary>

Snapshot — бэкап на живой VM через QEMU-снапшот + fsfreeze гостя через guest-агент: минимум даунтайма, консистентность ФС (но не всегда БД — используй hookscript). Suspend — пауза на время бэкапа, для БД без агента. Stop — выключение перед бэкапом: максимальная консистентность, максимальный даунтайм. Выбор = компромисс RPO/RTO.
</details>

<details>
<summary><b>Почему бэкапы на PBS «инкрементальные», если в GUI выглядит как полные?</b></summary>

PBS разбивает данные на чанки и дедуплицирует их: каждое задание логически выглядит полной копией (можно восстановить целиком), но по сети/диску пишутся только новые чанки. Удаление старого бэкапа ничего не ломает, пока чанки ссылаются более новыми бэкапами; место освобождает garbage collection.
</details>

<details>
<summary><b>Чем linked clone отличается от full clone?</b></summary>

Linked clone создаёт VM поверх снапшота родителя: почти мгновенно и без копирования места, но родительский снапшот нельзя удалить, пока живы клоны, и производительность диска зависит от родителя. Full clone — полная независимая копия: медленнее, занимает отдельное место, безопасно для продакшена.
</details>

<details>
<summary><b>Что такое pmxcfs и почему его поломка — катастрофа?</b></summary>

Proxmox Cluster File System — FUSE-ФС поверх SQLite (`/var/lib/pve-cluster/config.db`), смонтированная в `/etc/pve`, реплицируемая через corosync. В ней живут все конфиги VM/CT/кластера. Потеря базы на всех узлах = потеря конфигурации инфраструктуры (данные VM на дисках целы, но «склеить» придётся вручную). Поэтому — регулярный бэкап этой базы.
</details>

---

## 🧰 25. Практические сценарии

### Сценарий 1. VM с Ubuntu из cloud-образа за 5 минут

```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qm create 200 --name app-01 --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 \
  --agent enabled=1 --ostype l26
qm importdisk 200 noble-server-cloudimg-amd64.img local-lvm
qm set 200 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-200-disk-0
qm set 200 --ide2 local-lvm:cloudinit --boot order=scsi0 --serial0 socket --vga serial0
qm set 200 --ciuser deploy --sshkeys ~/.ssh/id_ed25519.pub \
  --ipconfig0 ip=192.168.1.70/24,gw=192.168.1.1
qm start 200 && ssh deploy@192.168.1.70
```

### Сценарий 2. Ежедневный бэкап всех VM на NFS

```bash
pvesm add nfs backup-nfs --server 192.168.1.10 --export /export/pve \
  --content backup --maxfiles 7

cat >> /etc/pve/jobs.cfg <<'EOF'
backup: all-vms
    schedule 02:30
    all 1
    mode snapshot
    compress zstd
    storage backup-nfs
    prune-backups keep-last=7,keep-weekly=4
EOF
```

### Сценарий 3. Кластер из трёх узлов

```bash
# Узел 1
pvecm create prod --link0 10.0.0.11
# Узел 2 и 3
pvecm add 10.0.0.11 --link0 10.0.0.12   # на каждом узле — свой link0 IP
pvecm status                             # убедиться: Quorate, 3 голоса
```

### Сценарий 4. Вывод узла в обслуживание

```bash
# HA-ресурсы: Maintenance mode для pve2 в GUI или перевези их ha-manager migrate вручную
qm migrate 100 pve3            # остальные — вручную
pct migrate 101 pve3
apt update && apt dist-upgrade && reboot
```

---

## 🎓 26. Практикум: задачи с решениями

### 🟢 Уровень 1: базовые операции

<details>
<summary><b>1. Создай контейнер Debian 12 с 1 ГБ RAM, 8 ГБ диска, статическим IP 192.168.1.60/24 и запусти его</b></summary>

```bash
pveam update
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
pct create 110 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname lab-ct --memory 1024 --rootfs local-lvm:8 --unprivileged 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.60/24,gw=192.168.1.1 --start 1
pct enter 110
```
</details>

<details>
<summary><b>2. Выведи список всех VM и контейнеров узла с их статусом одной командой (без GUI)</b></summary>

```bash
pvesh get /cluster/resources --type vm
```
</details>

<details>
<summary><b>3. Сделай снапшот VM 100 перед обновлением и откатись после неудачи</b></summary>

```bash
qm snapshot 100 pre-upgrade
# …обновление сломало сервис…
qm rollback 100 pre-upgrade
qm delsnapshot 100 pre-upgrade   # когда всё стабильно — удалить
```
</details>

<details>
<summary><b>4. Увеличь диск VM 100 на 10 ГБ и расширь ФС внутри гостя</b></summary>

```bash
qm resize 100 scsi0 +10G
# внутри гостя (Linux):
sudo growpart /dev/sda 1 && sudo resize2fs /dev/sda1     # ext4
# или: sudo lvextend -r -l +100%FREE /dev/vg0/root       # LVM
```
</details>

### 🟡 Уровень 2: бэкапы и восстановление

<details>
<summary><b>5. Сделай бэкап VM 100 в режиме snapshot с zstd и проверь, что файл появился</b></summary>

```bash
vzdump 100 --mode snapshot --compress zstd --storage local
ls -lh /var/lib/vz/dump/ | tail -1
```
</details>

<details>
<summary><b>6. Восстанови бэкап VM 100 как новую VM 300 (не трогая оригинал) и проверь загрузку</b></summary>

```bash
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst 300 --unique
qm start 300
qm agent 300 ping && echo "VM 300 живая"
```
</details>

<details>
<summary><b>7. Настрой задание бэкапа: ежедневно в 03:00, все VM, держать 7 последних</b></summary>

Через GUI: Datacenter → Backup → Add. Или в `/etc/pve/jobs.cfg`:

```
backup: daily
    schedule 03:00
    all 1
    mode snapshot
    storage local
    prune-backups keep-last=7
```

Проверить: `pvesh get /cluster/backup` и после 03:00 — `ls -lt /var/lib/vz/dump | head`.
</details>

<details>
<summary><b>8. Напиши hookscript, который останавливает приложение в контейнере 110 на время бэкапа</b></summary>

```bash
# Сохрани файл на хранилище с content=snippets: /path/snippets/ct-hook.sh
#!/usr/bin/env bash
case "$2" in
  backup-start) pct exec 110 -- systemctl stop myapp ;;
  backup-end)   pct exec 110 -- systemctl start myapp ;;
esac
# Подключить:
pct set 110 --hookscript local:snippets/ct-hook.sh
```
</details>

### 🔴 Уровень 3: кластер и HA

<details>
<summary><b>9. Собери кластер из двух узлов + QDevice на третьей машине</b></summary>

```bash
# pve1:
pvecm create lab --link0 10.0.0.11
# pve2:
pvecm add 10.0.0.11 --link0 10.0.0.12
# qdevice (любой Linux, можно не PVE):
apt install corosync-qnetd
# на pve1:
pvecm qdevice setup 10.0.0.13
pvecm status    # убедиться: 3 голоса, Quorate
```
</details>

<details>
<summary><b>10. Помести VM 100 под HA и проверь: выключи узел, где она работала</b></summary>

```bash
ha-manager add vm:100
ha-manager status
# Жёсткое отключение узла (имитация падения) — на самом узле по питанию;
# либо корректный тест:
# Жёсткое отключение узла — лучший тест HA; плановый вывод — через Maintenance mode
ha-manager status       # VM 100 должна переехать и запуститься на pve2
```
</details>

<details>
<summary><b>11. Узел из кластера выводится навсегда — как правильно его убрать?</b></summary>

На **оставшемся** узле: `pvecm delnode имя-узла` (узел должен быть выключен и осознан). Затем на бывшем узле, если он нужен standalone: снести `pve-cluster`/`corosync` по официальной процедуре wiki (удаление /etc/pve и /etc/corosync). 🔴 Никогда не выполняй `pvecm delnode` на «живом» узле без его вывода — кворум и конфигурация поедут.
</details>

<details>
<summary><b>12. Автоматизируй создание VM из шаблона: скрипт vm-new.sh <name> <ip></b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
name=${1:?имя} ip=${2:?ip}
id=$((200 + RANDOM % 700))
qm clone 150 "$id" --name "$name"
qm set "$id" --ipconfig0 ip="$ip/24,gw=192.168.1.1"
qm start "$id"
echo "VM $id ($name) запущена: $ip"
```
</details>

### 🏁 Задачи для самостоятельной работы

1. Подними PBS (можно в VM) и настрой инкрементальные бэкапы двух VM, затем выполни проверку `verify`.
2. Собери кластер из 3 узлов, включи HA для одной VM и попробуй плановый и аварийный вывод узла.
3. Передай USB-устройство в контейнер LXC (config: `dev0` + `cgroup`-права) и проверь, почему это хуже, чем passthrough в VM.
4. Настрой файрвол уровня datacenter: разрешить SSH только из management-сети, включить и убедиться, что доступ сохранился.
5. Смигрируй VM с локальным диском между узлами кластера в онлайн-режиме и замерь время.
6. Сделай cloud-init шаблон Debian/Ubuntu и полностью разворачивай тестовые VM из него скриптом.
7. Сымитируй потерю узла (выключи его) и опиши: что происходит с quorum, HA, задачами миграции; как вернуть всё назад.

---

## 📘 Глоссарий

| Термин | Коротко |
|---|---|
| **PVE** | Proxmox Virtual Environment |
| **KVM/QEMU** | Модуль ядра + эмулятор: полная виртуализация |
| **LXC** | Контейнеры Linux на общем ядре хоста |
| **pmxcfs** | Кластерная ФС конфигурации в `/etc/pve` (SQLite + corosync) |
| **VMID/CTID** | Идентификатор VM или контейнера |
| **Guest agent** | qemu-guest-agent: связь хост ↔ гость (IP, shutdown, fsfreeze) |
| **vzdump** | Инструмент бэкапа VM/CT в архив |
| **PBS** | Proxmox Backup Server: дедупликация, чанки, инкременты |
| **Quorum** | Большинство голосов кластера; защита от split-brain |
| **QDevice** | Внешний «голос» для quorum в кластере из 2 узлов |
| **HA** | High Availability: автоперезапуск ресурсов на живом узле |
| **Ballooning** | Динамическое изменение памяти VM гостем |
| **Cloud-Init** | Первичная настройка ОС при загрузке (пользователь, ключи, IP) |
| **Linked clone** | Клон поверх снапшота-родителя |
| **SDN** | Software Defined Networking: зоны VLAN/VXLAN в PVE 8+ |
| **OSD** | Диск-участник хранилища Ceph |
| **Hookscript** | Скрипт-хук на стадиях жизненного цикла VM/бэкапа |
| **Ring** | Адрес/интерфейс corosync для кластерного трафика |

---

## 📖 Полезные ресурсы

- [Proxmox VE Wiki](https://pve.proxmox.com/wiki/Main_Page) — официальная документация, лучшая в классе
- [Proxmox VE Administration Guide (PDF)](https://proxmox.com/en/downloads) — полный справочник по всем утилитам (pvecm, qm, pct, pvesm)
- [Proxmox Backup Server Documentation](https://pbs.proxmox.com/docs/) — про бэкапы
- [Community Forum](https://forum.proxmox.com/) — плотный багаж решений
- [r/Proxmox](https://www.reddit.com/r/Proxmox/) — homelab-практики
- [Cheat sheet cdocsa (EN)](https://github.com/cdocsa/cheat-sheets/blob/main/devops/proxmox/proxmox-cli-cheat-sheet.md) — CLI-справочник, сверен с официальной документацией
- [Proxmox commands (shortcut-tools, EN)](https://shortcut-tools.com/en/shortcuts/proxmox/) — 353 команды по PVE 9

---

## 🤝 Как помочь проекту

Нашёл ошибку или знаешь полезную команду?

1. Сделай **Fork**
2. Добавь или исправь команду (коротко, с понятным описанием)
3. Открой **Pull Request**

<div align="center">

**Если шпаргалка помогла — поставь ⭐!**

Распространяется по лицензии **MIT**.

</div>

<sub>proxmox шпаргалка · proxmox ve · pve · kvm · qm · pct · lxc · vzdump · pbs · proxmox backup server · corosync · quorum · ha-manager · zfs · ceph · cloud-init · devops · виртуализация · sysadmin</sub>