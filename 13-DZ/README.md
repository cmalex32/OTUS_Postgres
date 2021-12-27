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

demo=# \d bookings.flights;
                                              Table "bookings.flights"
       Column        |           Type           | Collation | Nullable |                  Default                   
---------------------+--------------------------+-----------+----------+--------------------------------------------
 flight_id           | integer                  |           | not null | nextval('flights_flight_id_seq'::regclass)
 flight_no           | character(6)             |           | not null | 
 scheduled_departure | timestamp with time zone |           | not null | 
 scheduled_arrival   | timestamp with time zone |           | not null | 
 departure_airport   | character(3)             |           | not null | 
 arrival_airport     | character(3)             |           | not null | 
 status              | character varying(20)    |           | not null | 
 aircraft_code       | character(3)             |           | not null | 
 actual_departure    | timestamp with time zone |           |          | 
 actual_arrival      | timestamp with time zone |           |          | 
 
 
 demo=# explain  select * from bookings.flights where scheduled_arrival between date'2017-01-01' and date'2017-02-01' - 1;
                                             QUERY PLAN                                              
-----------------------------------------------------------------------------------------------------
 Seq Scan on flights  (cost=0.00..5847.00 rows=15618 width=63)
   Filter: ((scheduled_arrival >= '2017-01-01'::date) AND (scheduled_arrival <= '2017-01-31'::date))
(2 rows)

create table bookings.flights2 (like bookings.flights including all);

create table bookings.flights2_2017_01 (like bookings.flights2 including all) inherits (bookings.flights2);
create table bookings.flights2_2017_02 (like bookings.flights2 including all) inherits (bookings.flights2);
create table bookings.flights2_2017_03 (like bookings.flights2 including all) inherits (bookings.flights2);
create table bookings.flights2_2017_04 (like bookings.flights2 including all) inherits (bookings.flights2);
create table bookings.flights2_2017_05 (like bookings.flights2 including all) inherits (bookings.flights2);

alter table bookings.flights2_2017_01 add check ( scheduled_arrival between date'2017-01-01' and date'2017-02-01' - 1);
alter table bookings.flights2_2017_02 add check ( scheduled_arrival between date'2017-02-01' and date'2017-03-01' - 1);
alter table bookings.flights2_2017_03 add check ( scheduled_arrival between date'2017-03-01' and date'2017-04-01' - 1);
alter table bookings.flights2_2017_04 add check ( scheduled_arrival between date'2017-04-01' and date'2017-05-01' - 1);
alter table bookings.flights2_2017_05 add check ( scheduled_arrival between date'2017-05-01' and date'2017-06-01' - 1);

CREATE OR REPLACE FUNCTION bookings.flights2_part()
RETURNS TRIGGER AS $$
BEGIN
    if new.scheduled_arrival between date'2017-01-01' and date'2017-02-01' - 1 then
        INSERT INTO bookings.flights2_2017_01 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-02-01' and date'2017-03-01' - 1 then
        INSERT INTO bookings.flights2_2017_02 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-03-01' and date'2017-04-01' - 1 then
        INSERT INTO bookings.flights2_2017_03 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-04-01' and date'2017-05-01' - 1 then
        INSERT INTO bookings.flights2_2017_04 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-05-01' and date'2017-06-01' - 1 then
        INSERT INTO bookings.flights2_2017_05 VALUES (NEW.*);

    end if;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER check_date_flights2
    BEFORE INSERT ON bookings.flights2
    FOR EACH ROW EXECUTE PROCEDURE bookings.flights2_part();

insert into bookings.flights2 (select * from bookings.flights);

  
