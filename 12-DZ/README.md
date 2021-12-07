# Работа с индексами, join'ами, статистикой

>Цель:
>знать и уметь применять основные виды индексов PostgreSQL
>строить и анализировать план выполнения запроса
>уметь оптимизировать запросы для с использованием индексов
>знать и уметь применять различные виды join'ов
>строить и анализировать план выполенения запроса
>оптимизировать запрос
>уметь собирать и анализировать статистику для таблицы
# 1 вариант:
>Создать индексы на БД, которые ускорят доступ к данным.
>В данном задании тренируются навыки:
>определения узких мест
>написания запросов 

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
gsutil -m cp gs://my_alex_bucket/taxi/taxi_00000000000*.csv  chicago_taxi_trips
...
Operation completed over 10 objects/2.5 GiB.
\c taxi
create database taxi;
create table taxi_trips (
unique_key text, 
taxi_id text, 
trip_start_timestamp TIMESTAMP, 
trip_end_timestamp TIMESTAMP, 
trip_seconds bigint, 
trip_miles numeric, 
pickup_census_tract bigint, 
dropoff_census_tract bigint, 
pickup_community_area bigint, 
dropoff_community_area bigint, 
fare numeric, 
tips numeric, 
tolls numeric, 
extras numeric, 
trip_total numeric, 
payment_type text, 
company text, 
pickup_latitude numeric, 
pickup_longitude numeric, 
pickup_location text, 
dropoff_latitude numeric, 
dropoff_longitude numeric, 
dropoff_location text
);

Processing chicago_taxi_trips/taxi_000000000000.csv file...
COPY 653941
Processing chicago_taxi_trips/taxi_000000000001.csv file...
...
real    3m41.720s
user    0m3.896s
sys     0m4.146s

taxi=# \timing
Timing is on.
taxi=# select count (*) from taxi_trips;
  count
---------
 6399680
(1 row)
Time: 42498.244 ms (00:42.498)
taxi=# select pg_size_pretty(pg_table_size('taxi_trips'));
 pg_size_pretty
----------------
 2716 MB
(1 row)

Time: 0.632 ms

>для создания индекса оптимизации 
>Необходимо:
>Создать индекс к какой-либо из таблиц вашей БД
>Прислать текстом результат команды explain, в которой используется данный индекс

taxi=# explain
taxi-# select taxi_id, count(*) from taxi_trips
taxi-# group by taxi_id;
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=388949.72..390338.83 rows=5483 width=139)
   Group Key: taxi_id
   ->  Gather Merge  (cost=388949.72..390229.17 rows=10966 width=139)
         Workers Planned: 2
         ->  Sort  (cost=387949.69..387963.40 rows=5483 width=139)
               Sort Key: taxi_id
               ->  Partial HashAggregate  (cost=387554.35..387609.18 rows=5483 width=139)
                     Group Key: taxi_id
                     ->  Parallel Seq Scan on taxi_trips  (cost=0.00..374224.23 rows=2666023 width=131)
(9 rows)

taxi=# create index idx_taxi_id on taxi_trips(taxi_id);
CREATE INDEX
Time: 82401.612 ms (01:22.402)
taxi=# select pg_size_pretty(pg_table_size('idx_taxi_id'));
 pg_size_pretty
----------------
 54 MB
(1 row)

Time: 0.487 ms

taxi=# explain
taxi-# select taxi_id, count(*) from taxi_trips
taxi-# group by taxi_id;
                                                        QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------
--
 Finalize GroupAggregate  (cost=1000.58..242007.21 rows=5483 width=139)
   Group Key: taxi_id
   ->  Gather Merge  (cost=1000.58..241897.55 rows=10966 width=139)
         Workers Planned: 2
         ->  Partial GroupAggregate  (cost=0.56..239631.78 rows=5483 width=139)
               Group Key: taxi_id
               ->  Parallel Index Only Scan using idx_taxi_id on taxi_trips  (cost=0.56..226244.29 rows=2666533 width=131
)
(7 rows)

Time: 1.214 ms

>Реализовать индекс для полнотекстового поиска
taxi=# select company, to_tsvector(company) @@ to_tsquery('Chicago') from taxi_trips;
                   company                    | ?column?
----------------------------------------------+----------
 Taxi Affiliation Services                    | f
 Taxi Affiliation Services                    | f
 Taxi Affiliation Services                    | f
 ...
 Chicago Medallion Leasing INC                | t
 ...

taxi=# explain
taxi-# select company, to_tsvector(company) @@ to_tsquery('Chicago') from taxi_trips;
                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Gather  (cost=1000.00..2355130.17 rows=6399680 width=25)
   Workers Planned: 2
   ->  Parallel Seq Scan on taxi_trips  (cost=0.00..1714162.17 rows=2666533 width=25)
(3 rows)

Time: 0.962 ms

taxi=# alter table taxi_trips add column search_company tsvector;
ALTER TABLE
Time: 3.887 ms
taxi=# update taxi_trips set search_company = to_tsvector(company);
UPDATE 6399680
Time: 561282.034 ms (09:21.282)

taxi=# select pg_size_pretty(pg_table_size('taxi_trips'));
 pg_size_pretty
----------------
 5753 MB
(1 row)

taxi=# create index search_index_company on taxi_trips using gin (search_company);
CREATE INDEX
Time: 192626.826 ms (03:12.627)

taxi=# select pg_size_pretty(pg_table_size('search_index_company'));
 pg_size_pretty
----------------
 27 MB
(1 row)

taxi=# explain
select company from taxi_trips where search_company @@ to_tsquery('Chicago');
                                           QUERY PLAN
-------------------------------------------------------------------------------------------------
 Gather  (cost=7733.33..1501304.61 rows=729430 width=24)
   Workers Planned: 2
   ->  Parallel Bitmap Heap Scan on taxi_trips  (cost=6733.33..1427361.61 rows=303929 width=24)
         Recheck Cond: (search_company @@ to_tsquery('Chicago'::text))
         ->  Bitmap Index Scan on search_index_company  (cost=0.00..6550.97 rows=729430 width=0)
               Index Cond: (search_company @@ to_tsquery('Chicago'::text))
(6 rows)

taxi=# explain
taxi-# select company, count(*) from taxi_trips where search_company @@ to_tsquery('Chicago') group by company;
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=1429885.65..1429911.24 rows=101 width=32)
   Group Key: company
   ->  Gather Merge  (cost=1429885.65..1429909.22 rows=202 width=32)
         Workers Planned: 2
         ->  Sort  (cost=1428885.63..1428885.88 rows=101 width=32)
               Sort Key: company
               ->  Partial HashAggregate  (cost=1428881.25..1428882.26 rows=101 width=32)
                     Group Key: company
                     ->  Parallel Bitmap Heap Scan on taxi_trips  (cost=6733.33..1427361.61 rows=303929 width=24)
                           Recheck Cond: (search_company @@ to_tsquery('Chicago'::text))
                           ->  Bitmap Index Scan on search_index_company  (cost=0.00..6550.97 rows=729430 width=0)
                                 Index Cond: (search_company @@ to_tsquery('Chicago'::text))
(12 rows)


taxi=# select company, count(*) from taxi_trips where search_company @@ to_tsquery('Chicago') group by company;
                 company                  | count
------------------------------------------+--------
 Chicago Carriage Cab Corp                | 447119
 Chicago Elite Cab Corp.                  |      9
 Chicago Elite Cab Corp. (Chicago Carriag | 175334
 Chicago Independents                     |  33043
 Chicago Medallion Leasing INC            |  41319
 Chicago Medallion Management             |  23600
 Chicago Star Taxicab                     |    311
 Chicago Taxicab                          |  36838
(8 rows)

Time: 23647.089 ms (00:23.647)


>Реализовать индекс на часть таблицы или индекс на поле с функцией

taxi=# explain
taxi-# select count(*) from taxi_trips
taxi-# where trip_start_timestamp::date between date '2016-01-01' and date'2016-02-01';
                                                                 QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------
--------------------
 Finalize Aggregate  (cost=790718.02..790718.03 rows=1 width=8)
   ->  Gather  (cost=790717.80..790718.01 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=789717.80..789717.81 rows=1 width=8)
               ->  Parallel Seq Scan on taxi_trips  (cost=0.00..789684.35 rows=13381 width=0)
                     Filter: (((trip_start_timestamp)::date >= '2016-01-01'::date) AND ((trip_start_timestamp)::date <= '
2016-02-01'::date))
(6 rows)

taxi=# create index idx_taxi_start_date_part on taxi_trips(trip_start_timestamp)
taxi-# where trip_start_timestamp::date between date '2016-01-01' and date'2016-12-31';
CREATE INDEX
Time: 180377.302 ms (03:00.377)

taxi=# select pg_size_pretty(pg_table_size('idx_taxi_start_date_part'));
 pg_size_pretty
----------------
 19 MB
(1 row)

taxi=# explain
taxi-# select count(*) from taxi_trips
taxi-# where trip_start_timestamp::date between date '2016-01-01' and date'2016-02-01';
                                                 QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=836.38..836.39 rows=1 width=8)
   ->  Index Only Scan using idx_taxi_start_date_part on taxi_trips  (cost=0.43..756.39 rows=31998 width=0)
         Filter: ((trip_start_timestamp)::date <= '2016-02-01'::date)
(3 rows)

taxi=# explain
taxi-# select count(*) from taxi_trips
taxi-# where trip_start_timestamp::date between date '2016-11-01' and date'2017-01-02';
                                                                 QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------
--------------------
 Finalize Aggregate  (cost=790524.21..790524.22 rows=1 width=8)
   ->  Gather  (cost=790524.00..790524.21 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=789524.00..789524.01 rows=1 width=8)
               ->  Parallel Seq Scan on taxi_trips  (cost=0.00..789490.67 rows=13332 width=0)
                     Filter: (((trip_start_timestamp)::date >= '2016-11-01'::date) AND ((trip_start_timestamp)::date <= '
2017-01-02'::date))
(6 rows)




>Создать индекс на несколько полей

taxi=# explain
taxi-# select taxi_id, trip_end_timestamp::date as trip_end_date, count(*) from taxi_trips
taxi-# group by taxi_id, trip_end_timestamp::date;
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=1419664.87..1619665.30 rows=639968 width=143)
   Group Key: taxi_id, ((trip_end_timestamp)::date)
   ->  Gather Merge  (cost=1419664.87..1602066.18 rows=1279936 width=143)
         Workers Planned: 2
         ->  Partial GroupAggregate  (cost=1418664.85..1453329.78 rows=639968 width=143)
               Group Key: taxi_id, ((trip_end_timestamp)::date)
               ->  Sort  (cost=1418664.85..1425331.18 rows=2666533 width=135)
                     Sort Key: taxi_id, ((trip_end_timestamp)::date)
                     ->  Parallel Seq Scan on taxi_trips  (cost=0.00..769491.67 rows=2666533 width=135)
(9 rows)

taxi=# create index idx_taxi_id_end_date on taxi_trips(taxi_id, trip_end_timestamp);
CREATE INDEX
Time: 295943.727 ms (04:55.944)

taxi=# select pg_size_pretty(pg_table_size('idx_taxi_id_end_date'));
 pg_size_pretty
----------------
 1054 MB
(1 row)

taxi=# explain
taxi-# select taxi_id, trip_end_timestamp::date as trip_end_date, count(*) from taxi_trips
taxi-# group by taxi_id, trip_end_timestamp::date;
                                                                QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------
-----------------
 Finalize GroupAggregate  (cost=1145.48..1026233.61 rows=639968 width=143)
   Group Key: taxi_id, ((trip_end_timestamp)::date)
   ->  Gather Merge  (cost=1145.48..1008634.49 rows=1279936 width=143)
         Workers Planned: 2
         ->  Partial GroupAggregate  (cost=145.46..859898.09 rows=639968 width=143)
               Group Key: taxi_id, ((trip_end_timestamp)::date)
               ->  Incremental Sort  (cost=145.46..831899.49 rows=2666533 width=135)
                     Sort Key: taxi_id, ((trip_end_timestamp)::date)
                     Presorted Key: taxi_id
                     ->  Parallel Index Only Scan using idx_taxi_id_end_date on taxi_trips  (cost=0.68..604966.75 rows=26
66533 width=135)
(10 rows)

>Написать комментарии к каждому из индексов
>Описать что и как делали и с какими проблемами столкнулись 
# 2 вариант: 
>В результате выполнения ДЗ вы научитесь пользоваться различными вариантами соединения таблиц. В данном задании тренируются навыки:
>написания запросов с различными типами соединений Необходимо:
>Реализовать прямое соединение двух или более таблиц
>Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
>Реализовать кросс соединение двух или более таблиц
>Реализовать полное соединение двух или более таблиц
>Реализовать запрос, в котором будут использованы разные типы соединений
>Сделать комментарии на каждый запрос
>К работе приложить структуру таблиц, для которых выполнялись соединения
>Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке
