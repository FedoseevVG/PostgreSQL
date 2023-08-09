# Работа с уровнями изоляции транзакции в postgresql

## Создание виртуальной машины

Предварительно я зарегистрировался в Yandex Cloud и создал приватное облако. 
Все действия выполнялись на ноутбуке под ОС Windows 11.

### Устанавливаем [Yandex Cli](https://cloud.yandex.ru/docs/cli/operations/install-cli#interactive) 

Запускаем Windows Powershell и работаем в нем, первоначальная установка YC:
```bash
PS D:\otus\Postgresql\yc> iex (New-Object System.Net.WebClient).DownloadString('https://storage.yandexcloud.net/yandexcloud-yc/install.ps1')
Downloading yc 0.108.1
Yandex Cloud CLI 0.108.1 windows/amd64
Now we have zsh completion. Type "echo 'source C:\Users\fvg00\yandex-cloud\completion.zsh.inc' >>  ~/.zshrc" to install itAdd yc installation dir to your PATH? [Y/n]: y
PS D:\otus\Postgresql\yc>
```
Далее выполняем первоначальный запуск YC (в процессе берем и подставляем ключ по ссылке):
```bash
PS D:\otus\Postgresql\yc> yc init
Welcome! This command will take you through the configuration process.
Please go to https://oauth.yandex.ru/authorize?response_type=token&client_id=1a6990aa636648e9b2ef855fa7bec2fb in order to obtain OAuth token.

Please enter OAuth token: y0_AgAAAAAaWzz5AATuwQAAAADpxDurNY_JLTUdQc-rxihT8BvUHtyBeTU
You have one cloud available: 'cloud-vf1' (id = b1gs02o04r4o0aq927n3). It is going to be used by default.
Please choose folder to use:
 [1] default (id = b1g17etduc7a8cals26t)
 [2] Create a new folder
Please enter your numeric choice: 1
Your current folder has been set to 'default' (id = b1g17etduc7a8cals26t).
Do you want to configure a default Compute zone? [Y/n] y
Which zone do you want to use as a profile default?
 [1] ru-central1-a
 [2] ru-central1-b
 [3] ru-central1-c
 [4] Don't set default zone
Please enter your numeric choice: 1
Your profile default Compute zone has been set to 'ru-central1-a'.
PS D:\otus\Postgresql\yc>
```
### Генерим [ssh-key](https://github.com/yandex-cloud/docs/blob/master/ru/compute/operations/serial-console/index.md)

```bash
PS D:\otus\Postgresql\yc> ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (C:\Users\fvg00/.ssh/id_ed25519):
...
The key fingerprint is:
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICSHaiylrjT2WGBfP8e7pXGBNqDcVBAWrAtOT1nqlic6 fvg00@NBH
The key's randomart image is:
+--[ED25519 256]--+
|          +=+o.+=|
|         o oo...*|
|        . o   .= |
|         .    o+*|
|        S     =O#|
|         o . +o@@|
|          o . +==|
|             ..oE|
|              oo.|
+----[SHA256]-----+
PS D:\otus\Postgresql\yc>
```
Полученный public key редактируем, добавив впереди имя windows пользователя fvg00 через двоеточие и сохраняем в файл fvg00.pub, далее укажем его при создании ВМ:
```bash
fvg00:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICSHaiylrjT2WGBfP8e7pXGBNqDcVBAWrAtOT1nqlic6 fvg00@NBH
```
### Создаем ВМ Pg02
```bash
S D:\otus\Postgresql\yc> yc compute instance create --name pg02 --hostname pg02 --create-boot-disk size=15G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub
  
done (57s)
id: fhmr0cu4aq1v4prajpoj
folder_id: b1g17etduc7a8cals26t
created_at: "2023-08-08T18:22:12Z"
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
  device_name: fhm961glgtel4a05mkq3
  auto_delete: true
  disk_id: fhm961glgtel4a05mkq3
network_interfaces:
  - index: "0"
    mac_address: d0:0d:1b:03:3c:45
    subnet_id: e9bkc3iu7l84ksblvk6r
    primary_v4_address:
      address: 10.128.0.8
      one_to_one_nat:
        address: 158.160.60.135
        ip_version: IPV4
gpu_settings: {}
fqdn: pg02.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
Далее подключаемся под ubuntu, так как bug YC не дает вход под пользоваталем, указанным в ssh-key:
```bash
PS D:\otus\Postgresql\yc> ssh ubuntu@158.160.60.135
ubuntu@pg02:~$ 
```


## Установка PostgreSQL 15

Сразу ставим PostgreSQL 15 вместе с обновлением репозитория ОС одной командой (заняло около 10 минут).

Ключ DEBIAN_FRONTEND=noninteractive - чтобы интерактивно не отвечать на обновление компонентов.
```bash
ubuntu@pg02:~$ sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
После окончания проверяем установленный PostgreSQL и запускаем psql:
```bash
postgres@pg02:~$ sudo su postgres
postgres@pg02:~$ cd
postgres@pg02:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
postgres@pg02:~$ psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=#
```

## Выполнение домашнего задания

Cоздаем БД hw2 и подключаемся к ней:
```bash
postgres=# CREATE DATABASE hw2;
CREATE DATABASE
postgres=#
postgres=# \c hw2
You are now connected to database "hw2" as user "postgres".
hw2=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 hw2       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)

hw2=# SELECT current_database();
 current_database
------------------
 hw2
(1 row)

hw2=#
```

Заходим в другом окне терминала по ssh, запускаем psql и подключаемся к нашей БД hw2:
```bash
postgres@pg02:~$ psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c hw2
You are now connected to database "hw2" as user "postgres".
hw2=#
```

Выключаем автокоммит в обеих сессиях:
```bash
hw2=# \echo :AUTOCOMMIT
on
hw2=# \set AUTOCOMMIT OFF
hw2=# \echo :AUTOCOMMIT
OFF
hw2=#
```

Создаем в первой сессии новую таблицу и наполняем ее данными:
```bash
hw2=# CREATE TABLE persons
(
   id            serial,
   first_name    text,
   second_name   text
);

INSERT INTO persons (first_name, second_name)
     VALUES ('ivan', 'ivanov');

INSERT INTO persons (first_name, second_name)
     VALUES ('petr', 'petrov');

COMMIT;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
hw2=#
```

Смотрим уровень изоляции транзакции в обеих сессиях:
```bash
hw2=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

hw2=*#
```

В первой сессии добавляем новую запись:
```bash
hw2=*# INSERT INTO persons (first_name, second_name)
     VALUES ('sergey', 'sergeev');
INSERT 0 1
hw2=*#
```

Во второй сессии выполняем SELECT:
```bash
hw2=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

hw2=*#
```

<span style="color:red"> **Вопрос - видите ли вы новую запись и если да, то почему?**</span>

<span style="color:green"> **Ответ - нет.** </span>

Выполняем commit в первой сессии и снова SELECT во второй:
```bash
hw2=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

hw2=*#
```

<span style="color:red"> **Вопрос - видите ли вы новую запись и если да, то почему?**</span>

<span style="color:green"> **Ответ - видим, потому что в данном примере используется уровень изоляции Read Committed. Аномалия фантомного чтения (phantom read) возникает, когда одна транзакция два раза читает набор строк по одинаковому условию, а в промежутке между чтениями другая транзакция добавляет строки, удовлетворяющие этому условию, и фиксирует изменения. Тогда первая транзакция получит разные наборы строк. Фантомное чтение допускается стандартом на уровнях Read Uncommitted, Read Committed и Repeatable Read.**</span>

<span style="color:green"> **[Из документации](https://postgrespro.ru/docs/postgresql/14/transaction-iso#XACT-READ-COMMITTED): "Read Committed — уровень изоляции транзакции, выбираемый в PostgreSQL по умолчанию. В транзакции, работающей на этом уровне, запрос SELECT (без предложения FOR UPDATE/SHARE) видит только те данные, которые были зафиксированы до начала запроса; он никогда не увидит незафиксированных данных или изменений, внесённых в процессе выполнения запроса параллельными транзакциями. По сути запрос SELECT видит снимок базы данных в момент начала выполнения запроса. Однако SELECT видит результаты изменений, внесённых ранее в этой же транзакции, даже если они ещё не зафиксированы. Также заметьте, что два последовательных оператора SELECT могут видеть разные данные даже в рамках одной транзакции, если какие-то другие транзакции зафиксируют изменения после запуска первого SELECT, но до запуска второго."**</span>

Завершаем транзакцию во второй сессии, выставляем в обеих сессиях уровень изоляции repeatable read:
```bash
hw2=# set transaction isolation level repeatable read;
SET
hw2=*#
```

В первой сессии добавляем новую запись:
```bash
hw2=*# INSERT INTO persons (first_name, second_name)
     VALUES ('sveta', 'svetova');
INSERT 0 1
hw2=*#
```

Во второй сессии выполняем SELECT:
```bash
hw2=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

<span style="color:red"> **Вопрос - видите ли вы новую запись и если да, то почему?**</span>

<span style="color:green"> **Ответ - нет.** </span>

Завершаем первую транзакцию - commit:
```bash
hw2=*# commit;
COMMIT
hw2=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  5 | sveta      | svetova
(4 rows)

hw2=*#
```

Во второй сессии выполняем SELECT:
```bash
hw2=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

<span style="color:red"> **Вопрос - видите ли вы новую запись и если да, то почему?**</span>

<span style="color:green"> **Ответ - нет.** </span>

Завершаем вторую транзакцию - commit и снова делаем SELECT:
```bash
hw2=*# commit;
COMMIT
hw2=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  5 | sveta      | svetova
(4 rows)
```

<span style="color:red"> **Вопрос: видите ли вы новую запись и если да, то почему?**</span>

<span style="color:green"> **Ответ - видим после завершения второй транзакции, потому что в данном примере используется уровень изоляции Repeatable Read.**</span>

<span style="color:green"> **[Из документации](https://postgrespro.ru/docs/postgresql/14/transaction-iso#XACT-REPEATABLE-READ): "В режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции, но не видны незафиксированные данные и изменения, произведённые другими транзакциями в процессе выполнения данной транзакции."**</span>