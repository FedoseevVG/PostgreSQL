# Установка и настройка PostgreSQL

## Создаем ВМ pg01
```bash
yc compute instance create --name pg01 --hostname pg01 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub
```
Подключаемся:
```bash
ssh ubuntu@51.250.76.81
```

## Устанавливаем PostgreSQL 15 и проверяем, что кластер работает
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
ubuntu@pg01:~$
```

Заходим под postgres в psql и создаем таблицу:
```bash
ubuntu@pg01:~$ sudo -u postgres psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# commit;
WARNING:  there is no transaction in progress
COMMIT
postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \q
ubuntu@pg01:~$
```

Останавливаем postgres:
```bash
ubuntu@pg01:~$ sudo systemctl stop postgresql@15-main
ubuntu@pg01:~$ sudo systemctl status postgresql@15-main
○ postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: inactive (dead) since Thu 2023-08-24 09:56:47 UTC; 7s ago
    Process: 37769 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=0/SUCCESS)
    Process: 37803 ExecStop=/usr/bin/pg_ctlcluster --skip-systemctl-redirect -m fast 15-main stop (code=exited, status=0/SUCCESS)
   Main PID: 37774 (code=exited, status=0/SUCCESS)
        CPU: 230ms

Aug 24 09:56:21 pg01 systemd[1]: Starting PostgreSQL Cluster 15-main...
Aug 24 09:56:23 pg01 systemd[1]: Started PostgreSQL Cluster 15-main.
Aug 24 09:56:47 pg01 systemd[1]: Stopping PostgreSQL Cluster 15-main...
Aug 24 09:56:47 pg01 systemd[1]: postgresql@15-main.service: Deactivated successfully.
Aug 24 09:56:47 pg01 systemd[1]: Stopped PostgreSQL Cluster 15-main.
ubuntu@pg01:~$
```

## Добавляем новый диск размером 10GB к ВМ 
```bash
PS C:\Users\fvg00> yc compute disk list
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
|          ID          | NAME |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
| fhm2582tvja868qfa413 |      | 10737418240 | ru-central1-a | READY  | fhm0572rbetng04abrsc |                 |             |
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+

PS C:\Users\fvg00> yc compute disk create --name disk2 --size 10
done (9s)
id: fhmngkgocvosc5tq1sne
folder_id: b1g17etduc7a8cals26t
created_at: "2023-08-24T10:08:43Z"
name: disk2
type_id: network-hdd
zone_id: ru-central1-a
size: "10737418240"
block_size: "4096"
status: READY
disk_placement_policy: {}

PS C:\Users\fvg00> yc compute disk list
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+
|          ID          | NAME  |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+
| fhm2582tvja868qfa413 |       | 10737418240 | ru-central1-a | READY  | fhm0572rbetng04abrsc |                 |             |
| fhmngkgocvosc5tq1sne | disk2 | 10737418240 | ru-central1-a | READY  |                      |                 |             |
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+

PS C:\Users\fvg00> yc compute instance list
+----------------------+------+---------------+---------+--------------+-------------+
|          ID          | NAME |    ZONE ID    | STATUS  | EXTERNAL IP  | INTERNAL IP |
+----------------------+------+---------------+---------+--------------+-------------+
| fhm0572rbetng04abrsc | pg01 | ru-central1-a | RUNNING | 51.250.76.81 | 10.128.0.13 |
+----------------------+------+---------------+---------+--------------+-------------+

PS C:\Users\fvg00> yc compute instance attach-disk pg01 --disk-name disk2 --mode rw --auto-delete
done (8s)
id: fhm0572rbetng04abrsc
folder_id: b1g17etduc7a8cals26t
created_at: "2023-08-24T09:41:29Z"
name: pg01
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "2147483648"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhm2582tvja868qfa413
  auto_delete: true
  disk_id: fhm2582tvja868qfa413
secondary_disks:
  - mode: READ_WRITE
    device_name: fhmngkgocvosc5tq1sne
    auto_delete: true
    disk_id: fhmngkgocvosc5tq1sne
network_interfaces:
  - index: "0"
    mac_address: d0:0d:29:c5:b5:bb
    subnet_id: e9bkc3iu7l84ksblvk6r
    primary_v4_address:
      address: 10.128.0.13
      one_to_one_nat:
        address: 51.250.76.81
        ip_version: IPV4
gpu_settings: {}
fqdn: pg01.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}

PS C:\Users\fvg00> yc compute disk list
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+
|          ID          | NAME  |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+
| fhm2582tvja868qfa413 |       | 10737418240 | ru-central1-a | READY  | fhm0572rbetng04abrsc |                 |             |
| fhmngkgocvosc5tq1sne | disk2 | 10737418240 | ru-central1-a | READY  | fhm0572rbetng04abrsc |                 |             |
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+

PS C:\Users\fvg00>
```

Подключаемся к ВМ:
```bash
ssh ubuntu@51.250.76.81

ubuntu@pg01:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.3M  1 loop /snap/core20/1822
loop1    7:1    0  63.5M  1 loop /snap/core20/2015
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0  49.8M  1 loop /snap/snapd/18357
loop4    7:4    0  53.3M  1 loop /snap/snapd/19457
vda    252:0    0    10G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    10G  0 part /
vdb    252:16   0    10G  0 disk
```

Инициализируем новый диск /dev/vdb, монтируем его как /mnt/data и добавляем в автозагрузку
```bash
ubuntu@pg01:~$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.

ubuntu@pg01:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.

ubuntu@pg01:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.3M  1 loop /snap/core20/1822
loop1    7:1    0  63.5M  1 loop /snap/core20/2015
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0  49.8M  1 loop /snap/snapd/18357
loop4    7:4    0  53.3M  1 loop /snap/snapd/19457
vda    252:0    0    10G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    10G  0 part /
vdb    252:16   0    10G  0 disk
└─vdb1 252:17   0    10G  0 part

ubuntu@pg01:~$ sudo mkfs.ext4 -L datapartition /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 1442aea6-c089-4bd5-b527-051fe9a8edbe
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

ubuntu@pg01:~$ sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME   FSTYPE   LABEL         UUID                                 MOUNTPOINT
loop0  squashfs                                                    /snap/core20/1822
loop1  squashfs                                                    /snap/core20/2015
loop2  squashfs                                                    /snap/lxd/24322
loop3  squashfs                                                    /snap/snapd/18357
loop4  squashfs                                                    /snap/snapd/19457
vda
├─vda1
└─vda2 ext4                   ed465c6e-049a-41c6-8e0b-c8da348a3577 /
vdb
└─vdb1 ext4     datapartition 1442aea6-c089-4bd5-b527-051fe9a8edbe

ubuntu@pg01:~$ sudo mkdir -p /mnt/data
ubuntu@pg01:~$ sudo mount -o defaults /dev/vdb1 /mnt/data

ubuntu@pg01:~$ sudo lsblk --fs
NAME   FSTYPE   FSVER LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                            0   100% /snap/core20/1822
loop1  squashfs 4.0                                                            0   100% /snap/core20/2015
loop2  squashfs 4.0                                                            0   100% /snap/lxd/24322
loop3  squashfs 4.0                                                            0   100% /snap/snapd/18357
loop4  squashfs 4.0                                                            0   100% /snap/snapd/19457
vda
├─vda1
└─vda2 ext4     1.0                 ed465c6e-049a-41c6-8e0b-c8da348a3577    4.8G    46% /
vdb
└─vdb1 ext4     1.0   datapartition 1442aea6-c089-4bd5-b527-051fe9a8edbe    9.2G     0% /mnt/data

ubuntu@pg01:~$ sudo vi /etc/fstab

ubuntu@pg01:~$ sudo cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/ed465c6e-049a-41c6-8e0b-c8da348a3577 / ext4 defaults 0 1
/dev/disk/by-uuid/1442aea6-c089-4bd5-b527-051fe9a8edbe /mnt/data ext4 defaults 0 2
ubuntu@pg01:~$
```

Делаем рестарт ВМ и проверяем, что диск подмонтировался:
```bash
ubuntu@pg01:~$ sudo reboot
Connection to 51.250.76.81 closed by remote host.
Connection to 51.250.76.81 closed.
PS C:\Users\fvg00> ssh ubuntu@51.250.76.81
PS C:\Users\fvg00> ssh ubuntu@51.250.76.81
ssh: connect to host 51.250.76.81 port 22: Connection refused
PS C:\Users\fvg00> ssh ubuntu@51.250.76.81
Last login: Thu Aug 24 10:16:32 2023 from 5.228.137.12

ubuntu@pg01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/vda2       9.8G  4.5G  4.9G  48% /
tmpfs           982M  1.1M  981M   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
tmpfs           197M  4.0K  197M   1% /run/user/1000
ubuntu@pg01:~$
```

Делаем пользователя postgres владельцем /mnt/data, останавливаем кластер и переносим данные из /var/lib/postgresql/15 в /mnt/data
```bash
ubuntu@pg01:~$ sudo systemctl stop postgresql@15-main

ubuntu@pg01:~$ sudo chown -R postgres:postgres /mnt/data/

ubuntu@pg01:~$ sudo mv /var/lib/postgresql/15 /mnt/data
ubuntu@pg01:~$ sudo ls -al /mnt/data/
total 28
drwxr-xr-x 4 postgres postgres  4096 Aug 24 11:21 .
drwxr-xr-x 3 root     root      4096 Aug 24 10:28 ..
drwxr-xr-x 3 postgres postgres  4096 Aug 24 09:50 15
drwx------ 2 postgres postgres 16384 Aug 24 10:27 lost+found
ubuntu@pg01:~$
```

Пробуем запустить кластер и получаем ошибку, что каталог /var/lib/postgresql/15/main не существует:
```bash
ubuntu@pg01:~$ sudo systemctl start postgresql@15-main
Job for postgresql@15-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@15-main.service" and "journalctl -xeu postgresql@15-main.service" for details.
ubuntu@pg01:~$
ubuntu@pg01:~$ sudo systemctl status postgresql@15-main
× postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: failed (Result: protocol) since Thu 2023-08-24 11:23:48 UTC; 36s ago
    Process: 1257 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=1/FAILURE)
        CPU: 40ms

Aug 24 11:23:47 pg01 systemd[1]: Starting PostgreSQL Cluster 15-main...
Aug 24 11:23:47 pg01 postgresql@15-main[1257]: Error: /var/lib/postgresql/15/main is not accessible or does not exist
Aug 24 11:23:47 pg01 systemd[1]: postgresql@15-main.service: Can't open PID file /run/postgresql/15-main.pid (yet?) after start: Operation not permitted
Aug 24 11:23:47 pg01 systemd[1]: postgresql@15-main.service: Failed with result 'protocol'.
Aug 24 11:23:48 pg01 systemd[1]: Failed to start PostgreSQL Cluster 15-main.
ubuntu@pg01:~$
```

Переходим в каталог /etc/postgresql/15/main под postgres и редактируем в файле postgresql.conf параметр data_directory
```bash
ubuntu@pg01:~$ cd /etc/postgresql/15/main
ubuntu@pg01:/etc/postgresql/15/main$ sudo su postgres
postgres@pg01:/etc/postgresql/15/main$ cat postgresql.conf | grep /var/lib/postgresql
data_directory = '/var/lib/postgresql/15/main'          # use data in another directory
postgres@pg01:/etc/postgresql/15/main$ vi postgresql.conf

postgres@pg01:/etc/postgresql/15/main$ cat postgresql.conf | grep data_directory
data_directory = '/mnt/data/15/main'            # use data in another directory
postgres@pg01:/etc/postgresql/15/main$
```

После этого успешно стартуем кластер. 
```bash
ubuntu@pg01:/etc/postgresql/15/main$ sudo systemctl start postgresql@15-main
ubuntu@pg01:/etc/postgresql/15/main$ sudo systemctl status postgresql@15-main
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Thu 2023-08-24 11:35:31 UTC; 7s ago
    Process: 1328 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=0/SUCCESS)
   Main PID: 1333 (postgres)
      Tasks: 6 (limit: 2219)
     Memory: 18.7M
        CPU: 149ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@15-main.service
             ├─1333 /usr/lib/postgresql/15/bin/postgres -D /mnt/data/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
             ├─1334 "postgres: 15/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─1335 "postgres: 15/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─1337 "postgres: 15/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─1338 "postgres: 15/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             └─1339 "postgres: 15/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">

Aug 24 11:35:28 pg01 systemd[1]: Starting PostgreSQL Cluster 15-main...
Aug 24 11:35:31 pg01 systemd[1]: Started PostgreSQL Cluster 15-main.
ubuntu@pg01:/etc/postgresql/15/main$
```

Далее заходим через psql и проверяем содержимое тестовой таблицы:
```bash
ubuntu@pg01:/etc/postgresql/15/main$ sudo -u postgres psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \q
ubuntu@pg01:/etc/postgresql/15/main$
```

# Перемонтируем новый диск (disk2) c PostgreSQL на другую виртуальную машину с установленным PostgreSQL и попробуем запустить

## Останавливаем postgres и отмонтируем второй диск от ВМ pg01, далее проверяем, что он отмонтировался
```bash
ubuntu@pg01:/etc/postgresql/15/main$ sudo systemctl stop postgresql@15-main
ubuntu@pg01:/etc/postgresql/15/main$ exit
PS C:\Users\fvg00> yc compute instance get --full pg01

PS C:\Users\fvg00> yc compute instance detach-disk pg01 --disk-id fhmngkgocvosc5tq1sne
done (14s)
id: fhm0572rbetng04abrsc
folder_id: b1g17etduc7a8cals26t
created_at: "2023-08-24T09:41:29Z"
name: pg01
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "2147483648"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhm2582tvja868qfa413
  auto_delete: true
  disk_id: fhm2582tvja868qfa413
network_interfaces:
  - index: "0"
    mac_address: d0:0d:29:c5:b5:bb
    subnet_id: e9bkc3iu7l84ksblvk6r
    primary_v4_address:
      address: 10.128.0.13
      one_to_one_nat:
        address: 51.250.76.81
        ip_version: IPV4
gpu_settings: {}
fqdn: pg01.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}

PS C:\Users\fvg00>
PS C:\Users\fvg00> ssh ubuntu@51.250.76.81

ubuntu@pg01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/vda2       9.8G  4.5G  4.9G  48% /
tmpfs           982M     0  982M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           197M  4.0K  197M   1% /run/user/1000
ubuntu@pg01:~$ exit
logout
Connection to 51.250.76.81 closed.
PS C:\Users\fvg00>
PS C:\Users\fvg00> yc compute disk list
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+
|          ID          | NAME  |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+
| fhm2582tvja868qfa413 |       | 10737418240 | ru-central1-a | READY  | fhm0572rbetng04abrsc |                 |             |
| fhmngkgocvosc5tq1sne | disk2 | 10737418240 | ru-central1-a | READY  |                      |                 |             |
+----------------------+-------+-------------+---------------+--------+----------------------+-----------------+-------------+

PS C:\Users\fvg00>
```


## Создаем ВМ pg02
```bash
yc compute instance create --name pg02 --hostname pg02 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub
```
Подключаемся:
```bash
ssh ubuntu@158.160.102.255
```

## Устанавливаем PostgreSQL 15 и проверяем, что кластер работает
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

ubuntu@pg02:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
ubuntu@pg02:
```

## Подключаем disk2 к pg02
```bash
PS C:\Users\fvg00> yc compute instance attach-disk pg02 --disk-name disk2 --mode rw
done (9s)
id: fhmtedmvk2aro4dobk7t
folder_id: b1g17etduc7a8cals26t
created_at: "2023-08-24T12:19:37Z"
name: pg02
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "2147483648"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhm05jk4nkk9sran3l7n
  auto_delete: true
  disk_id: fhm05jk4nkk9sran3l7n
secondary_disks:
  - mode: READ_WRITE
    device_name: fhmngkgocvosc5tq1sne
    disk_id: fhmngkgocvosc5tq1sne
network_interfaces:
  - index: "0"
    mac_address: d0:0d:1d:73:6d:fa
    subnet_id: e9bkc3iu7l84ksblvk6r
    primary_v4_address:
      address: 10.128.0.19
      one_to_one_nat:
        address: 158.160.102.255
        ip_version: IPV4
gpu_settings: {}
fqdn: pg02.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}

PS C:\Users\fvg00> ssh ubuntu@158.160.102.255
ubuntu@pg02:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.3M  1 loop /snap/core20/1822
loop1    7:1    0  63.5M  1 loop /snap/core20/2015
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0  49.8M  1 loop /snap/snapd/18357
loop4    7:4    0  53.3M  1 loop /snap/snapd/19457
vda    252:0    0    10G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    10G  0 part /
vdb    252:16   0    10G  0 disk
└─vdb1 252:17   0    10G  0 part
ubuntu@pg02:~$
ubuntu@pg02:~$ sudo mkdir -p /mnt/data
ubuntu@pg02:~$ sudo mount -o defaults /dev/vdb1 /mnt/data

ubuntu@pg02:~$ sudo ls -al /mnt/data/15/main/
total 88
drwx------ 19 postgres postgres 4096 Aug 24 12:14 .
drwxr-xr-x  3 postgres postgres 4096 Aug 24 09:50 ..
drwx------  5 postgres postgres 4096 Aug 24 09:50 base
drwx------  2 postgres postgres 4096 Aug 24 11:36 global
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_commit_ts
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_dynshmem
drwx------  4 postgres postgres 4096 Aug 24 12:14 pg_logical
drwx------  4 postgres postgres 4096 Aug 24 09:50 pg_multixact
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_notify
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_replslot
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_serial
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_snapshots
drwx------  2 postgres postgres 4096 Aug 24 12:14 pg_stat
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_stat_tmp
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_subtrans
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_tblspc
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_twophase
-rw-------  1 postgres postgres    3 Aug 24 09:50 PG_VERSION
drwx------  3 postgres postgres 4096 Aug 24 09:50 pg_wal
drwx------  2 postgres postgres 4096 Aug 24 09:50 pg_xact
-rw-------  1 postgres postgres   88 Aug 24 09:50 postgresql.auto.conf
-rw-------  1 postgres postgres  120 Aug 24 11:35 postmaster.opts
ubuntu@pg02:~$
```

Останавливаем кластер, удаляем все из /var/lib/postgresql
Переходим в каталог /etc/postgresql/15/main под postgres и редактируем в файле postgresql.conf параметр data_directory
```bash
ubuntu@pg02:~$ sudo systemctl stop postgresql@15-main
ubuntu@pg02:~$ sudo rm -fr /var/lib/postgresql/*
ubuntu@pg02:~$ sudo ls -al /var/lib/postgresql
total 16
drwxr-xr-x  2 postgres postgres 4096 Aug 24 12:40 .
drwxr-xr-x 40 root     root     4096 Aug 24 12:28 ..
-rw-------  1 postgres postgres   66 Aug 24 12:38 .bash_history
-rw-------  1 postgres postgres 1026 Aug 24 12:37 .viminfo
ubuntu@pg02:~$
ubuntu@pg02:~$ cd /etc/postgresql/15/main
ubuntu@pg02:/etc/postgresql/15/main$ sudo su postgres
postgres@pg02:/etc/postgresql/15/main$ vi postgresql.conf
postgres@pg02:/etc/postgresql/15/main$ cat postgresql.conf | grep data_directory
data_directory = '/mnt/data/15/main'            # use data in another directory
postgres@pg02:/etc/postgresql/15/main$ exit
exit
ubuntu@pg02:/etc/postgresql/15/main$
```

После этого успешно стартуем кластер. 
```bash
ubuntu@pg02:/etc/postgresql/15/main$ sudo systemctl start postgresql@15-main
ubuntu@pg02:/etc/postgresql/15/main$ sudo systemctl status postgresql@15-main
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Thu 2023-08-24 12:41:24 UTC; 6s ago
    Process: 37426 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=0/SUCCESS)
   Main PID: 37431 (postgres)
      Tasks: 6 (limit: 2219)
     Memory: 19.7M
        CPU: 156ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@15-main.service
             ├─37431 /usr/lib/postgresql/15/bin/postgres -D /mnt/data/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
             ├─37432 "postgres: 15/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─37433 "postgres: 15/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─37435 "postgres: 15/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─37436 "postgres: 15/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             └─37437 "postgres: 15/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >

Aug 24 12:41:22 pg02 systemd[1]: Starting PostgreSQL Cluster 15-main...
Aug 24 12:41:24 pg02 systemd[1]: Started PostgreSQL Cluster 15-main.
ubuntu@pg02:/etc/postgresql/15/main$
```

Далее заходим через psql и проверяем, что мы перенесли базу с таблицей с ВМ pg01 на pg02:
```bash
ubuntu@pg02:/etc/postgresql/15/main$ sudo -u postgres psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \q
ubuntu@pg02:/etc/postgresql/15/main$
```

## Отключаем диск от pg02, возвращаем его на pg01 и стартуем PostgreSQL с проверкой тестовой таблицы
```bash
ubuntu@pg02:/etc/postgresql/15/main$ sudo systemctl stop postgresql@15-main
ubuntu@pg02:/etc/postgresql/15/main$ exit
logout
Connection to 158.160.102.255 closed.
PS C:\Users\fvg00> yc compute instance detach-disk pg02 --disk-id fhmngkgocvosc5tq1sne
done (18s)
...
PS C:\Users\fvg00> yc compute instance attach-disk pg01 --disk-name disk2 --mode rw --auto-delete
done (14s)
...
PS C:\Users\fvg00> ssh ubuntu@51.250.76.81

ubuntu@pg01:~$ sudo mount -o defaults /dev/vdb1 /mnt/data
ubuntu@pg01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/vda2       9.8G  4.5G  4.9G  48% /
tmpfs           982M     0  982M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           197M  4.0K  197M   1% /run/user/1000
/dev/vdb1       9.8G   39M  9.2G   1% /mnt/data
ubuntu@pg01:~$ sudo systemctl start postgresql@15-main
ubuntu@pg01:~$ sudo -u postgres psql
could not change directory to "/home/ubuntu": Permission denied
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \q
ubuntu@pg01:~$
```
Работа завершена, удаляем ВМ.
