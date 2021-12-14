# Секционирование таблицы
>Цель:  
>научиться секционировать таблицы.  
>Секционировать большую таблицу из демо базы flights

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
Создаем каталог для данных, и меняем права на него
```console
mkdir /mnt/pgdata
chown postgres:postgres /mnt/pgdata
```
Инициализируем и запускаем сервер БД  
```console
/usr/pgsql-14/bin/initdb --locale en_US.UTF-8 -D /mnt/pgdata/14
/usr/pgsql-14/bin/pg_ctl -D /mnt/pgdata/14 start
```

-bash-4.2$ wget https://edu.postgrespro.ru/demo-big.zip
--2021-12-14 11:15:08--  https://edu.postgrespro.ru/demo-big.zip
Resolving edu.postgrespro.ru (edu.postgrespro.ru)... 93.174.131.139
Connecting to edu.postgrespro.ru (edu.postgrespro.ru)|93.174.131.139|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 243203214 (232M) [application/zip]
Saving to: ‘demo-big.zip’

100%[================================================>] 243,203,214 9.67MB/s   in 26s    

2021-12-14 11:15:36 (8.83 MB/s) - ‘demo-big.zip’ saved [243203214/243203214]

yum install unzip -y


bash-4.2$ unzip demo-big.zip 
Archive:  demo-big.zip
  inflating: demo-big-20170815.sql   
  
psql -f demo-big-20170815.sql 
SET
SET
SET
SET
...
ALTER TABLE
ALTER DATABASE
ALTER DATABASE

demo=# \dt+
                                                List of relations
  Schema  |      Name       | Type  |  Owner   | Persistence | Access method |  Size  |        Description        
----------+-----------------+-------+----------+-------------+---------------+--------+---------------------------
 bookings | aircrafts_data  | table | postgres | permanent   | heap          | 16 kB  | Aircrafts (internal data)
 bookings | airports_data   | table | postgres | permanent   | heap          | 56 kB  | Airports (internal data)
 bookings | boarding_passes | table | postgres | permanent   | heap          | 455 MB | Boarding passes
 bookings | bookings        | table | postgres | permanent   | heap          | 105 MB | Bookings
 bookings | flights         | table | postgres | permanent   | heap          | 21 MB  | Flights
 bookings | seats           | table | postgres | permanent   | heap          | 96 kB  | Seats
 bookings | ticket_flights  | table | postgres | permanent   | heap          | 547 MB | Flight segment
 bookings | tickets         | table | postgres | permanent   | heap          | 386 MB | Tickets
(8 rows)






  
