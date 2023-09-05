# Механизм блокировок

## Создаем ВМ pg01:
```bash
yc compute instance create --name pg01 --hostname pg01 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2204-lts  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 --zone ru-central1-a  --metadata-from-file ssh-keys=C:\Users\fvg00\.ssh\fvg00.pub --memory 4G
```
Подключаемся:
```bash
ssh ubuntu@51.250.10.2
```

## Вывод в журнал информации о блокировках:
<span style="color:red"> **Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.**</span>

Указанный пример рассматривался на лекции, пробуем воспроизвести:

[log_lock_waits (boolean)](https://postgrespro.ru/docs/postgresql/12/runtime-config-logging#GUC-LOG-LOCK-WAITS) - определяет, нужно ли фиксировать в журнале события, когда сеанс ожидает получения блокировки дольше, чем указано в [deadlock_timeout](https://postgrespro.ru/docs/postgresql/12/runtime-config-locks#GUC-DEADLOCK-TIMEOUT). Это позволяет выяснить, не связана ли низкая производительность с ожиданием блокировок. По умолчанию отключено. Только суперпользователи могут изменить этот параметр.
```bash
Настраиваем параметры:
postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout = '200ms';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

Воспроизведем блокировку на известном примере со счетами:
postgres=# CREATE TABLE accounts(acc_no integer PRIMARY KEY, amount numeric);
CREATE TABLE
postgres=# INSERT INTO accounts VALUES (1,1000.00),(2,2000.00),(3,3000.00);
INSERT 0 3
postgres=# select * from accounts;
 acc_no | amount
--------+---------
      1 | 1000.00
      2 | 2000.00
      3 | 3000.00
(3 rows)

Начнем транзакцию в первой сессии:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = 500.00 WHERE acc_no = 1;
UPDATE 1
postgres=*#

Начнем транзакцию во второй сессии и ожидаемо получаем, что UPDATE завис на блокировке:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = 200.00 WHERE acc_no = 1;

В первой сессии завершим транзакцию.
postgres=*# commit;
COMMIT
postgres=#

После этого у нас выполняется UPDATE во второй сессии:
UPDATE 1
postgres=*# commit;
COMMIT
postgres=#

Смотрим журнал событий БД, в котором мы получили ожидаемые сведения о блокировке более 200 миллисекунд и ее завершении:
postgres$ tail -f /var/log/postgresql/postgresql-15-main.log
2023-09-04 19:12:46.200 UTC [30429] postgres@postgres LOG:  process 30429 still waiting for ShareLock on transaction 1574926 after 200.130 ms
2023-09-04 19:12:46.200 UTC [30429] postgres@postgres DETAIL:  Process holding the lock: 30306. Wait queue: 30429.
2023-09-04 19:12:46.200 UTC [30429] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "accounts"
2023-09-04 19:12:46.200 UTC [30429] postgres@postgres STATEMENT:  UPDATE accounts SET amount = 200.00 WHERE acc_no = 1;
2023-09-04 19:13:49.097 UTC [30429] postgres@postgres LOG:  process 30429 acquired ShareLock on transaction 1574926 after 63097.059 ms
2023-09-04 19:13:49.097 UTC [30429] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "accounts"
2023-09-04 19:13:49.097 UTC [30429] postgres@postgres STATEMENT:  UPDATE accounts SET amount = 200.00 WHERE acc_no = 1;
```

## Создаем ситуации с блокировками:
<span style="color:red"> **Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.**</span>

Указанный пример рассматривался на лекции, пробуем воспроизвести:
```bash
Создадим расширение pageinspect:
CREATE EXTENSION pageinspect;

построим представление над pg_locks:
postgres=# CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'accounts'::regclass);
CREATE VIEW
postgres=#

Первая сессия/транзакция обновляет и блокирует строку: 
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      1574929 |          30306
(1 row)

postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1
postgres=*#

Во второй сессии делаем то же самое:
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      1574930 |          30429
(1 row)

postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

Третья сессия:
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      1574931 |          30642
(1 row)

postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

Блокировки для первой транзакции:
postgres=*# SELECT * FROM locks_v WHERE pid = 30306;
  pid  |   locktype    |  lockid  |       mode       | granted
-------+---------------+----------+------------------+---------
 30306 | relation      | accounts | RowExclusiveLock | t
 30306 | transactionid | 1574929  | ExclusiveLock    | t
(2 rows)

Тип relation для accounts в режиме RowExclusiveLock — устанавливается на изменяемое отношение. 
Тип transactionid в режиме ExclusiveLock — удерживается каждой транзакцией для самой себя. 

Блокировки для второй транзакции:
postgres=*# SELECT * FROM locks_v WHERE pid = 30429;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 30429 | relation      | accounts   | RowExclusiveLock | t
 30429 | tuple         | accounts:6 | ExclusiveLock    | t
 30429 | transactionid | 1574930    | ExclusiveLock    | t
 30429 | transactionid | 1574929    | ShareLock        | f
(4 rows)

Транзакция ожидает получение блокировки типа transactionid в режиме ShareLock для первой транзакции. 
Удерживается блокировка типа tuple для обновляемой строки. 


Блокировки для третьей транзакции:
postgres=*# SELECT * FROM locks_v WHERE pid = 30642;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 30642 | relation      | accounts   | RowExclusiveLock | t
 30642 | tuple         | accounts:6 | ExclusiveLock    | f
 30642 | transactionid | 1574931    | ExclusiveLock    | t
(3 rows)

Транзакция ожидает получение блокировки типа tuple для обновляемой строки. 

Общую картину текущих ожиданий можно увидеть в представлении pg_stat_activity. Для удобства можно добавить и информацию о блокирующих процессах: 
postgres=*# SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE backend_type = 'client backend';
  pid  | wait_event_type |  wait_event   | pg_blocking_pids
-------+-----------------+---------------+------------------
 30306 |                 |               | {}
 30429 | Lock            | transactionid | {30306}
 30642 | Lock            | tuple         | {30429}
(3 rows)

Первая сессия pid=30306 удерживает блокировку версии строки, сессия pid=30429 ждет эту блокировку, в свою очередь третья сессия pid=30642 ждет блокировку от второй pid=30429.

Для завершения примера выполним откат ROLLBACK; во всех сессиях.
```

<span style="color:red"> **Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?**</span>

Указанный пример рассматривался на лекции, пробуем воспроизвести:
```bash
Первая сессия: 
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
UPDATE 1
postgres=*#

Вторая сессия:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 2;
UPDATE 1
postgres=*#

Третья сессия:
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 3;
UPDATE 1

Теперь в каждой сессии пытаемся выполнить UPDATE другой записи:
Первая сессия: 
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

Вторая сессия:
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;

Первая и вторая сесии зависли на ожидании блокировки.

Третья сессия:
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
ERROR:  deadlock detected
DETAIL:  Process 30429 waits for ShareLock on transaction 1574932; blocked by process 30306.
Process 30306 waits for ShareLock on transaction 1574933; blocked by process 30642.
Process 30642 waits for ShareLock on transaction 1574934; blocked by process 30429.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,6) in relation "accounts"

При попытке выполнения UPDATE в третьей сессии получаем ошибку "deadlock detected" и ROLLBACK:
postgres=!#postgres=!# commit;
ROLLBACK
postgres=#

Смотрим журнал событий БД, в котором появилось описание возникшей ситуации. Видим, что в ситуации можно разобраться постфактум, изучая журнал сообщений.

2023-09-04 19:33:23.596 UTC [30429] postgres@postgres LOG:  process 30429 detected deadlock while waiting for ShareLock on transaction 1574932 after 200.120 ms
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres DETAIL:  Process holding the lock: 30306. Wait queue: .
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres CONTEXT:  while updating tuple (0,6) in relation "accounts"
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres ERROR:  deadlock detected
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres DETAIL:  Process 30429 waits for ShareLock on transaction 1574932; blocked by process 30306.
        Process 30306 waits for ShareLock on transaction 1574933; blocked by process 30642.
        Process 30642 waits for ShareLock on transaction 1574934; blocked by process 30429.
        Process 30429: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
        Process 30306: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
        Process 30642: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres HINT:  See server log for query details.
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres CONTEXT:  while updating tuple (0,6) in relation "accounts"
2023-09-04 19:33:23.596 UTC [30429] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2023-09-04 19:33:23.596 UTC [30642] postgres@postgres LOG:  process 30642 acquired ShareLock on transaction 1574934 after 6273.514 ms
2023-09-04 19:33:23.596 UTC [30642] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "accounts"
2023-09-04 19:33:23.596 UTC [30642] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
```


<span style="color:red"> **Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?**</span>

Для моделирования UPDATE одной и той же таблицы без where попробуем использовать курсор с прямой сортировкой по acc_no для первой сессии 
и курсор с обратной сортировкой по acc_no для второй сессии:
```bash
postgres=# select * from accounts order by acc_no;
 acc_no | amount
--------+---------
      1 |  900.00
      2 | 2000.00
      3 | 3100.00
(3 rows)

Первая сессия:
postgres=# BEGIN;
DECLARE cur1 CURSOR FOR SELECT * FROM accounts ORDER BY acc_no FOR UPDATE;
BEGIN
DECLARE CURSOR

Вторая сессия:
postgres=# BEGIN;
DECLARE cur2 CURSOR FOR SELECT * FROM accounts ORDER BY acc_no DESC FOR UPDATE;
BEGIN
DECLARE CURSOR

Первая сессия:
postgres=*# FETCH cur1;
 acc_no | amount
--------+--------
      1 | 900.00
(1 row)

Вторая сессия:
postgres=*# FETCH cur2;
 acc_no | amount
--------+---------
      3 | 3100.00
(1 row)

Первая сессия:
postgres=*# FETCH cur1;
 acc_no | amount
--------+---------
      2 | 2000.00
(1 row)

Вторая сессия:
postgres=*# FETCH cur2;

Вторая сессия ожидает блокировку, информация в журнале:
2023-09-04 20:12:36.521 UTC [30429] postgres@postgres LOG:  process 30429 still waiting for ShareLock on transaction 1574935 after 200.248 ms
2023-09-04 20:12:36.521 UTC [30429] postgres@postgres DETAIL:  Process holding the lock: 30306. Wait queue: 30429.
2023-09-04 20:12:36.521 UTC [30429] postgres@postgres CONTEXT:  while locking tuple (0,14) in relation "accounts"
2023-09-04 20:12:36.521 UTC [30429] postgres@postgres STATEMENT:  FETCH cur2;

Первая сессия:
postgres=*# FETCH cur1;
ERROR:  deadlock detected
DETAIL:  Process 30306 waits for ShareLock on transaction 1574936; blocked by process 30429.
Process 30429 waits for ShareLock on transaction 1574935; blocked by process 30306.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,13) in relation "accounts"
postgres=!#

Произошла взаимоблокировка, информация в журнале:
2023-09-04 20:14:01.345 UTC [30306] postgres@postgres LOG:  process 30306 detected deadlock while waiting for ShareLock on transaction 1574936 after 200.139 ms
2023-09-04 20:14:01.345 UTC [30306] postgres@postgres DETAIL:  Process holding the lock: 30429. Wait queue: .
2023-09-04 20:14:01.345 UTC [30306] postgres@postgres CONTEXT:  while locking tuple (0,13) in relation "accounts"
2023-09-04 20:14:01.345 UTC [30306] postgres@postgres STATEMENT:  FETCH cur1;
2023-09-04 20:14:01.346 UTC [30306] postgres@postgres ERROR:  deadlock detected
2023-09-04 20:14:01.346 UTC [30306] postgres@postgres DETAIL:  Process 30306 waits for ShareLock on transaction 1574936; blocked by process 30429.
        Process 30429 waits for ShareLock on transaction 1574935; blocked by process 30306.
        Process 30306: FETCH cur1;
        Process 30429: FETCH cur2;
2023-09-04 20:14:01.346 UTC [30306] postgres@postgres HINT:  See server log for query details.
2023-09-04 20:14:01.346 UTC [30306] postgres@postgres CONTEXT:  while locking tuple (0,13) in relation "accounts"
2023-09-04 20:14:01.346 UTC [30306] postgres@postgres STATEMENT:  FETCH cur1;
2023-09-04 20:14:01.346 UTC [30429] postgres@postgres LOG:  process 30429 acquired ShareLock on transaction 1574935 after 85025.028 ms
2023-09-04 20:14:01.346 UTC [30429] postgres@postgres CONTEXT:  while locking tuple (0,14) in relation "accounts"
2023-09-04 20:14:01.346 UTC [30429] postgres@postgres STATEMENT:  FETCH cur2;


После этого во второй сессии отрабатывает fetch:
postgres=*# FETCH cur2;
 acc_no | amount
--------+---------
      2 | 2000.00
(1 row)

postgres=*# commit;
COMMIT
postgres=#

Пробуем сделать commit в первой сессии и получаем откат:
postgres=!# commit;
ROLLBACK
postgres=#
```
<span style="color:red"> **В рассматриваемом примере команда UPDATE блокирует строки постепенно, по мере их обновления. Желаемая ситуация взаимоблокировки была получена путем обновления
строк в таблице в  разном порядке с помощью курсоров.**</span>