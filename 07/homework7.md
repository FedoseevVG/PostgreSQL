# Логический уровень PostgreSQL

## Создаем ВМ pg01
```bash
yc compute instance create --name pg01 --hostname pg01 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub
```
Подключаемся:
```bash
ssh ubuntu@51.250.79.62
```

## Устанавливаем PostgreSQL 14 и проверяем, что кластер работает:
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
ubuntu@pg01:~$
```

## Выполняем задание

### Заходим под postgres в psql и создаем БД testdb, схему testnm, таблицу t1 с одной колонкой c1 типа integer и вставляем строку со значением c1=1:
```bash
ubuntu@pg01:~$ sudo -u postgres psql
could not change directory to "/home/ubuntu": Permission denied
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA

testdb=# \dn
  List of schemas
  Name  |  Owner
--------+----------
 public | postgres
 testnm | postgres
(2 rows)

testdb=# CREATE TABLE testnm.t1
(
    c1 integer PRIMARY KEY
);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 (c1) VALUES (1);
INSERT 0 1
testdb=#
```

### создаем новую роль readonly:
```bash
testdb=# CREATE ROLE readonly;
CREATE ROLE
```
даем readonly право на подключение к базе данных testdb:
```bash
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
```
даем readonly право на использование схемы testnm:
```bash
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
```
даем readonly право на select для всех таблиц схемы testnm:
```bash
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=#
```

### Создаем пользователя testread с паролем test123 и даем ему роль readonly:
```bash
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=# GRANT readonly TO testread;
GRANT ROLE
testdb=#
```
###  Подключаемся под testread в базу данных testdb и делаем SELECT из t1:
```bash
ubuntu@pg01:~$ sudo -u postgres psql -U testread -h localhost -W -d testdb
Password:
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```
Так как таблица создавалась изначально в схеме testnm, то проблем не возникло.

### Подключимся снова под postgres и пересоздадим таблицу:
```bash
ubuntu@pg01:~$ sudo -u postgres psql
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=# drop table testnm.t1;
DROP TABLE

testdb=# CREATE TABLE testnm.t1
(
    c1 integer PRIMARY KEY
);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 (c1) VALUES (1);
INSERT 0 1
testdb=#
```

### Снова подключаемся под testread в базу данных testdb и пробуем SELECT из t1:
```bash
ubuntu@pg01:~$ sudo -u postgres psql -U testread -h localhost -W -d testdb
Password:
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
testdb=>
```
<span style="color:red"> **Получили ошибку "ERROR:  permission denied", потому что "GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly" дал доступ только для существующих на тот момент времени таблиц, а t1 пересоздавалась.**</span>

### Пробуем зафиксировать под postgres права на SELECT и пересоздать таблицу 
```bash
postgres=# \c testdb postgres;
You are now connected to database "testdb" as user "postgres".
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES

testdb=# drop table testnm.t1;
DROP TABLE

testdb=# CREATE TABLE testnm.t1
(
    c1 integer PRIMARY KEY
);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 (c1) VALUES (1);
INSERT 0 1
testdb=# \q
ubuntu@pg01:~$
```
### Снова подключаемся под testread в базу данных testdb и пробуем SELECT из t1:
```bash
ubuntu@pg01:~$ sudo -u postgres psql -U testread -h localhost -W -d testdb
Password:
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```
<span style="color:red"> **Результат был получен - права на SELECT bp новой таблицы в схеме testnm для пользователя testread выдались через роль readonly**</span>

### Пробуем под testread создать новую таблицу:
```bash
testdb=> create table t2(c1 integer);
CREATE TABLE
testdb=> insert into t2 values (2);
INSERT 0 1
testdb=> select * from t2;
 c1
----
  2
(1 row)

testdb=>
```
<span style="color:red"> **Причиной отсутствия ошибки является то, что мы создали таблицу в схеме PUBLIC, которая в SEARCH_PATH идет первой по умолчанию. А права на все действия в этой схеме даются роли public, которая, в свою очередь, добавляется всем новым пользователям.**</span>

### Заходим под postgres и забираем права у схемы PUBLIC на БД testdb:
```bash
postgres=# \c testdb postgres;
You are now connected to database "testdb" as user "postgres".
testdb=# REVOKE CREATE on SCHEMA public FROM public;
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE
testdb=#
```
### Снова пробуем под testread создать новую таблицу в testdb:
```bash
testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
testdb=>
```
<span style="color:red"> **Текст ошибки "ERROR:  permission denied for schema public" наглядно показывает результат наших действий по отбору прав.**</span>
