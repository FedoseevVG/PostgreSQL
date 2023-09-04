# Работа с журналами

## Создаем ВМ pg01 (2 ядра, 4GB ОЗУ, SSD 10GB):
```bash
yc compute instance create --name pg01 --hostname pg01 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub --memory 4G
```
Подключаемся:
```bash
ssh ubuntu@51.250.10.2
```

## Устанавливаем PostgreSQL 15 и проверяем, что кластер работает:
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
ubuntu@pg01:~$
```

## Готовим БД для тестов:
```bash
ubuntu@pg01:~$ sudo su postgres
postgres@pg01:/home/ubuntu$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.22 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.11 s, vacuum 0.05 s, primary keys 1.05 s).
postgres@pg01:/home/ubuntu$

Создадим расширение для просмотра кеша:
CREATE EXTENSION pg_buffercache; 

Создадим расширение pageinspect. Модуль pageinspect предоставляет функции, позволяющие исследовать страницы баз данных на низком уровне:
CREATE EXTENSION pageinspect;

Проверяем параметры журнала и контрольной точки:
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)

postgres=# show checkpoint_warning;
 checkpoint_warning
--------------------
 30s
(1 row)

postgres=# show log_checkpoints;
 log_checkpoints
-----------------
 on
(1 row)

postgres=# show checkpoint_completion_target;
 checkpoint_completion_target
------------------------------
 0.9
(1 row)

postgres=# show wal_level;
 wal_level
-----------
 replica
(1 row)

Сервер хранит журнальные файлы, необходимые для восстановления (обычно < max_wal_size), еще не прочитанные через слоты репликации, еще не записанные в архив, 
если настроена непрерывная архивация, не превышающие по объему wal_keep_size, не превышающие по объему min_wal_size (при переиспользовании).
Настройки по умолчанию:
max_wal_size = 1GB
wal_keep_size = 0
wal_recycle = on
min_wal_size = 80MB

Установим выполнение контрольной точки раз в 30 секунд:

postgres=# ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

## Запускаем нагрузку в течение 10-ти минут:
```bash
Cбрасываем статистику:
postgres=# SELECT pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared
----------------------
(1 row)

Запоминаем позицию в журнале:
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/330C9678
(1 row)

postgres=# SELECT pg_walfile_name('0/330C9678');
     pg_walfile_name
--------------------------
 000000010000000000000033
(1 row)


Тест:
postgres@pg01:/home/ubuntu$ pgbench -P 30 -T 600
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 328.6 tps, lat 3.041 ms stddev 4.506, 0 failed
progress: 60.0 s, 292.8 tps, lat 3.416 ms stddev 5.139, 0 failed
progress: 90.0 s, 304.8 tps, lat 3.280 ms stddev 4.861, 0 failed
progress: 120.0 s, 285.3 tps, lat 3.504 ms stddev 4.789, 0 failed
progress: 150.0 s, 320.7 tps, lat 3.118 ms stddev 5.047, 0 failed
progress: 180.0 s, 328.7 tps, lat 3.041 ms stddev 3.861, 0 failed
progress: 210.0 s, 286.8 tps, lat 3.486 ms stddev 4.610, 0 failed
progress: 240.0 s, 280.4 tps, lat 3.566 ms stddev 4.971, 0 failed
progress: 270.0 s, 310.7 tps, lat 3.217 ms stddev 3.658, 0 failed
progress: 300.0 s, 286.3 tps, lat 3.493 ms stddev 4.769, 0 failed
progress: 330.0 s, 311.9 tps, lat 3.206 ms stddev 4.583, 0 failed
progress: 360.0 s, 316.6 tps, lat 3.158 ms stddev 4.499, 0 failed
progress: 390.0 s, 258.6 tps, lat 3.867 ms stddev 5.277, 0 failed
progress: 420.0 s, 312.5 tps, lat 3.199 ms stddev 4.289, 0 failed
progress: 450.0 s, 346.3 tps, lat 2.887 ms stddev 4.378, 0 failed
progress: 480.0 s, 272.9 tps, lat 3.663 ms stddev 4.507, 0 failed
progress: 510.0 s, 300.8 tps, lat 3.324 ms stddev 5.233, 0 failed
progress: 540.0 s, 301.9 tps, lat 3.310 ms stddev 4.426, 0 failed
progress: 570.0 s, 352.2 tps, lat 2.840 ms stddev 4.420, 0 failed
progress: 600.0 s, 296.7 tps, lat 3.370 ms stddev 4.469, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 182869
number of failed transactions: 0 (0.000%)
latency average = 3.280 ms
latency stddev = 4.624 ms
initial connection time = 4.165 ms
tps = 304.782616 (without initial connection time)
postgres@pg01:/home/ubuntu$


В логе БД пишется информация о контрольных точках:
...
2023-09-04 08:44:54.164 UTC [27586] LOG:  checkpoint starting: time
2023-09-04 08:45:21.057 UTC [27586] LOG:  checkpoint complete: wrote 1667 buffers (1.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.777 s, sync=0.011 s, total=26.894 s; sync files=6, longest=0.010 s, average=0.002 s; distance=17036 kB, estimate=18069 kB
2023-09-04 08:45:24.060 UTC [27586] LOG:  checkpoint starting: time
2023-09-04 08:45:51.105 UTC [27586] LOG:  checkpoint complete: wrote 1779 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.982 s, sync=0.010 s, total=27.046 s; sync files=10, longest=0.010 s, average=0.001 s; distance=16281 kB, estimate=17890 kB
2023-09-04 08:45:54.108 UTC [27586] LOG:  checkpoint starting: time
2023-09-04 08:46:21.076 UTC [27586] LOG:  checkpoint complete: wrote 1739 buffers (1.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.794 s, sync=0.043 s, total=26.968 s; sync files=6, longest=0.041 s, average=0.008 s; distance=17024 kB, estimate=17804 kB
2023-09-04 08:46:24.079 UTC [27586] LOG:  checkpoint starting: time
2023-09-04 08:46:51.107 UTC [27586] LOG:  checkpoint complete: wrote 1958 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.920 s, sync=0.015 s, total=27.028 s; sync files=11, longest=0.015 s, average=0.002 s; distance=17545 kB, estimate=17778 kB
2023-09-04 08:46:54.109 UTC [27586] LOG:  checkpoint starting: time
2023-09-04 08:47:21.348 UTC [27586] LOG:  checkpoint complete: wrote 1722 buffers (1.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.880 s, sync=0.221 s, total=27.240 s; sync files=6, longest=0.185 s, average=0.037 s; distance=16507 kB, estimate=17651 kB
...

Берем текущую позицию в журнале:
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/485F5210
(1 row)

postgres=# SELECT pg_walfile_name('0/485F5210');
     pg_walfile_name
--------------------------
 000000010000000000000048
(1 row)

Объем журнальных файлов получаем на основе полученных данных о позиции в журнеле:
postgres=# SELECT pg_size_pretty('0/485F5210'::pg_lsn - '0/330C9678'::pg_lsn);
 pg_size_pretty
----------------
 341 MB
(1 row)

Смотрим статистику, описание:
checkpoints_timed - количество запланированных контрольных точек, которые уже были выполнены
checkpoints_req - количество запрошенных контрольных точек, которые уже были выполнены

postgres=# SELECT checkpoints_timed, checkpoints_req FROM pg_stat_bgwriter;
 checkpoints_timed | checkpoints_req
-------------------+-----------------
                37 |               0
(1 row)

```
<span style="color:red"> **Получили объем журнальных файлов - 341 MB, в среднем на одну контрольную точку - 17459 kB, все контрольные точки выполнились по расписанию, что подтверждается статистикой в логе. 
Это произошло, потому что размер журнальных файлов и частота выполнения контрольных точек были достаточными для указанного теста, чтобы для успевать сбрасывать изменения на диск.**</span>

## Попробуем нагрузочное тестирование в асинхронном режиме:
```bash
В предыдущем тесте данные tps = 304.782616 были получены в синхронном режиме:
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

Выставим асинхронный режим:
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/485F5210
(1 row)

Пробуем повторить тест, аналогичный предыдущему:
postgres@pg01:/home/ubuntu$ pgbench -P 30 -T 600
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 1741.9 tps, lat 0.573 ms stddev 0.281, 0 failed
progress: 60.0 s, 1731.3 tps, lat 0.577 ms stddev 0.091, 0 failed
progress: 90.0 s, 1737.4 tps, lat 0.575 ms stddev 0.066, 0 failed
progress: 120.0 s, 1745.0 tps, lat 0.573 ms stddev 0.053, 0 failed
progress: 150.0 s, 1761.2 tps, lat 0.567 ms stddev 0.089, 0 failed
progress: 180.0 s, 1754.6 tps, lat 0.569 ms stddev 0.089, 0 failed
progress: 210.0 s, 1743.0 tps, lat 0.573 ms stddev 0.097, 0 failed
progress: 240.0 s, 1736.6 tps, lat 0.575 ms stddev 0.079, 0 failed
progress: 270.0 s, 1748.4 tps, lat 0.571 ms stddev 0.142, 0 failed
progress: 300.0 s, 1737.5 tps, lat 0.575 ms stddev 0.113, 0 failed
progress: 330.0 s, 1729.2 tps, lat 0.578 ms stddev 0.720, 0 failed
progress: 360.0 s, 1744.9 tps, lat 0.573 ms stddev 0.071, 0 failed
progress: 390.0 s, 1744.4 tps, lat 0.573 ms stddev 0.080, 0 failed
progress: 420.0 s, 1736.2 tps, lat 0.575 ms stddev 0.437, 0 failed
progress: 450.0 s, 1734.7 tps, lat 0.576 ms stddev 0.102, 0 failed
progress: 480.0 s, 1726.9 tps, lat 0.579 ms stddev 0.550, 0 failed
progress: 510.0 s, 1731.3 tps, lat 0.577 ms stddev 0.126, 0 failed
progress: 540.0 s, 1746.9 tps, lat 0.572 ms stddev 0.047, 0 failed
progress: 570.0 s, 1735.0 tps, lat 0.576 ms stddev 0.412, 0 failed
progress: 600.0 s, 1720.8 tps, lat 0.581 ms stddev 0.649, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1043625
number of failed transactions: 0 (0.000%)
latency average = 0.574 ms
latency stddev = 0.299 ms
initial connection time = 4.114 ms
tps = 1739.384575 (without initial connection time)
postgres@pg01:/home/ubuntu$

postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/72A4C510
(1 row)

Объем журнальных файлов:
postgres=# SELECT pg_size_pretty('0/72A4C510'::pg_lsn - '0/485F5210'::pg_lsn);
 pg_size_pretty
----------------
 676 MB
(1 row)

```
<span style="color:red"> **Получили объем журнальных файлов примерно в 2 раза больше - 676 MB. Число транзакций в секунду увеличилось почти в 6 раз tps = 1739.384575. Это влияние "synchronous_commit = off".
Режим подтверждения транзакций управляется параметром synchronous_commit. Асинхронная фиксация — это возможность завершать транзакции быстрее, ценой того, что в случае краха СУБД последние транзакции могут быть потеряны. Для многих приложений такой компромисс приемлем.
Подтверждение транзакции обычно синхронное: сервер ждёт сохранения записей WAL транзакции в постоянном хранилище, прежде чем сообщить клиенту об успешном завершении. Таким образом, клиенту гарантируется, что транзакция, которую подтвердил сервер, будет защищена, даже если сразу после этого произойдёт крах сервера. Однако для коротких транзакций данная задержка будет основной составляющей общего времени транзакции. 
В режиме асинхронного подтверждения сервер сообщает об успешном завершении сразу, как только транзакция будет завершена логически, прежде чем сгенерированные записи WAL фактически будут записаны на диск. Это может значительно увеличить производительность при выполнении небольших транзакций. Если во время окна риска между асинхронным подтверждением транзакции и сохранением на диск записей WAL, происходит крах СУБД, то изменения, сделанные во время этой транзакции будут потеряны. Продолжительность окна риска ограничена, потому что фоновый процесс («WAL writer»), сохраняет не записанные записи WAL на диск каждые wal_writer_delay миллисекунд. Фактически, максимальная продолжительность окна риска составляет трёхкратное значение wal_writer_delay, потому что WAL writer разработан так, чтобы сразу сохранять целые страницы во время периодов занятости.
commit_delay также выглядит очень похоже на асинхронное подтверждение транзакций, но по сути это является методом синхронного подтверждения транзакций (фактически, во время асинхронных транзакций commit_delay игнорируется). commit_delay приводит к задержке только перед тем, как синхронная транзакция пытается записать данные WAL на диск, в надежде, что одиночная запись, выполняемая на одну такую транзакцию, сможет также обслужить другие транзакции, которые подтверждаются приблизительно в это же время. Установку этого параметра можно рассматривать как способ увеличения промежутка времени, в течение которого транзакции группируются для единовременной записи на диск. Это распределяет стоимость записи между несколькими транзакциями.**</span>


## Проводим тест кластера с включенной контрольной суммой страниц:
```bash
Останавливаем кластер:
ubuntu@pg01:~$ sudo systemctl stop postgresql
ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

включаем контрольные суммы:
ubuntu@pg01:~$ sudo su - postgres -c '/usr/lib/postgresql/15/bin/pg_checksums --enable -D "/var/lib/postgresql/15/main"'
Checksum operation completed
Files scanned:   961
Blocks scanned:  11474
Files written:  793
Blocks written: 11474
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster

Запускаем кластер:
ubuntu@pg01:~$ sudo systemctl start postgresql
ubuntu@pg01:~$ sudo su postgres
postgres@pg01:/home/ubuntu$ psql
could not change directory to "/home/ubuntu": Permission denied
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# SELECT datname, checksum_failures, checksum_last_failure FROM pg_stat_database WHERE datname IS NOT NULL;
  datname  | checksum_failures | checksum_last_failure
-----------+-------------------+-----------------------
 postgres  |                 0 |
 template1 |                 0 |
 template0 |                 0 |
(3 rows)

Создаем тестовую таблицу с данными:
postgres=# CREATE TABLE test1 (id SERIAL, val VARCHAR(10));
CREATE TABLE
postgres=# INSERT INTO test1(val) SELECT 'test '||s.id FROM generate_series(1,10) AS s(id);
INSERT 0 10
postgres=# select * from test1;
 id |   val
----+---------
  1 | test 1
  2 | test 2
  3 | test 3
  4 | test 4
  5 | test 5
  6 | test 6
  7 | test 7
  8 | test 8
  9 | test 9
 10 | test 10
(10 rows)

Смотрим размещение таблицы:
postgres=# SELECT pg_relation_filepath('test1');
 pg_relation_filepath
----------------------
 base/5/16491
(1 row)

postgres=#


Остановим сервер и поменяем несколько байтов в странице (сотрем из заголовка LSN последней журнальной записи):
ubuntu@pg01:~$ sudo systemctl stop postgresql
ubuntu@pg01:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/main/base/5/16491 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00791831 s, 1.0 kB/s
ubuntu@pg01:~$

Запускаем и пробуем сделать выборку из таблицы:
ubuntu@pg01:~$ sudo systemctl start postgresql
ubuntu@pg01:~$ sudo su postgres
postgres@pg01:/home/ubuntu$ psql
could not change directory to "/home/ubuntu": Permission denied
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test1;
WARNING:  page verification failed, calculated checksum 7133 but expected 46502
ERROR:  invalid page in block 0 of relation base/5/16491
postgres=#

В логе БД:
2023-09-04 10:51:11.703 UTC [28626] LOG:  database system is ready to accept connections
2023-09-04 10:51:39.686 UTC [28654] postgres@postgres WARNING:  page verification failed, calculated checksum 7133 but expected 46502
2023-09-04 10:51:39.686 UTC [28654] postgres@postgres ERROR:  invalid page in block 0 of relation base/5/16491
2023-09-04 10:51:39.686 UTC [28654] postgres@postgres STATEMENT:  select * from test1;
2023-09-04 10:51:41.719 UTC [28627] LOG:  checkpoint starting: time
2023-09-04 10:51:41.739 UTC [28627] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.007 s, sync=0.002 s, total=0.021 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB
```

Параметр [ignore_checksum_failure](https://postgrespro.ru/docs/postgrespro/9.5/runtime-config-developer#guc-ignore-checksum-failure) позволяет попробовать прочитать таблицу:
```bash
postgres=# SET ignore_checksum_failure = on;
SET
postgres=# select * from test1;
WARNING:  page verification failed, calculated checksum 7133 but expected 46502
 id |   val
----+---------
  1 | test 1
  2 | test 2
  3 | test 3
  4 | test 4
  5 | test 5
  6 | test 6
  7 | test 7
  8 | test 8
  9 | test 9
 10 | test 10
(10 rows)

Удалим таблицу:
postgres=# drop table test1;
DROP TABLE
postgres=#

```


<span style="color:red"> **ignore_checksum_failure (boolean)**</span>

<span style="color:red"> **Этот параметр действует, только если включён data checksums.**</span>

<span style="color:red"> **При обнаружении ошибок контрольных сумм при чтении Postgres Pro обычно сообщает об ошибке и прерывает текущую транзакцию. Если параметр ignore_checksum_failure включён, система игнорирует проблему (но всё же предупреждает о ней) и продолжает обработку. Это поведение может привести к краху, распространению или сокрытию повреждения данных и другим серьёзными проблемам. Однако, включив его, вы можете обойти ошибку и получить неповреждённые данные, которые могут находиться в таблице, если цел заголовок блока. Если же повреждён заголовок, будет выдана ошибка, даже когда этот параметр включён. По умолчанию этот параметр отключён (имеет значение off) и изменить его состояние может только суперпользователь.**</span>