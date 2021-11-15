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
[root@instance-2 ~]# yum update -y
[root@instance-2 ~]# reboot
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
Создаем публикацию таблицы test2 на первой машине.
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

>реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.  
