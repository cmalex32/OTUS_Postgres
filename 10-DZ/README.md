# Репликация

>Цель:
>реализовать свой миникластер на 3 ВМ.

>На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. На 2 ВМ   >создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 3 ВМ использовать   >как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). Небольшое описание, того, что получилось.  

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

>реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.  
