# Настройка autovacuum с учетом особенностей производительности

## Создаем ВМ pg01 (2 ядра, 4GB ОЗУ, SSD 10GB):
```bash
yc compute instance create --name pg01 --hostname pg01 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub --memory 4G
```
Подключаемся:
```bash
ssh ubuntu@158.160.50.36
```

## Устанавливаем PostgreSQL 15 и проверяем, что кластер работает:
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
ubuntu@pg01:~$
```

## Создаем дефолтную БД для тестов: 
```bash
ubuntu@pg01:~$ sudo su postgres
postgres@pg01:/home/ubuntu$
postgres@pg01:/home/ubuntu$
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
done in 1.34 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.15 s, vacuum 0.06 s, primary keys 1.11 s).
postgres@pg01:/home/ubuntu$
```
## Запускаем [тест](https://postgrespro.ru/docs/enterprise/15/pgbench):
```bash
postgres@pg01:/home/ubuntu$ pgbench -c 8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 481.3 tps, lat 16.541 ms stddev 9.931, 0 failed
progress: 12.0 s, 193.2 tps, lat 41.222 ms stddev 60.661, 0 failed
progress: 18.0 s, 427.7 tps, lat 18.754 ms stddev 12.325, 0 failed
progress: 24.0 s, 400.8 tps, lat 19.989 ms stddev 18.187, 0 failed
progress: 30.0 s, 369.2 tps, lat 21.606 ms stddev 21.347, 0 failed
progress: 36.0 s, 425.7 tps, lat 18.830 ms stddev 18.221, 0 failed
progress: 42.0 s, 322.0 tps, lat 24.852 ms stddev 19.867, 0 failed
progress: 48.0 s, 419.7 tps, lat 19.075 ms stddev 13.900, 0 failed
progress: 54.0 s, 435.5 tps, lat 18.371 ms stddev 19.098, 0 failed
progress: 60.0 s, 418.3 tps, lat 19.101 ms stddev 11.971, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 23368
number of failed transactions: 0 (0.000%)
latency average = 20.535 ms
latency stddev = 21.473 ms
initial connection time = 22.306 ms
tps = 389.485868 (without initial connection time)
postgres@pg01:/home/ubuntu$
```
## Настраиваем экземпляр, меняем параметры в /etc/postgresql/15/main/postgresql.conf, после этого делаем рестарт кластера:
max_connections = 40

shared_buffers = 1GB

effective_cache_size = 3GB

maintenance_work_mem = 512MB

checkpoint_completion_target = 0.9

wal_buffers = 16MB

default_statistics_target = 500

random_page_cost = 4

effective_io_concurrency = 2

work_mem = 6553kB

min_wal_size = 4GB

max_wal_size = 16GB
```bash
ubuntu@pg01:~$ sudo systemctl restart postgresql
```

## Снова запускаем тест:
```bash
postgres@pg01:/home/ubuntu$ pgbench -c 8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 581.7 tps, lat 13.683 ms stddev 7.908, 0 failed
progress: 12.0 s, 326.0 tps, lat 24.536 ms stddev 34.730, 0 failed
progress: 18.0 s, 545.5 tps, lat 14.512 ms stddev 12.428, 0 failed
progress: 24.0 s, 602.4 tps, lat 13.415 ms stddev 9.435, 0 failed
progress: 30.0 s, 361.5 tps, lat 22.127 ms stddev 30.357, 0 failed
progress: 36.0 s, 372.2 tps, lat 21.468 ms stddev 16.595, 0 failed
progress: 42.0 s, 306.8 tps, lat 26.062 ms stddev 33.842, 0 failed
progress: 48.0 s, 525.2 tps, lat 15.250 ms stddev 10.538, 0 failed
progress: 54.0 s, 497.7 tps, lat 16.078 ms stddev 10.737, 0 failed
progress: 60.0 s, 371.8 tps, lat 21.491 ms stddev 16.007, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26952
number of failed transactions: 0 (0.000%)
latency average = 17.807 ms
latency stddev = 19.170 ms
initial connection time = 23.622 ms
tps = 449.136095 (without initial connection time)
postgres@pg01:/home/ubuntu$
```
<span style="color:red"> **Получили прирост производительности операций в БД на 15.3%, на основании статистики транзакций. Если сравнить по "latency average", то получим такой же рост скорости операций на 15.3%.**</span>

Что мы изменили в параметрах, с целью более эффективного использования имеющихся ресурсов:
1. [shared_buffers](https://postgrespro.ru/docs/postgresql/15/runtime-config-resource#GUC-SHARED-BUFFERS) - выставили рекомендуемый начальный размер в 25% от объёма памяти. Задаёт объём памяти, который будет использовать сервер баз данных для буферов в разделяемой памяти. По умолчанию это обычно 128 мегабайт (128MB).
2. [effective_cache_size](https://postgrespro.ru/docs/postgresql/15/runtime-config-query#GUC-EFFECTIVE-CACHE-SIZE) - снизили размер до 3GB (стандартное значение - 4GB). Определяет представление планировщика об эффективном размере дискового кеша, доступном для одного запроса. Это представление влияет на оценку стоимости использования индекса; чем выше это значение, тем больше вероятность, что будет применяться сканирование по индексу, чем ниже, тем более вероятно, что будет выбрано последовательное сканирование.
3. [maintenance_work_mem](https://postgrespro.ru/docs/postgresql/15/runtime-config-resource#GUC-MAINTENANCE-WORK-MEM) - увеличили до 512MB (значение по умолчанию — 64 мегабайта). Задаёт максимальный объём памяти для операций обслуживания БД, в частности VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY. Таким образом, увеличив параметр конфигурации maintenance_work_mem, можно ускорить загрузку больших объёмов данных.
4. [max_wal_size](https://postgrespro.ru/docs/postgresql/15/runtime-config-wal#GUC-MAX-WAL-SIZE) и min_wal_size - были увеличены до 16GB(значение по умолчанию — 1GB) и 4GB(Значение по умолчанию — 80 MB) соответственно. Максимальный размер, до которого может вырастать WAL во время автоматических контрольных точек. Массовую загрузку данных можно ускорить, изменив на время загрузки параметр конфигурации max_wal_size. Загружая большие объёмы данных, PostgreSQL вынужден увеличивать частоту контрольных точек по сравнению с обычной (которая задаётся параметром checkpoint_timeout), а значит и чаще сбрасывать «грязные» страницы на диск. Временно увеличив max_wal_size, можно уменьшить частоту контрольных точек и связанных с ними операций ввода-вывода.
5. [checkpoint_completion_target](https://postgrespro.ru/docs/postgresql/15/runtime-config-wal#GUC-CHECKPOINT-COMPLETION-TARGET) - оставили значение по умолчанию 0.9. Задаёт целевое время для завершения процедуры контрольной точки, как долю общего времени между контрольными точками. Уменьшать значение этого параметра не рекомендуется. Чтобы избежать «заваливания» системы ввода/вывода при резкой интенсивной записи страниц, запись «грязных» буферов во время контрольной точки растягивается на определённый период времени. Этот период управляется параметром checkpoint_completion_target, который задаётся как часть интервала между контрольными точками (настраиваемого параметром checkpoint_timeout). Скорость ввода/вывода подстраивается так, чтобы контрольная точка завершилась к моменту истечения заданной части от checkpoint_timeout секунд или до превышения max_wal_size, если оно имеет место раньше. Со значением 0.9, заданным по умолчанию, можно ожидать, что PostgreSQL завершит процедуру контрольной точки незадолго до следующей запланированной (примерно на 90% выполнения предыдущей контрольной точки). При этом процесс ввода/вывода растягивается насколько возможно, чтобы обеспечить равномерную нагрузку ввода/вывода в течение интервала между контрольными точками.
6. [wal_buffers](https://postgrespro.ru/docs/postgresql/15/runtime-config-wal#GUC-WAL-BUFFERS) - задали максимальный размер 16MB. Содержимое буферов WAL записывается на диск при фиксировании каждой транзакции, так что очень большие значения вряд ли принесут значительную пользу. Однако значение как минимум в несколько мегабайт может увеличить быстродействие при записи на нагруженном сервере, когда сразу множество клиентов фиксируют транзакции (у нас в тесте 8 клиентов). Автонастройка, действующая при значении по умолчанию (-1), в большинстве случаев выбирает разумные значения.
7. [default_statistics_target](https://postgrespro.ru/docs/postgresql/15/runtime-config-query#GUC-DEFAULT-STATISTICS-TARGET) - увеличили до 500. Устанавливает значение ориентира статистики по умолчанию, распространяющееся на столбцы, для которых командой ALTER TABLE SET STATISTICS не заданы отдельные ограничения. Чем больше установленное значение, тем больше времени требуется для выполнения ANALYZE, но тем выше может быть качество оценок планировщика. Значение этого параметра по умолчанию — 100.
8. [random_page_cost](https://postgrespro.ru/docs/postgresql/15/runtime-config-query#GUC-RANDOM-PAGE-COST) - оставили значение по умолчанию 4.0. Задаёт приблизительную стоимость чтения одной произвольной страницы с диска. При уменьшении этого значения по отношению к seq_page_cost система начинает предпочитать сканирование по индексу; при увеличении такое сканирование становится более дорогостоящим. Оба эти значения также можно увеличить или уменьшить одновременно, чтобы изменить стоимость операций ввода/вывода по отношению к стоимости процессорных операций, которая определяется следующими параметрами.
Произвольный доступ к механическому дисковому хранилищу обычно гораздо дороже последовательного доступа, более чем в четыре раза. Однако по умолчанию выбран небольшой коэффициент (4.0), в предположении, что большой объём данных при произвольном доступе, например, при чтении индекса, окажется в кеше. Таким образом, можно считать, что значение по умолчанию моделирует ситуацию, когда произвольный доступ в 40 раз медленнее последовательного, но 90% операций произвольного чтения удовлетворяются из кеша.
Если вы считаете, что для вашей рабочей нагрузки процент попаданий не достигает 90%, вы можете увеличить параметр random_page_cost, чтобы он больше соответствовал реальной стоимости произвольного чтения. И напротив, если ваши данные могут полностью поместиться в кеше, например, когда размер базы меньше общего объёма памяти сервера, может иметь смысл уменьшить random_page_cost. С хранилищем, у которого стоимость произвольного чтения не намного выше последовательного, как например, у твердотельных накопителей, так же лучше выбрать меньшее значение random_page_cost, например 1.1.
9. [effective_io_concurrency](https://postgrespro.ru/docs/postgresql/15/runtime-config-resource#GUC-EFFECTIVE-IO-CONCURRENCY) - увеличили значение до 2. Значение по умолчанию равно 1 в системах, где это поддерживается, и 0 в остальных. Задаёт допустимое число параллельных операций ввода/вывода, которое говорит PostgreSQL о том, сколько операций ввода/вывода могут быть выполнены одновременно. Чем больше это число, тем больше операций ввода/вывода будет пытаться выполнить параллельно PostgreSQL в отдельном сеансе. Допустимые значения лежат в интервале от 1 до 1000, а нулевое значение отключает асинхронные запросы ввода/вывода. В настоящее время этот параметр влияет только на сканирование по битовой карте. Диски SSD и другие виды хранилища в памяти часто могут обрабатывать множество параллельных запросов, так что оптимальным числом может быть несколько сотен.


## Попробуем выставить effective_io_concurrency = 200, так как у нас SSD-диски и снова запускаем тест:
```bash
postgres@pg01:/home/ubuntu$ pgbench -c 8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 502.0 tps, lat 15.847 ms stddev 11.154, 0 failed
progress: 12.0 s, 462.5 tps, lat 17.306 ms stddev 11.797, 0 failed
progress: 18.0 s, 524.0 tps, lat 15.252 ms stddev 11.669, 0 failed
progress: 24.0 s, 388.2 tps, lat 20.623 ms stddev 22.199, 0 failed
progress: 30.0 s, 659.8 tps, lat 12.125 ms stddev 6.547, 0 failed
progress: 36.0 s, 558.8 tps, lat 14.310 ms stddev 21.856, 0 failed
progress: 42.0 s, 547.0 tps, lat 14.627 ms stddev 19.233, 0 failed
progress: 48.0 s, 534.3 tps, lat 14.915 ms stddev 11.385, 0 failed
progress: 54.0 s, 447.7 tps, lat 17.943 ms stddev 14.830, 0 failed
progress: 60.0 s, 563.0 tps, lat 14.196 ms stddev 10.036, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31132
number of failed transactions: 0 (0.000%)
latency average = 15.415 ms
latency stddev = 14.797 ms
initial connection time = 23.927 ms
tps = 518.853306 (without initial connection time)
postgres@pg01:/home/ubuntu$
```
<span style="color:red"> **Получили дополнительный прирост производительности операций в БД на 15.5% (или 33.2% к первоначальному тесту на дефолтной конфигурации кластера).**</span>

## Проводим тест работы автовакуум:
```bash
Создадим тестовую таблицу с полем типа TEXT:
postgres=# CREATE TABLE toast_test (id SERIAL, value TEXT);
CREATE TABLE

Создадим функцию для генерации строк заданной длины:
postgres=# CREATE OR REPLACE FUNCTION generate_random_string(
  length INTEGER,
  characters TEXT default '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
) RETURNS TEXT AS
$$
DECLARE
  result TEXT := '';
BEGIN
  IF length < 1 then
      RAISE EXCEPTION 'Invalid length';
  END IF;
  FOR __ IN 1..length LOOP
    result := result || substr(characters, floor(random() * length(characters))::int + 1, 1);
  end loop;
  RETURN result;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=#

вставляем 1 млн. записей, в каждой записи генерируется строка размером 200 символов:
postgres=# WITH str AS (SELECT generate_random_string(200) AS value)
INSERT INTO toast_test (value)
SELECT value
FROM generate_series(1, 1000000), str;
INSERT 0 1000000

смотрим размер таблицы:
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              241033216
(1 row)

Добавим к каждой строке в поле value символ '1':
postgres=#
postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=#
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              489340928
(1 row)

Добавим к каждой строке в поле value символ '2':
postgres=# update toast_test set value = value || '2';
UPDATE 1000000

Посмотрим "мертвых":
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'toast_test';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 toast_test |    1399911 |          0 |      0 | 2023-08-30 18:59:07.368175+00
(1 row)

смотрим размер таблицы:
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              737665024
(1 row)

postgres=# select count(id) from toast_test;
  count
---------
 1000000
(1 row)

Добавим к каждой строке в поле value символ '3':
postgres=# update toast_test set value = value || '3';
UPDATE 1000000
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              737665024
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'toast_test';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 toast_test |    1399911 |    1000000 |     71 | 2023-08-30 18:59:07.368175+00
(1 row)

Добавим к каждой строке в поле value символ '4':
postgres=# update toast_test set value = value || '4';
UPDATE 1000000
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'toast_test';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 toast_test |    1059558 |       1506 |      0 | 2023-08-30 19:02:38.365585+00
(1 row)

postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              737665024
(1 row)

Добавим к каждой строке в поле value символ '5':
postgres=# update toast_test set value = value || '5';
UPDATE 1000000
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              745054208
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'toast_test';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 toast_test |    1000000 |          0 |      0 | 2023-08-30 19:04:08.278586+00
(1 row)

postgres=#
```
<span style="color:red"> **Видим, что автовакуум справляется со своей задачей - почистил удаленные записи, которые возникают в процессе выполнения UPDATE, при этом размер таблицы увеличился примерно в 3 раза, но после 2-го апдейта он практически уже не рос.**</span>

## Попробуем отключить автовакуум на нашей таблице:
```bash
postgres=# ALTER TABLE toast_test  SET (autovacuum_enabled = off);
ALTER TABLE
postgres=#

теперь выполним 3 раза операции UPDATE всех записей, каждый раз добавляя символ в строку:

postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              745054208
(1 row)
postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              993239040
(1 row)

Видим, что пока размер вырос, но не сильно, попробуем еще повторить 3 раза:

postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
             1505337344
(1 row)

postgres=# update toast_test set value = value || '1';
UPDATE 1000000
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
             1761378304
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'toast_test';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 toast_test |    1000000 |    5999763 |    599 | 2023-08-30 19:04:08.278586+00
(1 row)

postgres=#
```

<span style="color:red"> **На 5 и 6 UPDATE время выполнения выросло и размер таблицы стал сильно увеличиваться - в итоге достигли 1.7 GB. Запрос "мертвых" записей дает нам объяснение этого роста - после 6-ти UPDATE получили 6 млн. n_dead_tup.**</span>

## Попробуем почистить таблицу, чтобы вернуть первоначальный размер до серии UPDATE:
```bash
postgres=# VACUUM FULL VERBOSE toast_test;
INFO:  vacuuming "public.toast_test"
INFO:  "public.toast_test": found 1000063 removable, 1000000 nonremovable row versions in 214953 pages
DETAIL:  0 dead row versions cannot be removed yet.
CPU: user: 0.88 s, system: 0.79 s, elapsed: 57.68 s.
VACUUM

Смотрим размер - сжало хорошо, почти чтолько же, как после начального INSERT (241033216)
postgres=# select pg_total_relation_size('toast_test');
 pg_total_relation_size
------------------------
              256008192
(1 row)

postgres=# ANALYZE toast_test;
ANALYZE
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'toast_test';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 toast_test |    1000000 |          0 |      0 | 2023-08-30 19:04:08.278586+00
(1 row)

Включаем обратно автовакуум:
postgres=# ALTER TABLE toast_test  SET (autovacuum_enabled = on);
ALTER TABLE
postgres=# SELECT pg_size_pretty(pg_total_relation_size('toast_test'));
 pg_size_pretty
----------------
 244 MB
(1 row)

postgres=#
```

## Задание - написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице, не забыть вывести номер шага цикла.

```bash
Создадим новую таблицу со строкой меньшего размера, чем в предыдущем примере, вставим туда 1 млн. записей со строкой длиной 20 символов:
postgres=# CREATE TABLE test2 (id SERIAL, value VARCHAR(50));
CREATE TABLE
postgres=# WITH str AS (SELECT generate_random_string(20) AS value)
INSERT INTO test2 (value)
SELECT value
FROM generate_series(1, 1000000), str;
INSERT 0 1000000
postgres=# select id,value from test2 limit 10;
 id |        value
----+----------------------
  1 | oCAQGiI1oJnwNVhFhF8x
  2 | oCAQGiI1oJnwNVhFhF8x
  3 | oCAQGiI1oJnwNVhFhF8x
  4 | oCAQGiI1oJnwNVhFhF8x
  5 | oCAQGiI1oJnwNVhFhF8x
  6 | oCAQGiI1oJnwNVhFhF8x
  7 | oCAQGiI1oJnwNVhFhF8x
  8 | oCAQGiI1oJnwNVhFhF8x
  9 | oCAQGiI1oJnwNVhFhF8x
 10 | oCAQGiI1oJnwNVhFhF8x
(10 rows)

Проверяем размер таблицы и наличие dead tuples:
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test2'));
 pg_size_pretty
----------------
 57 MB
(1 row)

postgres=#  SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test2';
 relname | n_live_tup | n_dead_tup | ratio% | last_autovacuum
---------+------------+------------+--------+-----------------
 test2   |    1000000 |          0 |      0 |
(1 row)

Запускаем анонимный pgplsql блок, выполняющий в цикле 10 раз апдейт указанной таблицы, добавляя каждый раз в поле VALUE номер цикла, в процессе выводится время выполнения каждого цикла:
postgres=# do
$$
declare
    i integer;
    t timestamp;
    tnew timestamp;
begin
    t := clock_timestamp();
    raise info 'цикл запущен в %', t;
    for i in 1..10
    loop
      update test2 set value = value || i::text;
      tnew := clock_timestamp();
      raise info 'update #% выполнился за %', i, tnew-t;
      t := tnew;
    end loop;
    raise info 'цикл завершен в %', clock_timestamp();
end;
$$;
INFO:  цикл запущен в 2023-08-31 08:10:26.542234
INFO:  update #1 выполнился за 00:00:09.262548
INFO:  update #2 выполнился за 00:00:13.678804
INFO:  update #3 выполнился за 00:00:14.460841
INFO:  update #4 выполнился за 00:00:21.864362
INFO:  update #5 выполнился за 00:00:25.290173
INFO:  update #6 выполнился за 00:00:14.235902
INFO:  update #7 выполнился за 00:00:19.787662
INFO:  update #8 выполнился за 00:00:26.916755
INFO:  update #9 выполнился за 00:00:10.805646
INFO:  update #10 выполнился за 00:00:09.909071
INFO:  цикл завершен в 2023-08-31 08:13:12.75416+00
DO

смотрим результат, что апдейт отработал:
postgres=# select id,value from test2 limit 10;
 id |              value
----+---------------------------------
  1 | oCAQGiI1oJnwNVhFhF8x12345678910
  2 | oCAQGiI1oJnwNVhFhF8x12345678910
  3 | oCAQGiI1oJnwNVhFhF8x12345678910
  4 | oCAQGiI1oJnwNVhFhF8x12345678910
  5 | oCAQGiI1oJnwNVhFhF8x12345678910
  6 | oCAQGiI1oJnwNVhFhF8x12345678910
  7 | oCAQGiI1oJnwNVhFhF8x12345678910
  8 | oCAQGiI1oJnwNVhFhF8x12345678910
  9 | oCAQGiI1oJnwNVhFhF8x12345678910
 10 | oCAQGiI1oJnwNVhFhF8x12345678910
(10 rows)

размер таблицы увеличился в 11.5 раз:
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test2'));
 pg_size_pretty
----------------
 655 MB
(1 row)

автовакуум отработал быстро:
postgres=#  SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test2';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test2   |    1000000 |          0 |      0 | 2023-08-31 08:13:31.479311+00
(1 row)

postgres=#

попробуем вернуть начальный размер таблицы:
postgres=# VACUUM FULL VERBOSE test2;
INFO:  vacuuming "public.test2"
INFO:  "public.test2": found 0 removable, 1000000 nonremovable row versions in 83824 pages
DETAIL:  0 dead row versions cannot be removed yet.
CPU: user: 0.31 s, system: 0.08 s, elapsed: 7.19 s.
VACUUM
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test2'));
 pg_size_pretty
----------------
 65 MB
(1 row)

postgres=#
```
<span style="color:red"> **Задача выполнена. Удаляем ВМ.**</span>
