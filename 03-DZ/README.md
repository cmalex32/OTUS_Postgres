# Установка и настройка PostgreSQL

>Цель:  
>создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему  
>переносить содержимое базы данных PostgreSQL на дополнительный диск  
>переносить содержимое БД PostgreSQL между виртуальными машинами  
>создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a  

Создаем ВМ (устанавливаем RHEL7)  
```console
gcloud compute instances create instance-3 --project=clever-muse-328410 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=816650361977-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-3,image=projects/rhel-cloud/global/images/rhel-7-v20210916,mode=rw,size=20,type=projects/clever-muse-328410/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

Подключаемся к ВМ  
`gcloud beta compute ssh --zone "us-central1-a" "instance-3"  --project "clever-muse-328410"`
Обновляем ОС и перегружаем  
`[root@instance-2 ~]# yum update -y`  
`[root@instance-2 ~]# reboot`  

>поставьте на нее PostgreSQL через sudo apt  

Добавляем репозиторий  
`yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm`  
Устанавилваем PostgreSQL 14 инициализируем и запускаем сервер БД  
`yum install -y postgresql14-server`  
`/usr/pgsql-14/bin/initdb --locale en_US.UTF-8 --data-checksums`  
`systemctl enable postgresql-14`  
`systemctl start postgresql-14`  
>проверьте что кластер запущен через sudo -u postgres pg_lsclusters  

`systemctl status postgresql-14`  
`...`  
`Oct 13 17:12:18 instance-2 systemd[1]: Starting PostgreSQL 14 database server...`  
`...`    

>зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым postgres=# create table test(c1 text); postgres=# insert into test values('1'); \q  

`[root@instance-2 bin]# su - postgres`  
`Last login: Wed Oct 13 17:12:36 UTC 2021 on pts/0`  
`-bash-4.2$ psql`  
`psql (14.0)`  
`Type "help" for help.`  
`postgres=# create table test(c1 text);`  
`CREATE TABLE`  
`postgres=# insert into test values('1'); \q`  
`INSERT 0 1`  
>остановите postgres например через sudo -u postgres pg_ctlcluster 13 main stop

`systemctl stop postgresql-14`    
>создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB

Создан
>добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk

Диск добавлен проверяем  
`lsblk`  
`NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT`  
`sda      8:0    0   20G  0 disk`  
`├─sda1   8:1    0  200M  0 part /boot/efi`  
`└─sda2   8:2    0 19.8G  0 part /`  
`sdb      8:16   0   10G  0 disk`  
>проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux

Используем для работы с диском LVM  
`yum install lvm2 -y`  
`vgcreate pgvg /dev/sdb`  
`lvcreate -n pg_data -l+100%FREE pgvg`  
`mkfs.ext4 /dev/mapper/pgvg-pg_data` 
Создаем каталог для монтирования
`mkdir -p /mnt/data`  
Добавляем в /etc/fstab  
`/dev/mapper/pgvg-pg_data /mnt/data              ext4    defaults        0 0`  
Монтируем  
`mount -av`  
>сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/

`chown postgres:postgres /mnt/data`  
>перенесите содержимое /var/lib/postgres/13 в /mnt/data - mv /var/lib/postgresql/13 /mnt/data

`mv /var/lib/pgsql/14/data /mnt/data` 
>попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start

`systemctl start postgresql-14`  
`Job for postgresql-14.service failed because the control process exited with error code. See "systemctl status postgresql-14.service" and "journalctl -xe" for details.`  
`systemctl status postgresql-14`  
`...`  
`Oct 13 17:48:20 instance-2 systemd[1]: Failed to start PostgreSQL 14 database server.`  
`...`  
>напишите получилось или нет и почему

Сервис не стартовал, так как в параметрах запуска указан каталог, который мы переместили  
>задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его
>напишите что и почему поменяли

меняем путь в конфигурации сервиса  
`systemctl edit postgresql-14.service`  
`[Service]`  
`Environment=PGDATA=/mnt/data/data/`  
>попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start  
>напишите получилось или нет и почему

Запускаем сервис, все в порядке
```console
systemctl start postgresql-14.service
systemctl status postgresql-14  
...  
Oct 13 17:12:18 instance-2 systemd[1]: Starting PostgreSQL 14 database server...
...  
```
>зайдите через через psql и проверьте содержимое ранее созданной таблицы


>задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
>ДЗ оформите в markdown на github с описанием что делали и с какими проблемами столкнулись. Также приложите имя Гугл проекта, где пользователь ifti@yandex.ru добавлен в роли project editor.
>
>Критерии оценки:
>Выполнение ДЗ: 10 баллов
>
>5 баллов за задание со *
>2 балла за красивое решение
>2 балла за рабочее решение, и недостатки указанные преподавателем не устранены



Установка и настройка PostgreSQL

Цель:
создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
переносить содержимое базы данных PostgreSQL на дополнительный диск
переносить содержимое БД PostgreSQL между виртуальными машинами
создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a
поставьте на нее PostgreSQL через sudo apt
проверьте что кластер запущен через sudo -u postgres pg_lsclusters
зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым postgres=# create table test(c1 text); postgres=# insert into test values('1'); \q
остановите postgres например через sudo -u postgres pg_ctlcluster 13 main stop
создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB
добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
перенесите содержимое /var/lib/postgres/13 в /mnt/data - mv /var/lib/postgresql/13 /mnt/data
попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
напишите получилось или нет и почему
задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его
напишите что и почему поменяли
попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
напишите получилось или нет и почему
зайдите через через psql и проверьте содержимое ранее созданной таблицы
задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
ДЗ оформите в markdown на github с описанием что делали и с какими проблемами столкнулись. Также приложите имя Гугл проекта, где пользователь ifti@yandex.ru добавлен в роли project editor.

Критерии оценки:
Выполнение ДЗ: 10 баллов

5 баллов за задание со *
2 балла за красивое решение
2 балла за рабочее решение, и недостатки указанные преподавателем не устранены
