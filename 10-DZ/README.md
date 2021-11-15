# Репликация

>Цель:
>реализовать свой миникластер на 3 ВМ.

>На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. На 2 ВМ   >создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 3 ВМ использовать   
>как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). Небольшое описание, того, что получилось.  

Создаем 3 ВМ RHEL1 RHEL2 RHEL3, запускаем, подключаемся
```console
 gcloud beta compute ssh --zone "us-central1-a" "rhel1"  --project "clever-muse-328410"
 gcloud beta compute ssh --zone "us-central1-a" "rhel2"  --project "clever-muse-328410"
 gcloud beta compute ssh --zone "us-central1-a" "rhel3"  --project "clever-muse-328410"
```
Обновляем ОС и перегружаем, на трех машинах
```console
yum update -y
reboot
```
Добавляем репозиторий, на трех машинах
```console
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Устанавилваем PostgreSQL 14 инициализируем и запускаем сервер БД, на трех машинах
```console
yum install -y postgresql14-server
/usr/pgsql-14/bin/initdb --locale en_US.UTF-8
systemctl enable postgresql-14
systemctl start postgresql-14  
```
Меняем на первых двух машинах уровень делализации WAL файлов на более высокий, перегружаем сервис БД, проверяем
```console
postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM

systemctl restart postgresql-14

postgres=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```
Создаем на первых двух машинах таблицы test и test2
```console
postgres=# create table test (id int, name text);
CREATE TABLE
postgres=# create table test2 (id int, name text);
CREATE TABLE
```
Создаем публикацию таблицы test на первой машине.
```console
postgres=# create publication test_pub for table test;
CREATE PUBLICATION
```
Создаем публикацию таблицы test2 на второй машине.
```console
postgres=# create publication test2_pub for table test2;
CREATE PUBLICATION
```
Необходимо разрешить сетевое подключение к БД между всеми машинами, добавляем строку в pg_hba.conf на всех ВМ
и добавляем строку с postgresql.conf, чтобы сервис слушал все интерфейсы
```console
echo "host    all             all             0.0.0.0/0               md5" >> /var/lib/pgsql/14/data/pg_hba.conf
echo listen_addresses = \'*\' >> /var/lib/pgsql/14/data/postgresql.conf
```
И применяем изменения, на всех ВМ
```console
systemctl restart postgresql-14
```
Чтобы была возможность сделать подключение к базе между всеми серверами, надо чтобы при таких настройках пользователь имел пароль.
Меняем его на всех машинах.
```console
postgres=# \password postgres
Enter new password:
Enter it again:
```
Проверяем
```console
-bash-4.2$ psql -h 10.128.0.7
Password for user postgres:
psql (14.1)
Type "help" for help.

postgres=#
```
Создаем подписку на первой ВМ ко второй на таблицу test2
```console
CREATE SUBSCRIPTION test2_sub 
CONNECTION 'host=10.128.0.7 port=5432 user=postgres password=1 dbname=postgres' 
PUBLICATION test2_pub WITH (copy_data = true);
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```
Создаем подписку на второй ВМ к первой на таблицу test
```console
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=10.128.0.6 port=5432 user=postgres password=1 dbname=postgres' 
PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
Просматриваем подписки на первой
postgres=# SELECT * FROM pg_stat_subscription \gx
```console
-[ RECORD 1 ]---------+------------------------------
subid                 | 16396
subname               | test2_sub
pid                   | 1662
relid                 |
received_lsn          | 0/1774A18
last_msg_send_time    | 2021-11-15 07:11:22.907109+00
last_msg_receipt_time | 2021-11-15 07:11:22.907202+00
latest_end_lsn        | 0/1774A18
latest_end_time       | 2021-11-15 07:11:22.907109+00
```
И на второй
```console
postgres=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16400
subname               | test_sub
pid                   | 1784
relid                 |
received_lsn          | 0/1772378
last_msg_send_time    | 2021-11-15 07:12:22.809504+00
last_msg_receipt_time | 2021-11-15 07:12:22.810557+00
latest_end_lsn        | 0/1772378
latest_end_time       | 2021-11-15 07:12:22.809504+00
```
Создаем аналогичные таблицы и две подписки на третьей ВМ.
```console
postgres=# create table test (id int, name text);
CREATE TABLE
postgres=# create table test2 (id int, name text);
CREATE TABLE
postgres=# CREATE SUBSCRIPTION test_sub3
CONNECTION 'host=10.128.0.6 port=5432 user=postgres password=1 dbname=postgres'
PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub3" on publisher
CREATE SUBSCRIPTION
postgres=# CREATE SUBSCRIPTION test2_sub3
CONNECTION 'host=10.128.0.7 port=5432 user=postgres password=1 dbname=postgres'
PUBLICATION test2_pub WITH (copy_data = true);
NOTICE:  created replication slot "test2_sub3" on publisher
CREATE SUBSCRIPTION
```
Проверяем
```console
postgres=# \dRs
             List of subscriptions
    Name    |  Owner   | Enabled | Publication
------------+----------+---------+-------------
 test2_sub3 | postgres | t       | {test2_pub}
 test_sub3  | postgres | t       | {test_pub}
(2 rows)

postgres=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16396
subname               | test_sub3
pid                   | 1556
relid                 |
received_lsn          | 0/17723E8
last_msg_send_time    | 2021-11-15 07:22:22.858358+00
last_msg_receipt_time | 2021-11-15 07:22:22.859345+00
latest_end_lsn        | 0/17723E8
latest_end_time       | 2021-11-15 07:22:22.858358+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16398
subname               | test2_sub3
pid                   | 1559
relid                 |
received_lsn          | 0/1774BA8
last_msg_send_time    | 2021-11-15 07:22:18.225785+00
last_msg_receipt_time | 2021-11-15 07:22:18.22662+00
latest_end_lsn        | 0/1774BA8
latest_end_time       | 2021-11-15 07:22:18.225785+00
```
Проверим работоспособность этой схемы логической репликации.
Добавляем строки в таблицы test на первой ВМ и в test2 на второй
```console
postgres=# insert into  test values (1,'test1'), (2,'test2');
INSERT 0 2

postgres=# insert into  test2 values (3,'test3'), (4,'test4');
INSERT 0 2
```
Просматриваем содержимое таблиц на всех ВМ.
```console
postgres=# select * from test;
 id | name
----+-------
  1 | test1
  2 | test2
(2 rows)

postgres=# select * from test2;
 id | name
----+-------
  3 | test3
  4 | test4
(2 rows)
```
Логическая репликация работает. Таким образом на первой машине изменения вносятся в таблицу test, на второй в таблицу test2.
Все таблицы на всех трех машинах доступны для чтения и данные задублированы 3 раза.
>реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.  

Создаем 4 ВМ RHEL4, запускаем, подключаемся
```console
gcloud beta compute ssh --zone "us-central1-a" "rhel4"  --project "clever-muse-328410"
```
Обновляем ОС и перегружаем
```console
yum update -y
reboot
```
Добавляем репозиторий
```console
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Устанавилваем PostgreSQL 14
```console
yum install -y postgresql14-server
```
Для того чтобы с третьей ВМ можно было реплицировать данные, надо добавить плавило в pg_hba.conf
```console
echo "host    replication             all             0.0.0.0/0               md5" >> /var/lib/pgsql/14/data/pg_hba.conf
```
И перечитать конфигурацию.
```console
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```
Делаем копию базы данных, используя pg_basebackup
```console
-bash-4.2$ pg_basebackup -p 5432 -h 10.128.0.8 -U postgres -W -v -P -R -D /var/lib/pgsql/14/data/
Password:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_1879"
27059/27059 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/2000138
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```
Запускаем сервис БД на четвертой машине
```console
systemctl start postgresql-14
```
Посмотрим процессы, видно что есть recovering и прием от мастера walreceiver streaming
```console
[root@rhel4 ~]# ps ax | grep post
 1202 ?        Ss     0:00 /usr/libexec/postfix/master -w
 1555 ?        Ss     0:00 /usr/pgsql-14/bin/postmaster -D /var/lib/pgsql/14/data/
 1557 ?        Ss     0:00 postgres: logger
 1558 ?        Ss     0:00 postgres: startup recovering 000000010000000000000003
 1559 ?        Ss     0:00 postgres: checkpointer
 1560 ?        Ss     0:00 postgres: background writer
 1561 ?        Ss     0:00 postgres: stats collector
 1562 ?        Ss     0:00 postgres: walreceiver streaming 0/3000060
 1564 pts/0    S+     0:00 grep --color=auto post
```
Убедимся что БД действительно в состоянии восстановления.
```console
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)
```
А на мастере мы также видим что идет асинхронная репликация и запущен процесс walsender.
```console
postgres=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 1931
usesysid         | 10
usename          | postgres
application_name | walreceiver
client_addr      | 10.128.0.9
client_hostname  |
client_port      | 49074
backend_start    | 2021-11-15 08:16:37.198527+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000148
write_lsn        | 0/3000148
flush_lsn        | 0/3000148
replay_lsn       | 0/3000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-15 08:24:34.673328+00

bash-4.2$ ps ax | grep send
 1931 ?        Ss     0:00 postgres: walsender postgres 10.128.0.9(49074) streaming 0/3000148
```
Проверим работоспособность физической репликации. Добавим по записи на первой ВМ в таблицу test, на второй в test2.
```console
postgres=# insert into  test values (5,'test5');
INSERT 0 1
postgres=# insert into  test2 values (6,'test6');
INSERT 0 1
```
Проверяем на четверной машине и видим что репликация успешно работает
```console
postgres=# select * from test;
 id | name
----+-------
  1 | test1
  2 | test2
  5 | test5
(3 rows)

postgres=# select * from test2;
 id | name
----+-------
  3 | test3
  4 | test4
  6 | test6
(3 rows)
```
Проверим что в в БД на четвертой машине нельзя изменять данные она запущена в режиме Read Only
```console
postgres=# create table test4(id int, name text);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

