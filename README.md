# DRBD 9, Pacemaker, Corosync and NFS

Ansible-проект для развёртывания отказоустойчивого NFSv4-кластера на
ALT Linux c10f2.

Кластер работает в режиме active-passive: только один storage-узел одновременно
держит DRBD Primary, монтирует XFS, обслуживает NFS и поднимает VIP.

## Архитектура

| Узел | IP | Назначение |
| --- | --- | --- |
| `node1` | `192.168.88.159` | DRBD diskful, возможный NFS active |
| `node2` | `192.168.88.160` | DRBD diskful, возможный NFS active |
| `node3` | `192.168.88.158` | Corosync quorum и diskless DRBD tiebreaker |
| VIP | `192.168.88.200` | Адрес подключения NFS-клиентов |

`node3` не хранит пользовательские данные и не требует отдельного data-диска.

Порядок запуска ресурсов:

```text
DRBD Primary
└── XFS state
    └── XFS data
        └── NFS server
            └── NFS export
                └── VIP 192.168.88.200
```

Corosync предоставляет членство и quorum. Pacemaker управляет DRBD, файловыми
системами, NFS и VIP. SBD использует watchdog для самозащиты потерявшего quorum
узла.

## Проверенное окружение

- ALT SP Server c10f2;
- kernel `6.12.113-6.12-alt0.c10f.2`;
- DRBD `9.2.13`;
- Pacemaker `3.0.3-alt1`;
- Ansible Core `2.15.9` на управляющей `node3`;
- `ansible.posix 1.5.4`;
- `community.general 8.6.10`.

Более новые версии коллекций из Ansible Galaxy несовместимы с Ansible Core 2.15,
поэтому версии зафиксированы и сохранены в проекте.

## Структура проекта

```text
.
├── ansible.cfg
├── playbook.yml
├── inventory/
│   ├── inventory.ini
│   └── host_vars/
│       ├── node1.yml
│       └── node2.yml
├── collections/
│   ├── requirements.yml
│   └── offline/
│       ├── ansible-posix-1.5.4.tar.gz
│       ├── community-general-8.6.10.tar.gz
│       └── requirements.yml
├── MANUAL-INSTALLATION-RU.md
└── roles/drbd9-pacemaker-corosync-nfs/
    ├── defaults/main.yml
    ├── tasks/
    ├── templates/
    └── files/
```

Основной вход — `playbook.yml`. Обычный запуск без тегов ничего не развёртывает.

## Подготовка узлов

Перед запуском необходимо:

1. Установить ALT Linux c10f2 и ядро 6.12 на трёх узлах.
2. Установить hostname `node1`, `node2` и `node3`.
3. Настроить root-доступ по SSH-ключу между управляющей машиной и всеми узлами.
4. Добавить отдельный data-диск только на `node1` и `node2`.
5. Указать диски через постоянные пути `/dev/disk/by-id/`.
6. Проверить, что VIP `192.168.88.200` не используется другим устройством.

Пример проверки дисков:

```bash
lsblk -d -o NAME,PATH,SIZE,MODEL,SERIAL,TYPE
ls -l /dev/disk/by-id/
```

Текущие лабораторные пути находятся в:

```text
inventory/host_vars/node1.yml
inventory/host_vars/node2.yml
```

Роль проверяет, что указанный путь существует и является блочным устройством.
Создание LVM и DRBD metadata разрешается только явным параметром запуска.

## Управляющая машина на ALT Linux

В текущем стенде Ansible запускается с `node3`:

```bash
apt-get install ansible python3-module-packaging
```

Репозиторий расположен в:

```text
/root/drbd9-pacemaker-corosync-nfs
```

Проверка соединения:

```bash
cd /root/drbd9-pacemaker-corosync-nfs
ansible cluster -m ping
```

## Установка коллекций без интернета

Архивы уже находятся в `collections/offline/`. Интернет для установки не нужен:

```bash
cd /root/drbd9-pacemaker-corosync-nfs/collections/offline

ansible-galaxy collection install \
  -r requirements.yml \
  --collections-path .. \
  --offline

cd ../..
```

Проверка версий:

```bash
ansible-galaxy collection list --collections-path collections
```

Ожидаемый результат:

```text
ansible.posix       1.5.4
community.general   8.6.10
```

Для установки с доступом в интернет используется `collections/requirements.yml`.

## Основные параметры

Параметры находятся в
`roles/drbd9-pacemaker-corosync-nfs/defaults/main.yml`.

| Переменная | Значение | Назначение |
| --- | --- | --- |
| `ha_nfs_state_lv_size` | `1g` | Общее состояние NFS |
| `ha_nfs_data_lv_size` | `48g` | Пользовательские данные |
| `ha_nfs_filesystem` | `xfs` | Файловая система DRBD-томов |
| `ha_nfs_vip` | `192.168.88.200` | Виртуальный IP NFS |
| `ha_nfs_clients` | `192.168.88.0/24` | Разрешённая сеть клиентов |
| `ha_nfs_data_mode` | `0777` | Права корня общего хранилища |
| `ha_nfs_sbd_watchdog_timeout` | `30` | Таймаут watchdog SBD |
| `ha_nfs_sbd_recovery_timeout` | `60` | Ожидание fencing Pacemaker |

Размеры LV должны помещаться на выбранном data-диске. `node3` в LVM не участвует.

## Развёртывание

Развёртывание разделено на два явных этапа, чтобы дождаться первоначальной
синхронизации DRBD вручную.

### Шаг 1 — LVM и DRBD

> Команда создаёт LVM и DRBD metadata на дисках из `host_vars`. Не запускайте её
> для дисков с нужными данными без отдельного плана миграции.

```bash
ansible-playbook playbook.yml --tags step1 \
  -e ha_nfs_storage_initialize=true
```

Во время этого этапа кластерные службы остановлены. SBD и softdog запускаются
только на втором этапе, поэтому watchdog не мешает первичной синхронизации.

Наблюдение за синхронизацией:

```bash
watch -n 2 "ssh root@192.168.88.159 drbdadm status ha_nfs"
```

Можно переходить дальше, когда состояние выглядит так:

```text
ha_nfs role:Primary
  volume:0 disk:UpToDate
  volume:1 disk:UpToDate
  node2 role:Secondary
    volume:0 peer-disk:UpToDate
    volume:1 peer-disk:UpToDate
  node3 role:Secondary
    volume:0 peer-disk:Diskless
    volume:1 peer-disk:Diskless
```

Не должно оставаться состояний `Inconsistent`, `Outdated`, `Connecting`,
`SyncSource` или `SyncTarget`.

### Шаг 2 — XFS и HA-кластер

```bash
ansible-playbook playbook.yml --tags step2
```

Этап выполняет следующее:

- создаёт XFS без принудительного переформатирования;
- подготавливает общее состояние NFS;
- настраивает softdog и watchdog-only SBD;
- запускает Corosync и проверяет quorum;
- запускает Pacemaker;
- создаёт promotable DRBD-ресурс;
- создаёт файловые системы, NFS export и VIP;
- проверяет итоговое состояние кластера.

Повторный запуск `step2` безопасен. На проверенном стенде итог повторного запуска:

```text
node1: changed=0 failed=0
node2: changed=0 failed=0
node3: changed=0 failed=0
```

## Проверка кластера

```bash
pcs status --full
drbdadm status ha_nfs
corosync-quorumtool -s
systemctl status sbd corosync pacemaker
ip address show | grep 192.168.88.200
```

Ожидается:

- три узла online и `Quorate: Yes`;
- один DRBD `Promoted`, два `Unpromoted`;
- NFS-группа полностью запущена на DRBD Primary;
- VIP присутствует только на активном storage-узле;
- отсутствуют `Failed Resource Actions`.

`pcsd` в этом проекте не используется. Поэтому `pcs status --full` может показывать
для PCSD `Unable to authenticate`; это не влияет на Corosync и Pacemaker.

## Проверка с NFS-клиента

```bash
mkdir -p /mnt/ha-nfs
mount -t nfs4 192.168.88.200:/ /mnt/ha-nfs
echo test >/mnt/ha-nfs/test.txt
sync
cat /mnt/ha-nfs/test.txt
umount /mnt/ha-nfs
```

На стенде дополнительно проверена запись файла `ansible-test.txt` через VIP.

## Проверка аварийного переключения под нагрузкой

Тест выполнен с NFS-клиента при жёстком отключении активной `node1`.

Монтирование NFS:

```bash
mkdir -p test
mount -t nfs4 192.168.88.200:/ test
```

Непрерывная запись с контрольными суммами:

```bash
fio --name=nfs-ha-write \
  --filename="$PWD/test/fio-ha-test.bin" \
  --size=2G --rw=write --bs=1M --ioengine=sync --direct=1 \
  --fsync=16 --time_based=1 --runtime=180 \
  --verify=crc32c --do_verify=0 --group_reporting
```

Во время записи активная VM `node1` была выключена без штатного завершения ОС:

```bash
VBoxManage controlvm "altlinux 2" poweroff
```

Команда относится только к текущему стенду VirtualBox. На другом гипервизоре
нужно использовать его механизм аварийного отключения активного узла.

Результат переключения:

- `node2` и `node3` сохранили quorum;
- SBD завершил fencing отключённой `node1`;
- DRBD стал `Primary` на `node2`;
- обе XFS, NFS и VIP автоматически запустились на `node2`;
- NFS-клиент не потребовал повторного монтирования;
- FIO продолжил запись без I/O errors;
- максимальная пауза одной операции составила около `93,6` секунды;
- за 180 секунд записано `9293 MiB` со средней скоростью `51,6 MiB/s`.

После переключения целостность файла проверена повторным чтением CRC:

```bash
fio --name=nfs-ha-verify \
  --filename="$PWD/test/fio-ha-test.bin" \
  --size=2G --rw=read --bs=1M --ioengine=sync --direct=1 \
  --verify=crc32c --verify_only=1 --group_reporting
```

Проверка завершилась с `err=0`: прочитано и проверено `2048 MiB`. Итоговое
состояние кластера: `node1` — `OFFLINE`, `node2` — DRBD `Primary` и активная
NFS-группа, `node3` — diskless DRBD `Secondary` и участник quorum.

Тестовые файлы `fio-ha-test.bin` и `failover-after.txt` оставлены на NFS. Возврат
`node1` и повторная синхронизация DRBD выполняются отдельным тестом.

## Права NFS и root_squash

Export создаётся с параметром `root_squash`. Пользователь `root` на клиенте не
получает root-права на сервере, а преобразуется в `nobody:nobody`.

При правах `0755 root:root` клиентский root может читать каталог, но не может
создавать файлы. Поэтому лабораторное значение:

```yaml
ha_nfs_data_mode: "0777"
```

позволяет выполнить простой тест записи. Созданный клиентским root файл будет
иметь владельца `nobody:nobody`.

Для production `0777` обычно слишком широко. Рекомендуется использовать
согласованные UID/GID на NFS-клиентах, отдельную группу и режим наподобие `2770`
либо настроенные ACL. `no_root_squash` не рекомендуется: он даёт клиентскому root
полные права на серверном хранилище.

## Особенности ALT Linux

- установочный CD-ROM-репозиторий автоматически отключается перед `apt_rpm`;
- сервисы Pacemaker, Corosync, SBD, DRBD и NFS останавливаются до начальной
  настройки, потому что пакеты ALT могут запустить их автоматически;
- Pacemaker 3 в ALT использует свойства `fencing-enabled` и
  `fencing-watchdog-timeout`;
- Python зафиксирован как `/usr/bin/python3.9` в inventory;
- все task name и диагностические сообщения роли написаны на русском языке.

## Ограничения и безопасность

- `softdog` подходит для лабораторного стенда. В production нужен аппаратный
  watchdog, продолжающий работать при зависании ядра или виртуальной машины.
- Watchdog-only SBD требует настоящего quorum. Три узла должны участвовать в
  Corosync, даже если только два узла имеют data-диски.
- Не запускайте ручной `mount`, `drbdadm primary` или системный `nfs-server` после
  передачи ресурсов под управление Pacemaker.
- Не отключайте fencing и не устанавливайте `no-quorum-policy=ignore`.
- Перед production необходимо отдельно проверить управляемый failover и поведение
  аппаратного watchdog.

## Материалы для изучения

Рекомендуемый порядок изучения стека:

1. **Pacemaker** — ресурсы, группы, promotable clones, constraints, quorum,
   fencing и resource stickiness:
   [Pacemaker Explained 3.0](https://clusterlabs.org/projects/pacemaker/doc/3.0/Pacemaker_Explained/pdf/Pacemaker_Explained.pdf).
2. **DRBD 9** — роли Primary/Secondary, репликация, синхронизация, quorum,
   split-brain и интеграция с Pacemaker:
   [DRBD 9 User's Guide](https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/).
   Для этого проекта особенно важна глава 9 об интеграции с Pacemaker.
3. **Corosync** — членство узлов, обмен сообщениями и quorum:
   [официальная документация Corosync](https://corosync.github.io/).
4. **SBD и watchdog** — самозащита узла, watchdog-only режим и fencing:
   [официальная документация SBD](https://github.com/ClusterLabs/sbd/blob/main/man/sbd.8.pod.in).

Команды для изучения текущей конфигурации Pacemaker:

```bash
pcs status --full
pcs resource config
pcs constraint config --full
pcs property config
pcs resource defaults
pcs config
cibadmin --query | less
```

Описание используемых resource agents:

```bash
pcs resource describe ocf:linbit:drbd
pcs resource describe ocf:heartbeat:Filesystem
pcs resource describe ocf:heartbeat:nfsserver
pcs resource describe ocf:heartbeat:exportfs
pcs resource describe ocf:heartbeat:IPaddr2
```

Диагностика DRBD:

```bash
drbdadm status ha_nfs
drbdsetup status --verbose --statistics
cat /proc/drbd
man drbdadm
man drbdsetup
man drbd.conf-9.0
```

Диагностика Corosync и SBD:

```bash
corosync-quorumtool -s
corosync-cfgtool -s
corosync-cmapctl | less
journalctl -u corosync
systemctl status sbd
journalctl -u sbd
man sbd
```

Команды `sbd test-watchdog` и ручное управление ролями DRBD нельзя запускать на
работающем кластере без отдельного тестового сценария: они могут вызвать
перезагрузку узла или конфликт с Pacemaker.

Подробная ручная последовательность находится в
[`MANUAL-INSTALLATION-RU.md`](MANUAL-INSTALLATION-RU.md).
