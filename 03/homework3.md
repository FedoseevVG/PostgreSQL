# Установка и настройка PostgteSQL в контейнере Docker

## Создаем ВМ pg01
```bash
yc compute instance create --name pg01 --hostname pg01 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub
```
Подключаемся:
```bash
ssh ubuntu@158.160.98.136
```

## Устанавливаем PostgreSQL 15
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
Удаляем предустановленный кластер:
```bash
ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
ubuntu@pg01:~$ sudo pg_ctlcluster 15 main stop
ubuntu@pg01:~$ sudo pg_dropcluster 15 main
ubuntu@pg01:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file
ubuntu@pg01:~$
```

## Ставим [docker](https://docs.docker.com/engine/install/ubuntu/)
```bash
ubuntu@pg01:~$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
# Executing docker install script, commit: c2de0811708b6d9015ed1a2c80f02c9b70c8ce7b
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null
+ sh -c install -m 0755 -d /etc/apt/keyrings
+ sh -c curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
+ sh -c chmod a+r /etc/apt/keyrings/docker.gpg
+ sh -c echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-ce-rootless-extras docker-buildx-plugin >/dev/null
+ sh -c docker version
Client: Docker Engine - Community
 Version:           24.0.5
 API version:       1.43
 Go version:        go1.20.6
 Git commit:        ced0996
 Built:             Fri Jul 21 20:35:18 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.5
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.6
  Git commit:       a61e2b4
  Built:            Fri Jul 21 20:35:18 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.22
  GitCommit:        8165feabfdfe38c65b599c4993d227328c231fca
 runc:
  Version:          1.1.8
  GitCommit:        v1.1.8-0-g82f18fe
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================

ubuntu@pg01:~$
```
Создаем docker-сеть: 
```bash
ubuntu@pg01:~$ sudo docker network create pg-net
b88ebe24ff89e0ada55f3c19a3828664405b99e9e24b9be570f36478f2c27832
ubuntu@pg01:~$
```
Подключаем созданную сеть к контейнеру сервера PostgreSQL (разворачиваем контейнер, монтируя в него каталог /var/lib/postgres по пути /var/lib/postgresql):
```bash
ubuntu@pg01:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
648e0aadf75a: Pull complete
f715c8c55756: Pull complete
b11a1dc32c8c: Pull complete
f29e8ba9d17c: Pull complete
78af88a8afb0: Pull complete
b74279c188d9: Pull complete
6e3e5bf64fd2: Pull complete
b62a2c2d2ce5: Pull complete
eba91ca3c7a3: Pull complete
d4a24cdf2433: Pull complete
b20f8a8dfd5c: Pull complete
e0731dd084c3: Pull complete
0361da6a228e: Pull complete
Digest: sha256:8775adb39f0db45cf4cdb3601380312ee5e9c4f53af0f89b7dc5cd4c9a78e4e8
Status: Downloaded newer image for postgres:15
9587d6f310bdab60bf474d5d31bf4c3a17a8ae3b5b36106b43ee1506102da16b
ubuntu@pg01:~$
```

Проверяем:
```bash
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS              PORTS                                       NAMES
9587d6f310bd   postgres:15   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
ubuntu@pg01:~$
```
Подключаемся локально и создаем БД hw3
```bash
ubuntu@pg01:~$ sudo -u postgres psql -p 5432 -h localhost -W
could not change directory to "/home/ubuntu": Permission denied
Password:
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1), server 15.3 (Debian 15.3-1.pgdg120+1))
Type "help" for help.

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(3 rows)

postgres=# create database hw3;
CREATE DATABASE
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 hw3       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=# \q
ubuntu@pg01:~$
```

Запускаем отдельный контейнер с клиентом в общей сети с БД: 
```bash
ubuntu@pg01:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.3 (Debian 15.3-1.pgdg120+1))
Type "help" for help.

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 hw3       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=# \q
ubuntu@pg01:~$
```

Пробуем подключиться снаружи с ноутбука:
```bash
postgres@NBH:/mnt/c/Users/fvg00$ psql -p 5432 -U postgres -h 158.160.98.136 -d postgres -W
Password:
psql (14.6 (Ubuntu 14.6-1.pgdg22.04+1), server 15.3 (Debian 15.3-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 hw3       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
```

Подключаемся к базе hw3 и создаем таблицу:
```bash
postgres=# \c hw3
Password:
psql (14.6 (Ubuntu 14.6-1.pgdg22.04+1), server 15.3 (Debian 15.3-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
You are now connected to database "hw3" as user "postgres".
hw3=#

hw3=# CREATE TABLE persons
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
hw3=#
hw3=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

hw3=#
hw3=# \q
postgres@NBH:/mnt/c/Users/fvg00$
```

Далее остановим контейнер, удалим контейнер и создадим его заново:
```bash
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
cb22698aaf28   postgres:15   "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
cb22698aaf28   postgres:15   "docker-entrypoint.s…"   20 minutes ago   Up 20 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
ubuntu@pg01:~$ sudo docker stop cb22698aaf28
cb22698aaf28
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                     PORTS     NAMES
cb22698aaf28   postgres:15   "docker-entrypoint.s…"   20 minutes ago   Exited (0) 5 seconds ago             pg-server
ubuntu@pg01:~$ sudo docker rm cb22698aaf28
cb22698aaf28
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
ubuntu@pg01:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
6871398e8a86b168ed61333c67bb7c73d67329f28c3b8c87c8f76be575a03025
ubuntu@pg01:~$
```

Подключаемся к БД и проверяем, что база hw3 и таблица на месте:
```bash
postgres@NBH:/mnt/c/Users/fvg00$ psql -p 5432 -U postgres -h 158.160.98.136 -d postgres -W
Password:
psql (14.6 (Ubuntu 14.6-1.pgdg22.04+1), server 15.3 (Debian 15.3-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 hw3       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=# \c hw3
Password:
psql (14.6 (Ubuntu 14.6-1.pgdg22.04+1), server 15.3 (Debian 15.3-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
You are now connected to database "hw3" as user "postgres".
hw3=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

hw3=# \q
postgres@NBH:/mnt/c/Users/fvg00$
```

Заходим внутрь контейнера, пробуем поставить vim и atop, затем выходим:
```bash
ubuntu@pg01:~$ sudo docker exec -it pg-server bash
root@6871398e8a86:/#
apt-get update
apt-get install atop -y
apt-get install vim -y
root@6871398e8a86:/# exit
exit
ubuntu@pg01:~$
```

Останавливаем контейнер:
```bash
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
6871398e8a86   postgres:15   "docker-entrypoint.s…"   30 minutes ago   Up 30 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
ubuntu@pg01:~$ sudo docker stop 6871398e8a86
6871398e8a86
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                      PORTS     NAMES
6871398e8a86   postgres:15   "docker-entrypoint.s…"   30 minutes ago   Exited (0) 26 seconds ago             pg-server
ubuntu@pg01:~$
```


## Ставим docker compose
```bash
sudo apt install docker-compose -y
```
Копируем файл параметров docker-compose.yml на ВМ:
```bash
PS D:\Otus\PostgreSQL\03> scp docker-compose.yml ubuntu@158.160.98.136:/home/ubuntu
Enter passphrase for key 'C:\Users\fvg00/.ssh/id_ed25519':
docker_compose.yml                                                                    100%  313    42.3KB/s   00:00
PS D:\Otus\PostgreSQL\03>


ubuntu@pg01:~$ cat docker-compose.yml
version: '3.1'

volumes:
  pg_project:

services:
  pg_db:
    image: postgres:14
    restart: always
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_USER=postgres
      - POSTGRES_DB=stage
    volumes:
      - pg_project:/var/lib/postgresql/data
    ports:
      - ${POSTGRES_PORT:-5432}:5432
ubuntu@pg01:~$


ubuntu@pg01:~$ sudo docker-compose up -d
Creating network "ubuntu_default" with the default driver
Creating volume "ubuntu_pg_project" with default driver
Pulling pg_db (postgres:14)...
14: Pulling from library/postgres
648e0aadf75a: Already exists
f715c8c55756: Already exists
b11a1dc32c8c: Already exists
f29e8ba9d17c: Already exists
78af88a8afb0: Already exists
b74279c188d9: Already exists
6e3e5bf64fd2: Already exists
b62a2c2d2ce5: Already exists
765629a1b92d: Pull complete
365d9a245882: Pull complete
aeb308034f5e: Pull complete
ddb205754449: Pull complete
d403994c1833: Pull complete
Digest: sha256:eecfb2ab484dadaae790866c9e5090a67deeee76417f072786569bea03f53e3f
Status: Downloaded newer image for postgres:14
Creating ubuntu_pg_db_1 ... done
ubuntu@pg01:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                         PORTS                                       NAMES
dbae226c6fb8   postgres:14   "docker-entrypoint.s…"   45 seconds ago   Up 35 seconds                  0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   ubuntu_pg_db_1
6871398e8a86   postgres:15   "docker-entrypoint.s…"   2 hours ago      Exited (0) About an hour ago                                               pg-server
ubuntu@pg01:~$
```

Подключаемся (password - secret):
```bash
ubuntu@pg01:~$ sudo -u postgres psql -h localhost
could not change directory to "/home/ubuntu": Permission denied
Password for user postgres:
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1), server 14.8 (Debian 14.8-1.pgdg120+1))
Type "help" for help.

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 stage     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=#\q
```

Смотрим куда смонтировалась БД:
```bash
sudo su
root@pg01:/home/ubuntu# ls -al /var/lib/docker/volumes/ubuntu_pg_project/_data/

```
Работа завершена.
Удаляем ВМ pg01.

