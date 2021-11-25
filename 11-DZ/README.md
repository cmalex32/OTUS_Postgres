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
Copying gs://my_alex_bucket/hacker_news.full/full_000000000000.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000003.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000002.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000006.csv...
==> NOTE: You are downloading one or more large file(s), which would
run significantly faster if you enabled sliced object downloads. This
feature is enabled by default but requires that compiled crcmod be
installed (see "gsutil help crcmod").

Copying gs://my_alex_bucket/hacker_news.full/full_000000000001.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000005.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000009.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000008.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000007.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000010.csv...38
Copying gs://my_alex_bucket/hacker_news.full/full_000000000012.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000011.csv...29
Copying gs://my_alex_bucket/hacker_news.full/full_000000000016.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000018.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000017.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000019.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000013.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000014.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000015.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000020.csv...17
Copying gs://my_alex_bucket/hacker_news.full/full_000000000021.csv...03
Copying gs://my_alex_bucket/hacker_news.full/full_000000000022.csv...59
Copying gs://my_alex_bucket/hacker_news.full/full_000000000023.csv...35
Copying gs://my_alex_bucket/hacker_news.full/full_000000000024.csv...35
Copying gs://my_alex_bucket/hacker_news.full/full_000000000025.csv...37
Copying gs://my_alex_bucket/hacker_news.full/full_000000000026.csv...36
Copying gs://my_alex_bucket/hacker_news.full/full_000000000027.csv...39
Copying gs://my_alex_bucket/hacker_news.full/full_000000000028.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000029.csv...38
Copying gs://my_alex_bucket/hacker_news.full/full_000000000030.csv...26
Copying gs://my_alex_bucket/hacker_news.full/full_000000000031.csv...23
Copying gs://my_alex_bucket/hacker_news.full/full_000000000032.csv...19
Copying gs://my_alex_bucket/hacker_news.full/full_000000000033.csv...54
Copying gs://my_alex_bucket/hacker_news.full/full_000000000034.csv...54
Copying gs://my_alex_bucket/hacker_news.full/full_000000000035.csv...54
Copying gs://my_alex_bucket/hacker_news.full/full_000000000036.csv...55
Copying gs://my_alex_bucket/hacker_news.full/full_000000000037.csv...55
Copying gs://my_alex_bucket/hacker_news.full/full_000000000038.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000039.csv...55
Copying gs://my_alex_bucket/hacker_news.full/full_000000000040.csv...04
Copying gs://my_alex_bucket/hacker_news.full/full_000000000042.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000041.csv...
Copying gs://my_alex_bucket/hacker_news.full/full_000000000043.csv...07
Copying gs://my_alex_bucket/hacker_news.full/full_000000000044.csv...07
Copying gs://my_alex_bucket/hacker_news.full/full_000000000045.csv...07
Copying gs://my_alex_bucket/hacker_news.full/full_000000000046.csv...06
[root@instance-2 ~]#  GiB/ 11.6 GiB]  99% Done  25.4 MiB/s ETA 00:00:00

postgres=# create database hacker_news
       
psql "dbname=hacker_news user=postgres" -c "\\COPY full FROM PROGRAM 'cat full_000000000000.csv' CSV HEADER"   
```console
create table full (       
title	TEXT,
url	TEXT,
text	TEXT,
dead	BOOLEAN,
by	TEXT,
score	BIGINT,
time	BIGINT,
timestamp	TIMESTAMP,
type	TEXT,
id	BIGINT,
parent	BIGINT,
descendants	BIGINT,
ranking	BIGINT,
deleted	BOOLEAN
);      
```       
