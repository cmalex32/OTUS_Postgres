# Разворачиваем и настраиваем БД с большими данными

>Цель:
>знать различные механизмы загрузки данных
>уметь пользоваться различными механизмами загрузки данных
>Необходимо провести сравнение скорости работы 
>запросов на различных СУБД

>Выбрать одну из СУБД
>Загрузить в неё данные (10 Гб)
>Сравнить скорость выполнения запросов на PosgreSQL и выбранной СУБД
>Описать что и как делали и с какими проблемами столкнулись

Подключаемся к ВМ  
```console
gcloud beta compute ssh --zone "us-central1-a" "instance-2"  --project "clever-muse-328410"
```
Добавляем репозиторий  
```console
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Устанавливаем PostgreSQL 14 
```console
yum install -y postgresql14-server
```
Проверяем что кластер запустился
```console
systemctl status postgresql-14
...
Oct 13 17:12:18 instance-2 systemd[1]: Starting PostgreSQL 14 database server...
...
```
Добавляем SSD диск для данных размером 30Гб.
Диск добавлен проверяем.  
```console
lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   30G  0 disk 
```
Используем для работы с диском LVM  
```console
vgcreate datavg /dev/sdb
  Volume group "datavg" successfully created
lvcreate -n lv_data -l+100%FREE datavg
  Logical volume "lv_data" created.
 mkfs.ext4 /dev/mapper/datavg-lv_data
```
Добавляем в /etc/fstab  
```console
/dev/mapper/datavg-lv_data /mnt              ext4    defaults        0 0
```
Монтируем  
```console
mount -av
```
Создаем каталог для данных, и меняем права на него
```console
mkdir /mnt/pgdata
chown postgres:postgres /mnt/pgdata
```
Инициализируем и запускаем сервер БД  
```console
/usr/pgsql-14/bin/initdb --locale en_US.UTF-8 -D /mnt/pgdata
/usr/pgsql-14/bin/pg_ctl -D /mnt/pgdata start
```
Для того чтобы вытянуть файлы из клоуда, нужна утилита gsutil
Устанавливаем репозиторий
```console
tee /etc/yum.repos.d/gcsfuse.repo > /dev/null <<EOF
[gcsfuse]
name=gcsfuse (packages.cloud.google.com)
baseurl=https://packages.cloud.google.com/yum/repos/gcsfuse-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
И устанавливаем эти утилиты
```console
yum install gcsfuse
```
Выбираем в клоуде, подходящую по размеру (10.96 GB) таблицу full из схемы hacker_news
Делаем ее экспорт в csv файлы и вытягиваем утилитой
```console
gsutil -m cp gs://my_alex_bucket/hacker_news.full/*  hacker_news.full
Copying gs://my_alex_bucket/hacker_news.full/full_000000000004.csv...
...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000046.csv...06
99% Done  25.4 MiB/s ETA 00:00:00
```
Создаем базу
```console
postgres=# create database hacker_news
```
Создаем таблицу
```console
create table cfull (       
title TEXT,
url TEXT,
text TEXT,
dead BOOLEAN,
by TEXT,
score BIGINT,
time BIGINT,
timestamp TIMESTAMP,
type TEXT,
id BIGINT,
parent BIGINT,
descendants BIGINT,
ranking BIGINT,
deleted BOOLEAN
);     
```       
И в цикле заливаем данные
```console
time for f in full*
do
  echo -e "Processing $f file..."
  psql "dbname=hacker_news user=postgres" -c "\\COPY cfull FROM PROGRAM 'cat $f' CSV HEADER"
done
Processing full_000000000000.csv file...
COPY 623070
...
Processing full_000000000046.csv file...
COPY 623594

real    24m36.834s
user    0m20.895s
sys     0m25.254s
```
Посмотрим размеры базы и таблицы
```console
postgres=# \l+
                                                                     List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   |  Size   | Tablespace | Description
-------------+----------+----------+-------------+-------------+-----------------------+---------+------------+--------------------------------------------
 hacker_news | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 12 GB   | pg_default |

hacker_news=# \d+
                                   List of relations
 Schema | Name  | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+-------+-------+----------+-------------+---------------+-------+-------------
 public | cfull | table | postgres | permanent   | heap          | 12 GB |
(1 row)
```  
Сравнивать будем с очень популярной СУБД MariaDB.  
Устанавливаем.
```console
yum install mariadb-server mariadb
```
Меняем расположение базы данных
```console
vi /etc/my.cnf.d/server.cnf
[mysqld]
datadir = /mnt/mysqldata
```
Создаем каталог и меняем владельца
```console
mkdir /mnt/mysqldata
chown mysql:mysql /mnt/mysqldata/                                                    
```
Запускаем сервис БД
```console
systemctl start mariadb
```
Создаем базу
```console
MariaDB [(none)]> create database hacker_news;
Query OK, 1 row affected (0.00 sec)
```
И таблицу
```console
MariaDB [(none)]> use hacker_news;
Database changed
MariaDB [hacker_news]>

create table cfull (       
title TEXT,
url TEXT,
text TEXT,
dead BOOLEAN,
byt TEXT,
score BIGINT,
time BIGINT,
timestamp TIMESTAMP,
type TEXT,
id BIGINT,
parent BIGINT,
descendants BIGINT,
ranking BIGINT,
deleted BOOLEAN
);      
```
Заливаем в цикле данные в таблицу. Заливка в MariaDB была медленней на целых 6 минут.
```console
time for f in full*
do
  echo -e "Processing $f file..."
  echo 'LOAD DATA LOCAL INFILE '\"/root/hacker_news.full/$f\"' INTO TABLE cfull FIELDS TERMINATED BY "," ENCLOSED BY "\"" LINES TERMINATED BY "\n" IGNORE 1 LINES;'
  mysql -D hacker_news  -e 'LOAD DATA LOCAL INFILE '\"/root/hacker_news.full/$f\"' INTO TABLE cfull FIELDS TERMINATED BY "," ENCLOSED BY "\"" LINES TERMINATED BY "\n" IGNORE 1 LINES;'
done
Processing full_000000000000.csv file...
LOAD DATA LOCAL INFILE "/root/hacker_news.full/full_000000000000.csv" INTO TABLE cfull FIELDS TERMINATED BY "," ENCLOSED BY "\"" LINES TERMINATED BY "\n" IGNORE 1 LINES;
...
Processing full_000000000046.csv file...
LOAD DATA LOCAL INFILE "/root/hacker_news.full/full_000000000046.csv" INTO TABLE cfull FIELDS TERMINATED BY "," ENCLOSED BY "\"" LINES TERMINATED BY "\n" IGNORE 1 LINES;

real    30m31.013s
user    0m2.646s
sys     0m13.220s
```
Смотрим размеры БД и таблицы. Размер также оказался на 2 Гб больше.
```console
MariaDB [hacker_news]> SELECT table_schema AS "Имя базы данных", ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS "Размер в Гб" FROM information_schema.TABLES GROUP BY table_schema;
+------------------------------+----------------------+
| Имя базы данных              | Размер в Гб          |
+------------------------------+----------------------+
| hacker_news                  |                14.74 |
| information_schema           |                 0.00 |
| mysql                        |                 0.00 |
| performance_schema           |                 0.00 |
+------------------------------+----------------------+
4 rows in set (0.04 sec)

MariaDB [hacker_news]> SELECT table_name AS `Table`, round(((data_length + index_length) / 1024 / 1024), 2) `Size in MB` FROM information_schema.TABLES WHERE table_schema = "hacker_news";
+-------+------------+
| Table | Size in MB |
+-------+------------+
| cfull |   15095.00 |
+-------+------------+
1 row in set (0.02 sec)
```
Останавливаем MariaDB.
```console
systemctl stop mariadb
```
Смотрим количество строк и время выполнения запроса в PostgreSQL
```console
 time psql "dbname=hacker_news user=postgres" -c "select count(*) from cfull;"
  count
----------
 29326780
(1 row)

real    7m35.223s
user    0m0.003s
sys     0m0.010s
```
Останавливаем сервис БД PostgreSQL запускаем MariaDB.

/usr/pgsql-14/bin/pg_ctl -D /mnt/pgdata stop
waiting for server to shut down.... done
```console
server stopped
systemctl start mariadb                                                  
```
Выполняем такой же запрос. Количество записей получилось чуть больше в PostgreSQL.
Время выполнения запроса в PostgreSQL меньше более чем на 2 минуты.
```console
time mysql -D hacker_news  -e "select count(*) from cfull;"
+----------+
| count(*) |
+----------+
| 29325740 |
+----------+

real    9m51.630s
user    0m0.004s
sys     0m0.007s
```
Выполняем запрос с сортировкой и группировкой, так как нет никаких индексов и происходит full scan, время примерно одинаковое.
```console
time mysql -D hacker_news  -e "SELECT byt, count(*) cnt  FROM cfull group by byt order by cnt;" > mariadb.out

real    9m51.558s
user    0m0.398s
sys     0m0.066s

cat mariadb.out
byt     cnt
blableblable    1
liberaliter     1
...
rbanffy 41466
dragonwriter    42107
pjmlp   42803
jacquesm        45728
dang    52109
tptacek 55114
        877032
```
Останавливаем сервис MariaDB и запускаем PostgreSQL.
```console
systemctl stop mariadb
/usr/pgsql-14/bin/pg_ctl -D /mnt/pgdata start
```
Выпоняем такой же запрос к БД PostgreSQL.
Результат ожидаем, примерно такое же время как и предыдущий, время быстрее запроса к MariaDB также примерно на 2 минуты
```console
time psql "dbname=hacker_news user=postgres" -c "SELECT by, count(*) cnt  FROM cfull group by by order by cnt;" > pgsql.out

real    7m38.440s
user    0m0.852s
sys     0m0.079s

cat pgsql.out
       by        |  cnt
-----------------+--------
 tauk            |      1
 Throwaway27b    |      1
 The-Amagi       |      1
...
dragonwriter    |  42107
 pjmlp           |  42816
 jacquesm        |  45729
 dang            |  52110
 tptacek         |  55114
                 | 876494
(764337 rows)

```

Обе СУБД при настройках по умолчанию показывают один и тот же порядок временных затрат на заливку и исполнению запросов.
Однако БД PostgreSQL показывает более лучшие показатели времени и более компактное хранение данных без индексов.

Проблемы возникшие при выполнении ДЗ.
При создании таблиц оказалось что для PostgreSQL зарезервированное слово full, а для MariaDB зарезервировано by, пришлось использовать другие.
При заливке данных закончилось место на диске, так как в БД MariaDB данные заняли больше места чем расчетное.
Чтобы не переделывать диск проще пододвинуть зарезервированное пространство на файловой системе и перезалить данные в БД MariaDB.
```console
tune2fs -m 1 /dev/mapper/datavg-lv_data
```
