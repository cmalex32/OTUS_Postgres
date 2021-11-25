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
Устанавилваем PostgreSQL 14 инициализируем и запускаем сервер БД  
```console
yum install -y postgresql14-server
/usr/pgsql-14/bin/initdb --locale en_US.UTF-8 -D /mnt/pgdata
/usr/pgsql-14/bin/pg_ctl -D /mnt/pgdata start
```
>проверьте что кластер запущен через sudo -u postgres pg_lsclusters  

```console
systemctl status postgresql-14
...
Oct 13 17:12:18 instance-2 systemd[1]: Starting PostgreSQL 14 database server...
...
```
Диск добавлен проверяем  
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
[root@instance-2 ~]# vgcreate datavg /dev/sdb
  Volume group "datavg" successfully created
[root@instance-2 ~]# lvcreate -n lv_data -l+100%FREE datavg
  Logical volume "lv_data" created.
 mkfs.ext4 /dev/mapper/datavg-lv_data
```
Создаем каталог для монтирования
```console
mkdir -p /mnt/data
```
Добавляем в /etc/fstab  
```console
/dev/mapper/datavg-lv_data /mnt              ext4    defaults        0 0
```
Монтируем  
```console
mount -av
```
```console
[root@instance-2 ~]# mkdir /mnt/pgdata
[root@instance-2 ~]# chown postgres:postgres /mnt/pgdata
```

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
yum install gcsfuse
                                                    
gsutil -m cp gs://my_alex_bucket/hacker_news.full/*  hacker_news.full
Copying gs://my_alex_bucket/hacker_news.full/full_000000000004.csv...
...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000046.csv...06
99% Done  25.4 MiB/s ETA 00:00:00

postgres=# create database hacker_news
       
psql "dbname=hacker_news user=postgres" -c "\\COPY full FROM PROGRAM 'cat full_000000000000.csv' CSV HEADER"   
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

  
yum install mariadb-server mariadb
vi /etc/my.cnf.d/server.cnf
[mysqld]
datadir = /mnt/mysqldata
                                                    
chown mysql:mysql /mnt/mysqldata/                                                    
systemctl start mariadb
                                                    

                                                    
  
