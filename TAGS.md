# 🏷 Справочник тегов — proxmox-cheats

Теги для навигации и поиска. Основная шпаргалка — `README.md` (русский, стиль linux-cheats), источники — `sources/`. Новое значение добавляется сначала сюда.

## topic (тематика)

| Тег | Значение |
|---|---|
| `install-upgrade` | установка, обновление PVE, репозитории (enterprise/no-subscription) |
| `storage-zfs` | ZFS: пулы, датасеты, снапшоты, ARC, замена дисков |
| `storage-lvm` | LVM, LVM-thin, локальные хранилища |
| `ceph` | Ceph: RBD, CephFS, OSD, мониторинг кластера |
| `network` | сеть: bridges, VLAN, bonds, SDN, firewall PVE |
| `qemu-kvm` | QEMU/KVM: qm, драйверы, CPU-типы, GPU-passthrough, миграция |
| `lxc` | LXC-контейнеры: pct, привилегированные/непривилегированные |
| `backup-restore` | бэкап (vzdump, PBS), восстановление, репликация |
| `cluster-ha` | кластер, quorum, corosync, HA, fencing |
| `guest-agent` | QEMU guest agent, cloud-init, шаблоны ВМ |
| `pvesh-api` | pvesh, REST API, токены, автоматизация |
| `hardening` | усиление безопасности PVE, 2FA, RBAC, audit |
| `performance` | производительность, тюнинг, balloon, IO-планировщики |

## type (характер команды)

| Тег | Значение |
|---|---|
| `theory` | теория, концепция |
| `basic` | 🟢 базовая безопасная команда |
| `advanced` | 🟡 требует понимания архитектуры |
| `dangerous` | 🔴 опасно (кластер, хранилище, живые ВМ) |
| `tip` | 💡 совет |
| `scenario` | практический сценарий «сделай X» |

## severity (для опасных операций)

| Тег | Значение |
|---|---|
| `data-loss` | риск потери данных хранилища/ВМ |
| `downtime` | простой ВМ или ноды |
| `cluster-split` | риск расщепления кластера/потери quorum |
| `irreversible` | необратимо без бэкапа |

## Конвенция для новых шпаргалок
- Авторские разделы — на русском, с легендой 🟢🟡🔴💡 (как linux-cheats).
- Каждой команде группы dangerous — тег severity из таблицы выше.
- Иностранные источники складывать в `sources/`, не переводить оригинал.
