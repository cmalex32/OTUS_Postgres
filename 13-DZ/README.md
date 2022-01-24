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
Вытягиваем демо базу flights
```console
wget https://edu.postgrespro.ru/demo-big.zip
--2021-12-14 11:15:08--  https://edu.postgrespro.ru/demo-big.zip
Resolving edu.postgrespro.ru (edu.postgrespro.ru)... 93.174.131.139
Connecting to edu.postgrespro.ru (edu.postgrespro.ru)|93.174.131.139|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 243203214 (232M) [application/zip]
Saving to: ‘demo-big.zip’

100%[================================================>] 243,203,214 9.67MB/s   in 26s    

2021-12-14 11:15:36 (8.83 MB/s) - ‘demo-big.zip’ saved [243203214/243203214]
```
Устанавливаем unzip
```console
yum install unzip -y
```
Распаковываем
```console
unzip demo-big.zip 
Archive:  demo-big.zip
  inflating: demo-big-20170815.sql   
```
Заливаем в базу
```console
psql -f demo-big-20170815.sql 
```
Посмотрим таблицы
```console
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
```
Смотрим структуру таблицы bookings.flights
```console
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
 ```
 
demo=# explain analyze  select * from bookings.flights where scheduled_arrival between date'2017-01-01' and date'2017-02-01' - 1;
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Seq Scan on flights  (cost=0.00..5847.00 rows=17063 width=63) (actual time=0.042..37.618 rows=16286 loops=1)
   Filter: ((scheduled_arrival >= '2017-01-01'::date) AND (scheduled_arrival <= '2017-01-31'::date))
   Rows Removed by Filter: 198581
 Planning Time: 0.324 ms
 Execution Time: 38.336 ms
(5 rows)

create table bookings.flights2 (like bookings.flights including all);
CREATE TABLE table0_2020_02 (check (create_date between date'2020-02-01' and date'2020-03-01' - 1)) INHERITS (table0);
create table bookings.flights2_2017_01 (check (scheduled_arrival between date'2017-01-01' and date'2017-02-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_02 (check (scheduled_arrival between date'2017-02-01' and date'2017-03-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_03 (check (scheduled_arrival between date'2017-03-01' and date'2017-04-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_04 (check (scheduled_arrival between date'2017-04-01' and date'2017-05-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_05 (check (scheduled_arrival between date'2017-05-01' and date'2017-06-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_06 (check (scheduled_arrival between date'2017-06-01' and date'2017-07-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_07 (check (scheduled_arrival between date'2017-07-01' and date'2017-08-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_08 (check (scheduled_arrival between date'2017-08-01' and date'2017-09-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_09 (check (scheduled_arrival between date'2017-09-01' and date'2017-10-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_10 (check (scheduled_arrival between date'2017-10-01' and date'2017-11-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_11 (check (scheduled_arrival between date'2017-11-01' and date'2017-12-01' - 1)) inherits (bookings.flights2);
create table bookings.flights2_2017_12 (check (scheduled_arrival between date'2017-12-01' and date'2018-01-01' - 1)) inherits (bookings.flights2);


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
    elsif new.scheduled_arrival between date'2017-06-01' and date'2017-07-01' - 1 then
        INSERT INTO bookings.flights2_2017_06 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-07-01' and date'2017-08-01' - 1 then
        INSERT INTO bookings.flights2_2017_07 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-08-01' and date'2017-09-01' - 1 then
        INSERT INTO bookings.flights2_2017_08 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-09-01' and date'2017-10-01' - 1 then
        INSERT INTO bookings.flights2_2017_09 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-10-01' and date'2017-11-01' - 1 then
        INSERT INTO bookings.flights2_2017_10 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-11-01' and date'2017-12-01' - 1 then
        INSERT INTO bookings.flights2_2017_11 VALUES (NEW.*);
    elsif new.scheduled_arrival between date'2017-12-01' and date'2018-01-01' - 1 then
        INSERT INTO bookings.flights2_2017_12 VALUES (NEW.*);
    else
        raise exception 'this date not in your partitions. add partition';
    end if;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER check_date_flights2
    BEFORE INSERT ON bookings.flights2
    FOR EACH ROW EXECUTE PROCEDURE bookings.flights2_part();

insert into bookings.flights2 select * from bookings.flights where scheduled_arrival between date'2017-01-01' and date'2018-01-01' - 1 ;

  
  
  
  
  
  explain analyze select ac.model,ap.airport_name,count(tf.*) c from aircrafts ac, airports ap, flights fl, ticket_flights tf where ac.aircraft_code = fl.aircraft_code and 
ap.airport_code = fl.departure_airport and tf.flight_id=fl.flight_id group by ac.model,ap.airport_name order by c desc ;
                                                                          QUERY PLAN                                                                          
--------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=4722420.17..4722422.51 rows=936 width=72) (actual time=36172.950..36172.986 rows=219 loops=1)
   Sort Key: (count(tf.*)) DESC
   Sort Method: quicksort  Memory: 51kB
   ->  HashAggregate  (cost=4721891.94..4722373.98 rows=936 width=72) (actual time=36172.794..36172.868 rows=219 loops=1)
         Group Key: (ml.model ->> lang()), (ml_1.airport_name ->> lang())
         Batches: 1  Memory Usage: 97kB
         ->  Hash Join  (cost=8515.05..4658953.05 rows=8391852 width=120) (actual time=77.930..33433.440 rows=8391852 loops=1)
               Hash Cond: (fl.departure_airport = ml_1.airport_code)
               ->  Hash Join  (cost=8509.71..398136.15 rows=8391852 width=92) (actual time=77.816..9849.760 rows=8391852 loops=1)
                     Hash Cond: (fl.aircraft_code = ml.aircraft_code)
                     ->  Hash Join  (cost=8508.51..365733.07 rows=8391852 width=64) (actual time=77.801..7773.120 rows=8391852 loops=1)
                           Hash Cond: (tf.flight_id = fl.flight_id)
                           ->  Seq Scan on ticket_flights tf  (cost=0.00..153851.52 rows=8391852 width=60) (actual time=0.019..2418.891 rows=8391852 loops=1)
                           ->  Hash  (cost=4772.67..4772.67 rows=214867 width=12) (actual time=77.652..77.653 rows=214867 loops=1)
                                 Buckets: 131072  Batches: 4  Memory Usage: 3338kB
                                 ->  Seq Scan on flights fl  (cost=0.00..4772.67 rows=214867 width=12) (actual time=0.005..34.402 rows=214867 loops=1)
                     ->  Hash  (cost=1.09..1.09 rows=9 width=48) (actual time=0.006..0.007 rows=9 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 9kB
                           ->  Seq Scan on aircrafts_data ml  (cost=0.00..1.09 rows=9 width=48) (actual time=0.003..0.005 rows=9 loops=1)
               ->  Hash  (cost=4.04..4.04 rows=104 width=65) (actual time=0.052..0.052 rows=104 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 18kB
                     ->  Seq Scan on airports_data ml_1  (cost=0.00..4.04 rows=104 width=65) (actual time=0.011..0.028 rows=104 loops=1)
 Planning Time: 0.501 ms
 Execution Time: 36173.199 ms
(24 rows)


demo=# explain analyze select ac.model,ap.airport_name,fl.scheduled_arrival::date,count(tf.*) c from aircrafts ac, airports ap, flights fl, ticket_flights tf where ac.aircraft_code = fl.aircraft_code and ap.airport_code = fl.departure_airport and tf.flight_id=fl.flight_id group by ac.model,ap.airport_name,fl.scheduled_arrival::date order by c desc ;
                                                                             QUERY PLAN                                                                             
--------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=13438915.51..13459895.14 rows=8391852 width=76) (actual time=115420.114..115431.311 rows=73560 loops=1)
   Sort Key: (count(tf.*)) DESC
   Sort Method: external merge  Disk: 5160kB
   ->  GroupAggregate  (cost=7280381.56..11728063.12 rows=8391852 width=76) (actual time=110874.763..115380.428 rows=73560 loops=1)
         Group Key: ((ml.model ->> lang())), ((ml_1.airport_name ->> lang())), ((fl.scheduled_arrival)::date)
         ->  Sort  (cost=7280381.56..7301361.19 rows=8391852 width=124) (actual time=110874.668..113784.234 rows=8391852 loops=1)
               Sort Key: ((ml.model ->> lang())), ((ml_1.airport_name ->> lang())), ((fl.scheduled_arrival)::date)
               Sort Method: external merge  Disk: 952600kB
               ->  Hash Join  (cost=8724.05..4680350.68 rows=8391852 width=124) (actual time=83.899..35180.591 rows=8391852 loops=1)
                     Hash Cond: (fl.departure_airport = ml_1.airport_code)
                     ->  Hash Join  (cost=8718.71..398554.15 rows=8391852 width=100) (actual time=83.781..10309.994 rows=8391852 loops=1)
                           Hash Cond: (fl.aircraft_code = ml.aircraft_code)
                           ->  Hash Join  (cost=8717.51..366151.07 rows=8391852 width=72) (actual time=83.761..8187.884 rows=8391852 loops=1)
                                 Hash Cond: (tf.flight_id = fl.flight_id)
                                 ->  Seq Scan on ticket_flights tf  (cost=0.00..153851.52 rows=8391852 width=60) (actual time=0.018..2368.398 rows=8391852 loops=1)
                                 ->  Hash  (cost=4772.67..4772.67 rows=214867 width=20) (actual time=83.649..83.652 rows=214867 loops=1)
                                       Buckets: 65536  Batches: 4  Memory Usage: 3244kB
                                       ->  Seq Scan on flights fl  (cost=0.00..4772.67 rows=214867 width=20) (actual time=0.006..36.306 rows=214867 loops=1)
                           ->  Hash  (cost=1.09..1.09 rows=9 width=48) (actual time=0.008..0.008 rows=9 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 ->  Seq Scan on aircrafts_data ml  (cost=0.00..1.09 rows=9 width=48) (actual time=0.004..0.005 rows=9 loops=1)
                     ->  Hash  (cost=4.04..4.04 rows=104 width=65) (actual time=0.051..0.052 rows=104 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 18kB
                           ->  Seq Scan on airports_data ml_1  (cost=0.00..4.04 rows=104 width=65) (actual time=0.012..0.028 rows=104 loops=1)
 Planning Time: 1.497 ms
 Execution Time: 115587.999 ms
(26 rows)


select distinct 'create table flights1_' || EXTRACT(YEAR from fl.scheduled_arrival::date) || '_' || to_char(scheduled
_arrival,'MM') || ' partition of table flights1 for values from (''' || to_char(scheduled_arrival,'YYYY-MM') || '-01'') to (
''' || to_char((scheduled_arrival::date + interval '1 months')::date,'YYYY-MM') || '-01'')' from flights fl;
                                                  ?column?                                                  
------------------------------------------------------------------------------------------------------------
 create table flights1_2016_10 partition of table flights1 for values from ('2016-10-01') to ('2016-11-01')
 create table flights1_2016_08 partition of table flights1 for values from ('2016-08-01') to ('2016-09-01')
 create table flights1_2016_09 partition of table flights1 for values from ('2016-09-01') to ('2016-10-01')
 create table flights1_2016_11 partition of table flights1 for values from ('2016-11-01') to ('2016-12-01')
 create table flights1_2017_02 partition of table flights1 for values from ('2017-02-01') to ('2017-03-01')
 create table flights1_2017_09 partition of table flights1 for values from ('2017-09-01') to ('2017-10-01')
 create table flights1_2016_12 partition of table flights1 for values from ('2016-12-01') to ('2017-01-01')
 create table flights1_2017_03 partition of table flights1 for values from ('2017-03-01') to ('2017-04-01')
 create table flights1_2017_08 partition of table flights1 for values from ('2017-08-01') to ('2017-09-01')
 create table flights1_2017_01 partition of table flights1 for values from ('2017-01-01') to ('2017-02-01')
 create table flights1_2017_04 partition of table flights1 for values from ('2017-04-01') to ('2017-05-01')
 create table flights1_2017_06 partition of table flights1 for values from ('2017-06-01') to ('2017-07-01')
 create table flights1_2017_05 partition of table flights1 for values from ('2017-05-01') to ('2017-06-01')
 create table flights1_2017_07 partition of table flights1 for values from ('2017-07-01') to ('2017-08-01')
 

