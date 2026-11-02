# Ручная установка HA NFS-кластера на ALT Linux c10f2

Этот документ описывает ручную установку кластера без Ansible и без установочного
скрипта. Его цель — сначала понять и проверить каждый компонент, а уже потом решать,
нужна ли автоматизация.

Документ рассчитан на **чистые виртуальные машины**. Не выполняйте команды
инициализации хранилища на уже работающем кластере.

## 1. Что получится в результате

Топология стенда:

| Узел | IP | Назначение | Диск с данными |
| --- | --- | --- | --- |
| `node1` | `192.168.88.159` | DRBD, NFS, Pacemaker | Да |
| `node2` | `192.168.88.160` | DRBD, NFS, Pacemaker | Да |
| `node3` | `192.168.88.158` | quorum и diskless DRBD | Нет |

Клиенты будут подключаться к NFS через VIP `192.168.88.200/24`.

Цепочка запуска ресурсов:

```text
DRBD Primary
  -> XFS с состоянием NFS
  -> XFS с пользовательскими данными
  -> NFS server
  -> NFS export
  -> VIP 192.168.88.200
```

Одновременно эта цепочка работает только на `node1` или `node2`. `node3` хранит
голоса Corosync и DRBD quorum, но не хранит пользовательские данные и не может стать
DRBD Primary.

## 2. Обозначения и правила выполнения

Перед каждой группой команд указано место выполнения:

- **все узлы** — выполнить на `node1`, `node2` и `node3`;
- **diskful-узлы** — выполнить только на `node1` и `node2`;
- **node1** — выполнить только на `node1`;
- **NFS-клиент** — выполнить на отдельной клиентской машине.

Все команды на серверах выполняются от `root`.

В рабочем кластере узлам не нужен SSH-доступ друг к другу. `scp` ниже используется
только для удобного переноса конфигурации во время установки. Эти же файлы можно
скопировать с административной машины.

### Опасные команды

Следующие команды изменяют разметку или данные и выделены в отдельных разделах:

```text
pvcreate
vgcreate
lvcreate
drbdadm create-md
drbdadm primary --force
mkfs.xfs
```

Перед ними обязательно проверяются выбранные устройства. В документе нет `wipefs`,
автоматического разрешения split-brain и принудительного пересоздания XFS.

## 3. Начальные условия

Перед установкой должны выполняться условия:

1. На всех узлах установлен ALT Linux c10f2 и ядро 6.12.
2. У каждого узла уникальные hostname, IP, `/etc/machine-id` и SSH host keys.
3. На `node1` и `node2` есть отдельный пустой диск примерно 50 ГиБ.
4. На `node3` дополнительного диска нет.
5. Узлы видят друг друга по сети.
6. На узлах синхронизировано время.
7. VIP `192.168.88.200` свободен.

Проверка версии системы и ядра — **на всех узлах**:

```bash
cat /etc/altlinux-release
uname -r
```

Команды только читают информацию. Версия ядра должна начинаться с `6.12`.

## 4. Настройка имён узлов

На `node1`:

```bash
hostnamectl set-hostname node1
```

На `node2`:

```bash
hostnamectl set-hostname node2
```

На `node3`:

```bash
hostnamectl set-hostname node3
```

`hostnamectl set-hostname` меняет постоянное имя текущего узла.

Добавьте на **всех узлах** в `/etc/hosts`:

```text
192.168.88.159 node1
192.168.88.160 node2
192.168.88.158 node3
```

Проверка — **на всех узлах**:

```bash
hostnamectl --static
getent hosts node1 node2 node3
ping -c 2 node1
ping -c 2 node2
ping -c 2 node3
```

`getent hosts` проверяет разрешение имён, `ping` — базовую сетевую связность.

Проверьте идентичность клонированных VM — **на всех узлах**:

```bash
cat /etc/machine-id
ssh-keygen -lf /etc/openssh/ssh_host_ed25519_key.pub
```

Значения должны отличаться на всех трёх узлах. Если они совпадают, остановите
установку и подготовьте клоны корректно. Этот runbook намеренно не меняет системную
идентичность автоматически.

## 5. Проверка сети, времени и VIP

На **всех узлах**:

```bash
ip -brief address
ip route
timedatectl status
```

Команды показывают адреса, маршруты и состояние синхронизации времени.

Проверьте, что VIP пока никем не используется — с любого узла:

```bash
ping -c 3 192.168.88.200
ip neigh show 192.168.88.200
```

До создания ресурса Pacemaker ответов от VIP быть не должно. Если адрес отвечает,
нужно найти его владельца и выбрать другой свободный адрес.

Порты, которые должны пропускаться внутренней сетью кластера:

| Назначение | Протокол/порт |
| --- | --- |
| Corosync KNET | UDP 5405 |
| DRBD | TCP 7788 |
| NFSv4 для клиентов | TCP 2049 |

Не отключайте firewall целиком. Правила firewall зависят от принятой на серверах
системы управления и в этот runbook пока не включены.

## 6. Проверка и установка пакетов

Сначала проверьте доступность пакетов — **на всех узлах**:

```bash
apt-get update
apt-get --simulate install \
  kernel-modules-drbd9-6.12 drbd-utils drbd-utils-pacemaker \
  corosync pacemaker pacemaker-cli pcs sbd \
  resource-agents resource-agents-nfs \
  nfs-server nfs-utils rpcbind xfsprogs
```

`apt-get update` обновляет метаданные настроенных репозиториев.
`apt-get --simulate install` ничего не устанавливает, а показывает, могут ли
репозитории разрешить все зависимости.

Если симуляция завершилась без ошибок, установите пакеты — **на всех узлах**:

```bash
apt-get install -y \
  kernel-modules-drbd9-6.12 drbd-utils drbd-utils-pacemaker \
  corosync pacemaker pacemaker-cli pcs sbd \
  resource-agents resource-agents-nfs \
  nfs-server nfs-utils rpcbind xfsprogs
```

ALT может автоматически запустить установленные службы. До завершения настройки
остановите и отключите их — **на всех узлах**:

```bash
systemctl disable --now pacemaker corosync drbd nfs-server
systemctl disable sbd
```

На **node1 и node2** дополнительно установите LVM:

```bash
apt-get install -y lvm2
```

Проверьте установленные версии — **на всех узлах**:

```bash
rpm -q drbd-utils corosync pacemaker pcs sbd resource-agents nfs-server
modinfo -F version drbd
```

`modinfo` читает версию установленного модуля. Она должна быть не ниже DRBD 9.

Загрузите модуль и проверьте фактическую версию — **на всех узлах**:

```bash
modprobe drbd
cat /sys/module/drbd/version
```

Если вывод показывает DRBD 8 или модуль для текущего ядра не найден, дальше не
продолжайте.

## 7. Настройка watchdog для лаборатории

В стенде используется `softdog`. Он подходит для проверки логики SBD, но не является
production fencing: при полном зависании ядра программный watchdog тоже может не
сработать. Для production нужен аппаратный watchdog или независимое внешнее fencing.

Загрузите `softdog` — **на всех узлах**:

```bash
modprobe softdog
test -c /dev/watchdog
ls -l /dev/watchdog*
```

`test -c` завершается успешно, если `/dev/watchdog` является символьным устройством.

ALT Linux может не загружать `softdog` через обычный modules-load. Поэтому создайте
**на всех узлах** `/etc/systemd/system/ha-nfs-watchdog.service`:

```ini
[Unit]
Description=Load watchdog module for HA NFS
After=systemd-modules-load.service
Before=sbd.service

[Service]
Type=oneshot
ExecStart=/sbin/modprobe softdog
RemainAfterExit=yes

[Install]
RequiredBy=sbd.service
```

Примените unit — **на всех узлах**:

```bash
systemctl daemon-reload
systemctl enable --now ha-nfs-watchdog.service
systemctl is-active ha-nfs-watchdog.service
test -c /dev/watchdog
```

## 8. Настройка watchdog-only SBD

Создайте **на всех узлах** `/etc/sysconfig/sbd`:

```bash
install -m 0600 /dev/null /etc/sysconfig/sbd
cat >/etc/sysconfig/sbd <<'EOF'
SBD_DEVICE=""
SBD_PACEMAKER=yes
SBD_STARTMODE=always
SBD_DELAY_START=no
SBD_WATCHDOG_DEV=/dev/watchdog
SBD_WATCHDOG_TIMEOUT=30
SBD_TIMEOUT_ACTION=flush,reboot
SBD_MOVE_TO_ROOT_CGROUP=auto
SBD_SYNC_RESOURCE_STARTUP=yes
SBD_OPTS=
EOF
```

Здесь нет SBD-диска: каждый узел подтверждает свою работоспособность через локальный
watchdog. Таймаут watchdog равен 30 секундам.

Включите SBD в автозагрузку — **на всех узлах**:

```bash
systemctl enable sbd
```

Пока не запускайте SBD вручную. В ALT Linux он запускается как зависимость Corosync и
имеет защиту `RefuseManualStart=true`.

Никогда не меняйте watchdog-устройство или таймаут на работающем SBD без поузлового
maintenance. Неверное изменение может вызвать перезагрузку узла.

## 9. Поиск дисков данных

Этот раздел выполняется только на **node1 и node2**.

Посмотрите все диски и стабильные имена:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
findmnt /
ls -l /dev/disk/by-id/
```

Выберите отдельный пустой диск на каждом узле. Используйте путь
`/dev/disk/by-id/...`, а не `/dev/sdb`: имя `/dev/sdX` может измениться после reboot.

Сохраните выбранный путь в переменную только для текущего shell:

```bash
DATA_DISK=/dev/disk/by-id/ЗАМЕНИТЬ_НА_РЕАЛЬНЫЙ_ID
```

Проверьте, куда ведёт ссылка:

```bash
readlink -f "$DATA_DISK"
lsblk -f "$DATA_DISK"
wipefs --no-act "$DATA_DISK"
pvs
vgs
lvs
```

`wipefs --no-act` только показывает найденные сигнатуры и ничего не удаляет.

**Остановитесь**, если на выбранном диске есть разделы, файловая система, LVM PV или
любые нужные данные. Также убедитесь, что это не диск с `/`.

## 10. Создание LVM — необратимая инициализация

> ОПАСНО: следующие команды изменяют выбранный диск. Выполняйте их только после
> проверки предыдущего раздела и только на пустых дисках `node1` и `node2`.

На **node1 и node2**, используя правильный локальный `DATA_DISK`:

```bash
pvcreate "$DATA_DISK"
vgcreate vg_ha_nfs "$DATA_DISK"
lvcreate -L 1G -n nfs_state vg_ha_nfs
lvcreate -L 48G -n nfs_data vg_ha_nfs
```

Назначение команд:

- `pvcreate` создаёт LVM Physical Volume на диске;
- `vgcreate` создаёт Volume Group `vg_ha_nfs`;
- первый `lvcreate` создаёт 1 ГиБ для общего состояния NFS;
- второй `lvcreate` создаёт 48 ГиБ для пользовательских данных.

Проверьте результат — **на node1 и node2**:

```bash
pvs
vgs
lvs -a -o name,vg_name,lv_size,devices vg_ha_nfs
```

Размеры `nfs_state` и `nfs_data` должны совпадать на обоих узлах.

## 11. Создание конфигурации DRBD

Сгенерируйте секрет **один раз на node1**:

```bash
umask 077
DRBD_SECRET=$(openssl rand -hex 24)
printf '%s\n' "$DRBD_SECRET" >/root/ha-nfs-drbd-secret
```

`openssl rand` создаёт случайный секрет. Файл `/root/ha-nfs-drbd-secret` нельзя
публиковать в Git.

Не закрывая этот shell, создайте **на node1** `/etc/drbd.d/ha_nfs.res`:

```bash
cat >/etc/drbd.d/ha_nfs.res <<EOF
resource ha_nfs {
  protocol C;

  options {
    auto-promote no;
    quorum majority;
    on-no-quorum suspend-io;
    on-no-data-accessible suspend-io;
  }

  net {
    allow-two-primaries no;
    cram-hmac-alg sha256;
    shared-secret "$DRBD_SECRET";
  }

  on node1 {
    node-id 0;
    address ipv4 192.168.88.159:7788;
    volume 0 {
      device minor 0;
      disk /dev/vg_ha_nfs/nfs_state;
      meta-disk internal;
    }
    volume 1 {
      device minor 1;
      disk /dev/vg_ha_nfs/nfs_data;
      meta-disk internal;
    }
  }

  on node2 {
    node-id 1;
    address ipv4 192.168.88.160:7788;
    volume 0 {
      device minor 0;
      disk /dev/vg_ha_nfs/nfs_state;
      meta-disk internal;
    }
    volume 1 {
      device minor 1;
      disk /dev/vg_ha_nfs/nfs_data;
      meta-disk internal;
    }
  }

  on node3 {
    node-id 2;
    address ipv4 192.168.88.158:7788;
    volume 0 {
      device minor 0;
      disk none;
    }
    volume 1 {
      device minor 1;
      disk none;
    }
  }

  connection-mesh {
    hosts node1 node2 node3;
  }
}
EOF

chmod 0600 /etc/drbd.d/ha_nfs.res
```

Скопируйте тот же файл на `node2` и `node3`:

```bash
scp -p /etc/drbd.d/ha_nfs.res root@node2:/etc/drbd.d/ha_nfs.res
scp -p /etc/drbd.d/ha_nfs.res root@node3:/etc/drbd.d/ha_nfs.res
```

`scp` нужен только для установки; кластер не использует SSH для failover.

Проверьте синтаксис — **на всех узлах**:

```bash
drbdadm dump ha_nfs
```

Команда разбирает конфигурацию и выводит её нормализованный вариант. При ошибке
исправьте конфигурацию до продолжения.

## 12. Создание DRBD metadata — необратимая инициализация

> ОПАСНО: metadata создаётся только на diskful-узлах и только один раз.

На **node1 и node2**:

```bash
drbdadm create-md ha_nfs
```

Команда создаёт внутренние DRBD metadata в конце обоих logical volumes. На `node3`
её выполнять нельзя, потому что узел diskless.

После успешного создания поднимите ресурс — сначала на `node1` и `node2`, затем на
`node3`:

```bash
drbdadm up ha_nfs
drbdadm status ha_nfs
```

`drbdadm up` создаёт DRBD-устройства и соединения, но не выбирает источник данных.

## 13. Выбор первоначального источника DRBD

Убедитесь, что это действительно новая установка и на томах нет нужных данных.

Только на **node1**:

```bash
drbdadm primary --force ha_nfs
```

> ОПАСНО: `--force` допустим только при первой инициализации совершенно нового
> ресурса. Он объявляет содержимое `node1` эталонным. При восстановлении существующего
> кластера эту команду использовать нельзя.

Наблюдайте синхронизацию на `node1` или `node2`:

```bash
watch -n 2 drbdadm status ha_nfs
```

Выход из `watch` — `Ctrl+C`.

Продолжать можно только когда оба тома на `node1` и `node2` имеют `UpToDate`, а
соединения находятся в `Established`. `node3` должен отображаться как diskless.

## 14. Создание XFS — необратимая операция

> ОПАСНО: `mkfs.xfs` уничтожает существующую файловую систему. Выполняется один раз,
> только на первоначальном DRBD Primary и только после полной синхронизации.

На **node1**:

```bash
mkfs.xfs /dev/drbd/by-res/ha_nfs/0
mkfs.xfs /dev/drbd/by-res/ha_nfs/1
```

Том `0` хранит состояние NFS, том `1` — пользовательские данные.

Создайте каталоги и служебный каталог NFS — на **node1**:

```bash
mkdir -p /srv/ha-nfs/state /srv/ha-nfs/data
mount /dev/drbd/by-res/ha_nfs/0 /srv/ha-nfs/state
mkdir -m 0700 /srv/ha-nfs/state/nfs_info
findmnt /srv/ha-nfs/state
umount /srv/ha-nfs/state

mount /dev/drbd/by-res/ha_nfs/1 /srv/ha-nfs/data
chmod 0777 /srv/ha-nfs/data
findmnt /srv/ha-nfs/data
umount /srv/ha-nfs/data
```

`0777` используется для лабораторного теста записи с `root_squash`. В production
настройте согласованные UID/GID, группу и более строгие права, например `2770`.

Проверьте тип файловых систем:

```bash
blkid /dev/drbd/by-res/ha_nfs/0
blkid /dev/drbd/by-res/ha_nfs/1
```

Обе должны иметь `TYPE="xfs"`.

Верните DRBD в Secondary — на **node1**:

```bash
drbdadm secondary ha_nfs
```

Передайте управление DRBD Pacemaker — **на всех узлах**:

```bash
drbdadm down ha_nfs
systemctl disable --now drbd
```

`drbdadm down` убирает ручной экземпляр ресурса. Данные и metadata не удаляются.
Дальше DRBD поднимает только Pacemaker.

## 15. Настройка Corosync

Создайте ключ — только на **node1**:

```bash
corosync-keygen -l
chmod 0400 /etc/corosync/authkey
```

Ключ аутентифицирует трафик Corosync. Скопируйте его на остальные узлы:

```bash
scp -p /etc/corosync/authkey root@node2:/etc/corosync/authkey
scp -p /etc/corosync/authkey root@node3:/etc/corosync/authkey
```

Создайте **на всех узлах** одинаковый `/etc/corosync/corosync.conf`:

```text
totem {
  version: 2
  cluster_name: ha-nfs
  transport: knet
  crypto_cipher: aes256
  crypto_hash: sha256
}

nodelist {
  node {
    name: node1
    nodeid: 1
    ring0_addr: 192.168.88.159
  }
  node {
    name: node2
    nodeid: 2
    ring0_addr: 192.168.88.160
  }
  node {
    name: node3
    nodeid: 3
    ring0_addr: 192.168.88.158
  }
}

quorum {
  provider: corosync_votequorum
  expected_votes: 3
}

logging {
  to_syslog: yes
  timestamp: on
}
```

Проверьте конфигурацию — **на всех узлах**:

```bash
corosync -t
```

Команда проверяет синтаксис и ничего не запускает.

Запустите Corosync и Pacemaker — сначала `node1`, затем `node2`, затем `node3`:

```bash
systemctl enable --now corosync pacemaker
```

Pacemaker запускается сразу вместе с Corosync, чтобы SBD не оставался без процесса,
который он должен контролировать. После запуска всех узлов проверьте — **на каждом
узле**:

```bash
systemctl is-active corosync
systemctl is-active sbd
systemctl is-active pacemaker
corosync-quorumtool -s
```

Ожидается `Quorate: Yes`, три узла, активные SBD и Pacemaker.

## 16. Базовые свойства Pacemaker

Проверьте с `node1`:

```bash
pcs status
```

Должны отображаться три online-узла без ресурсов.

Следующие команды выполняются только на **node1**:

```bash
pcs property set fencing-enabled=true
pcs property set fencing-watchdog-timeout=60s
pcs property set no-quorum-policy=stop
pcs resource defaults update resource-stickiness=200
pcs property config --all
pcs resource defaults config
```

Назначение свойств:

- `fencing-enabled=true` включает обязательную изоляцию отказавшего узла;
- `fencing-watchdog-timeout=60s` даёт SBD с таймаутом 30 секунд время перезагрузить
  потерянный узел до запуска ресурсов на другом;
- `no-quorum-policy=stop` останавливает ресурсы без большинства голосов;
- `resource-stickiness=200` не возвращает ресурсы назад сразу после восстановления
  старого узла.

В режиме watchdog-only отдельный STONITH resource не создаётся. Fencing выполняется
через SBD и локальный watchdog каждого узла.

## 17. Подготовка NFS на узлах

Выполните только на **diskful-узлах** `node1` и `node2`:

```bash
systemctl enable --now rpcbind
systemctl disable --now nfs-server
mkdir -p /srv/ha-nfs/state /srv/ha-nfs/data
```

`rpcbind` остаётся системной службой. `nfs-server` отключается, потому что его должен
запускать и останавливать только Pacemaker.

Создайте на **diskful-узлах** `/usr/local/sbin/ha-nfs-server`:

```sh
#!/bin/sh
case "$1" in
  start) systemctl start nfs-server ;;
  stop) systemctl stop nfs-server ;;
  restart) systemctl restart nfs-server ;;
  status) systemctl is-active --quiet nfs-server && exit 0; exit 3 ;;
  *) exit 2 ;;
esac
```

Установите права — на **diskful-узлах**:

```bash
chmod 0755 /usr/local/sbin/ha-nfs-server
/usr/local/sbin/ha-nfs-server status; test "$?" -eq 3
```

Код `3` означает, что служба корректно определена как остановленная. Обёртка нужна из-за
особенностей проверки статуса NFS в ALT Linux.

## 18. Создание ресурсов Pacemaker

Все команды раздела выполняются только на **node1**.

Включите maintenance mode, чтобы ресурсы не начали запускаться до создания всей
цепочки и ограничений:

```bash
pcs property set maintenance-mode=true
```

Создайте promotable DRBD resource:

```bash
pcs resource create p_drbd_nfs ocf:linbit:drbd \
  drbd_resource=ha_nfs \
  op start interval=0s timeout=240s \
  op stop interval=0s timeout=240s \
  op monitor interval=29s timeout=30s role=Promoted \
  op monitor interval=31s timeout=30s role=Unpromoted \
  promotable promoted-max=1 promoted-node-max=1 \
  clone-max=3 clone-node-max=1 notify=true
```

Pacemaker создаёт экземпляр DRBD на каждом узле, но повышает до Primary только один.

Создайте группу `g_nfs`. Порядок добавления определяет порядок запуска:

```bash
pcs resource create p_fs_nfs_state ocf:heartbeat:Filesystem \
  device=/dev/drbd/by-res/ha_nfs/0 \
  directory=/srv/ha-nfs/state fstype=xfs run_fsck=no \
  op monitor interval=20s timeout=40s \
  --group g_nfs

pcs resource create p_fs_nfs_data ocf:heartbeat:Filesystem \
  device=/dev/drbd/by-res/ha_nfs/1 \
  directory=/srv/ha-nfs/data fstype=xfs run_fsck=no \
  op monitor interval=20s timeout=40s \
  --group g_nfs

pcs resource create p_nfsserver ocf:heartbeat:nfsserver \
  nfs_shared_infodir=/srv/ha-nfs/state/nfs_info \
  nfs_ip=192.168.88.200 \
  nfs_init_script=/usr/local/sbin/ha-nfs-server \
  op monitor interval=15s timeout=30s \
  --group g_nfs

pcs resource create p_exportfs ocf:heartbeat:exportfs \
  clientspec=192.168.88.0/24 \
  directory=/srv/ha-nfs/data fsid=0 \
  options=rw,sync,root_squash unlock_on_stop=1 \
  op monitor interval=15s timeout=30s \
  --group g_nfs

pcs resource create p_nfs_vip ocf:heartbeat:IPaddr2 \
  ip=192.168.88.200 cidr_netmask=24 \
  op monitor interval=10s timeout=20s \
  --group g_nfs
```

Ресурсы группы выполняют следующие функции:

- `p_fs_nfs_state` монтирует общий служебный XFS;
- `p_fs_nfs_data` монтирует пользовательский XFS;
- `p_nfsserver` запускает NFS server;
- `p_exportfs` публикует `/srv/ha-nfs/data` как корень NFSv4;
- `p_nfs_vip` назначает клиентский адрес `192.168.88.200`.

Запретите повышение DRBD на diskless `node3`:

```bash
pcs constraint location p_drbd_nfs-clone rule \
  role=Promoted score=-INFINITY \
  constraint-id=location-p_drbd_nfs-clone \
  '#uname' eq node3
```

Разместите NFS вместе с DRBD Primary:

```bash
pcs constraint colocation add g_nfs \
  with Promoted p_drbd_nfs-clone INFINITY \
  id=colocation-g_nfs-p_drbd_nfs-clone-INFINITY
```

Задайте порядок запуска:

```bash
pcs constraint order promote p_drbd_nfs-clone then start g_nfs \
  id=order-p_drbd_nfs-clone-g_nfs-mandatory
```

Проверьте итоговую конфигурацию до запуска:

```bash
pcs resource config
pcs constraint config --full
crm_verify -L -V
```

`crm_verify` проверяет согласованность CIB. При ошибке maintenance mode не выключайте.

Если ошибок нет, разрешите Pacemaker запустить ресурсы:

```bash
pcs property set maintenance-mode=false
watch -n 2 pcs status --full
```

Дождитесь стабильного состояния и выйдите из `watch` через `Ctrl+C`.

## 19. Проверка результата

На **node1**:

```bash
pcs status --full
pcs resource status
pcs constraint config --full
pcs property config --all
corosync-quorumtool -s
drbdadm status ha_nfs
```

На **всех узлах**:

```bash
systemctl is-active ha-nfs-watchdog.service
systemctl is-active sbd
systemctl is-active corosync
systemctl is-active pacemaker
```

На активном diskful-узле:

```bash
findmnt /srv/ha-nfs/state
findmnt /srv/ha-nfs/data
exportfs -v
ip -brief address | grep 192.168.88.200
```

Ожидаемый результат:

1. Все три узла `Online`.
2. Corosync имеет quorum.
3. SBD активен на всех узлах.
4. DRBD Promoted только на `node1` или `node2`.
5. Оба дисковых экземпляра DRBD находятся в `UpToDate`.
6. `node3` остаётся diskless и Unpromoted.
7. Все пять ресурсов `g_nfs` запущены на DRBD Primary.
8. На том же узле назначен VIP `192.168.88.200`.
9. В `pcs status` нет failed actions и `UNCLEAN` узлов.

## 20. Проверка с NFS-клиента

На отдельном **NFS-клиенте**:

```bash
mkdir -p /mnt/ha-nfs
mount -t nfs4 192.168.88.200:/ /mnt/ha-nfs
printf 'ha-nfs-test\n' >/mnt/ha-nfs/test.txt
sync
sha256sum /mnt/ha-nfs/test.txt
cat /mnt/ha-nfs/test.txt
```

`mount` подключает NFS только через VIP. Запомните контрольную сумму: после failover
она должна остаться той же.

## 21. Плановый тест failover

Сначала определите активный узел:

```bash
pcs status --full
```

Если группа `g_nfs` работает на `node1`, выполните с любого кластерного узла:

```bash
pcs node standby node1
watch -n 2 pcs status --full
```

Если она работает на `node2`, замените `node1` на `node2`.

Pacemaker должен:

1. остановить VIP, export, NFS и файловые системы;
2. понизить DRBD на старом узле;
3. повысить DRBD на втором diskful-узле;
4. смонтировать XFS;
5. запустить NFS, export и VIP.

Во время и после переключения на NFS-клиенте:

```bash
cat /mnt/ha-nfs/test.txt
sha256sum /mnt/ha-nfs/test.txt
```

Сумма должна совпасть с первоначальной.

Верните узел в кластер:

```bash
pcs node unstandby node1
```

Из-за `resource-stickiness=200` ресурсы не должны автоматически возвращаться назад.

## 22. Проверка после перезагрузки

Перезагружайте только один узел за раз. Перед каждым reboot убедитесь, что два других
узла online и кластер стабилен.

Для арбитра:

```bash
pcs node standby node3
ssh root@node3 reboot
```

После возврата `node3`:

```bash
pcs status --full
ssh root@node3 systemctl is-active ha-nfs-watchdog.service
ssh root@node3 systemctl is-active sbd
ssh root@node3 corosync-quorumtool -s
pcs node unstandby node3
```

Для diskful-узла сначала убедитесь, что он не обслуживает `g_nfs`. Затем переведите
его в standby, перезагрузите и дождитесь `UpToDate` перед следующими тестами.

## 23. Диагностика

Общее состояние:

```bash
pcs status --full
crm_mon -1Arf
corosync-quorumtool -s
drbdadm status ha_nfs
```

Последние сообщения служб:

```bash
journalctl -u ha-nfs-watchdog.service -b --no-pager
journalctl -u sbd -b --no-pager
journalctl -u corosync -b --no-pager
journalctl -u pacemaker -b --no-pager
journalctl -k -b | grep -i drbd
```

Сеть и прослушиваемые порты:

```bash
ss -lntup
ip -brief address
ip route
```

DRBD:

```bash
drbdadm status ha_nfs
drbdsetup status --verbose --statistics
```

Pacemaker:

```bash
pcs resource config
pcs resource failcount show
pcs constraint config --full
pcs property config --all
```

Не выполняйте автоматически при проблемах:

```text
drbdadm primary --force
drbdadm invalidate
drbdadm disconnect --discard-my-data
pcs resource delete
mkfs.xfs
```

Сначала нужно определить, на каком узле находится актуальная копия данных и почему
кластер остановил ресурс.

После устранения причины ошибку конкретного ресурса можно очистить командой:

```bash
pcs resource cleanup ИМЯ_РЕСУРСА
```

Она удаляет историю ошибки и разрешает Pacemaker повторить запуск. Она не исправляет
первопричину.

## 24. Тест watchdog

Команда `sbd test-watchdog` намеренно перезагружает узел. Это не обычная диагностика.
Её можно запускать только в согласованное окно на тестовом стенде, по одному узлу и
при здоровых оставшихся участниках.

```bash
sbd -w /dev/watchdog test-watchdog
```

Успешным результатом является принудительный reboot. Без отдельного согласования эту
команду не выполнять.

## 25. Расширение хранилища

Расширение нельзя начинать, пока кластер нестабилен, идёт DRBD resync или отсутствует
резервная копия.

Безопасная последовательность на уровне проекта:

1. Сделать проверенную резервную копию данных NFS.
2. Увеличить физические диски `node1` и `node2` на одинаковый размер.
3. Убедиться, что ОС обоих узлов видит новый размер.
4. Выполнить `pvresize` на обоих узлах.
5. Одинаково увеличить `nfs_data` на обоих узлах.
6. Увеличить DRBD volume 1.
7. Увеличить XFS только на активном смонтированном томе.
8. Проверить failover и размер после переключения.

Команды осмотра, которые не изменяют данные:

```bash
lsblk
pvs
vgs
lvs
drbdadm status ha_nfs
findmnt /srv/ha-nfs/data
xfs_info /srv/ha-nfs/data
```

Точные изменяющие команды расширения будут добавлены только после отдельного теста на
копии стенда. До такой проверки не используйте непроверенную процедуру resize на
production-данных.

## 26. Критерии готовности

Кластер можно считать собранным только после выполнения всех проверок:

- три узла online и Corosync quorate;
- SBD и watchdog переживают reboot каждого узла;
- DRBD имеет только один Primary и две diskful-копии `UpToDate`;
- `node3` никогда не становится Primary;
- XFS монтируется только Pacemaker;
- NFS доступен через `192.168.88.200`;
- плановый failover сохраняет данные и контрольную сумму;
- возвращённый узел синхронизируется без автоматического failback;
- в Pacemaker нет failed actions;
- риск использования `softdog` принят только для лаборатории.

