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
Скачиваем десяток файлов csv из chicago_taxi_trips
```console
gsutil -m cp gs://my_alex_bucket/taxi/taxi_00000000000*.csv  chicago_taxi_trips
...
Operation completed over 10 objects/2.5 GiB.
```
Создаем базу переключаемся в нее
```console
create database taxi;
\c taxi
```
Создаем таблицу taxi_trips
```console
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
```
Заливаем данные из файлов в созданную таблицу
```console
Processing chicago_taxi_trips/taxi_000000000000.csv file...
COPY 653941
Processing chicago_taxi_trips/taxi_000000000001.csv file...
...
real    3m41.720s
user    0m3.896s
sys     0m4.146s
```
Посмотрим солько получилось записей и размер таблицы
```console
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
```
>для создания индекса оптимизации 
>Необходимо:
>Создать индекс к какой-либо из таблиц вашей БД
>Прислать текстом результат команды explain, в которой используется данный индекс
Посмотрим план запроса подсчета количества выездов такси по их идентификаторам, видно что происходит полное сканирование таблицы Seq Scab
```console
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
```
Создадим индекс по групперуемому полю taxi_id
```console
taxi=# create index idx_taxi_id on taxi_trips(taxi_id);
CREATE INDEX
Time: 82401.612 ms (01:22.402)
```
Посмотрим его размер
```console
taxi=# select pg_size_pretty(pg_table_size('idx_taxi_id'));
 pg_size_pretty
----------------
 54 MB
(1 row)
```
Провераем план запроса, видно что теперь сканирование идет по индексу idx_taxi_id, а значит выполняется значительно быстрее
```console
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
```
>Реализовать индекс для полнотекстового поиска

Выполним запрос по поиску подстроки Chicago с приведением поля company к типу tsvector, убедимся что он правильно работает
```console
taxi=# select company, to_tsvector(company) @@ to_tsquery('Chicago') from taxi_trips;
                   company                    | ?column?
----------------------------------------------+----------
 Taxi Affiliation Services                    | f
 Taxi Affiliation Services                    | f
 Taxi Affiliation Services                    | f
 ...
 Chicago Medallion Leasing INC                | t
 ...
```
Посмотрим план этого запроса, конечно происходит полное сканирование таблицы Seq Scan
```console
taxi=# explain
taxi-# select company, to_tsvector(company) @@ to_tsquery('Chicago') from taxi_trips;
                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Gather  (cost=1000.00..2355130.17 rows=6399680 width=25)
   Workers Planned: 2
   ->  Parallel Seq Scan on taxi_trips  (cost=0.00..1714162.17 rows=2666533 width=25)
(3 rows)
```
Добавим в таблицу поле search_company типа tsvector и наполним его данными из поля company это поле необходимо для создания индекса типа gin  
```console
taxi=# alter table taxi_trips add column search_company tsvector;
ALTER TABLE
Time: 3.887 ms
taxi=# update taxi_trips set search_company = to_tsvector(company);
UPDATE 6399680
Time: 561282.034 ms (09:21.282)
```
Посмотрим насколько изменился размер таблицы было 2716 MB, размер увеличился более чем в два раза  
Достаточто ресурсоемкое использование данного функционала
```console
taxi=# select pg_size_pretty(pg_table_size('taxi_trips'));
 pg_size_pretty
----------------
 5753 MB
(1 row)
```
Для использования полнотекстового поиска создают индексы типа gin или gist, предпочтительным для такого рода поиска является тип gin  
Но для использования такого типа индекса надо создать поле типа tsvector, поэтому его и пришлось создать на предыдущем шаге  
Создаем индекс
```console
taxi=# create index search_index_company on taxi_trips using gin (search_company);
CREATE INDEX
Time: 192626.826 ms (03:12.627)
```
Псмотрим его размер, он достаточно небольшой
```console
taxi=# select pg_size_pretty(pg_table_size('search_index_company'));
 pg_size_pretty
----------------
 27 MB
(1 row)
```
Рассмотрим план запроса, выберем все компании имеющие в названии Chicago, только, конечно поле с индексом должно фигурировать в предикате  
Видно что сканирование идет по индексу
```console
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
```
Посмотрим еще один план запроса, применим группировку по названию компании и выведем количество такси используемых в них  
Видно что также используется сканирование индекса, а значит запрос будет гораздо быстрее
```console
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
```
Посмотрим насколько рельно быстро работает такой запрос  
Запрос выполнился за 23,6 секунды, это очень быстро для данной таблицы
```console
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
```
>Реализовать индекс на часть таблицы или индекс на поле с функцией

Рассмотрим план запроса с использованием подсчета количества выехавших такси в течении дат от 2016-01-01 до 2016-02-01 включительно  
Видим что используется полное сканирование таблицы
```console
taxi=# explain
taxi-# select count(*) from taxi_trips
taxi-# where trip_start_timestamp::date between date '2016-01-01' and date '2016-02-01';
                                                                 QUERY PLAN

---------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=790718.02..790718.03 rows=1 width=8)
   ->  Gather  (cost=790717.80..790718.01 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=789717.80..789717.81 rows=1 width=8)
               ->  Parallel Seq Scan on taxi_trips  (cost=0.00..789684.35 rows=13381 width=0)
                     Filter: (((trip_start_timestamp)::date >= '2016-01-01'::date) AND ((trip_start_timestamp)::date <= '
2016-02-01'::date))
(6 rows)
```
Создадим индекс с предикатом по данным выезда такси за 2016 год
```console
taxi=# create index idx_taxi_start_date_part on taxi_trips(trip_start_timestamp)
taxi-# where trip_start_timestamp::date between date '2016-01-01' and date'2016-12-31';
CREATE INDEX
Time: 180377.302 ms (03:00.377)
```
Посмотрим его размер, оно достаточно небольшой
```console
taxi=# select pg_size_pretty(pg_table_size('idx_taxi_start_date_part'));
 pg_size_pretty
----------------
 19 MB
(1 row)
```
Проверяеи план запроса с индексом. Как и ожидалось идет сканирование по индексу, так как предикат запроса попал в диапазон созданного индекса
```console
taxi=# explain
taxi-# select count(*) from taxi_trips
taxi-# where trip_start_timestamp::date between date '2016-01-01' and date'2016-02-01';
                                                 QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=836.38..836.39 rows=1 width=8)
   ->  Index Only Scan using idx_taxi_start_date_part on taxi_trips  (cost=0.43..756.39 rows=31998 width=0)
         Filter: ((trip_start_timestamp)::date <= '2016-02-01'::date)
(3 rows)
```
Посмотрим теперь план запроса где диапазон в предикате выходит за рамки созданного индекса на пару дней  
Как и ожидалось идет полное сканирование таблицы
```console
taxi=# explain
taxi-# select count(*) from taxi_trips
taxi-# where trip_start_timestamp::date between date '2016-11-01' and date'2017-01-02';
                                                                 QUERY PLAN

---------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=790524.21..790524.22 rows=1 width=8)
   ->  Gather  (cost=790524.00..790524.21 rows=2 width=8)
         Workers Planned: 2
         ->  Partial Aggregate  (cost=789524.00..789524.01 rows=1 width=8)
               ->  Parallel Seq Scan on taxi_trips  (cost=0.00..789490.67 rows=13332 width=0)
                     Filter: (((trip_start_timestamp)::date >= '2016-11-01'::date) AND ((trip_start_timestamp)::date <= '
2017-01-02'::date))
(6 rows)
```
>Создать индекс на несколько полей

Посмотрим план запроса сгруппированный по идентификатору такси и времени приезда  
Конечно идет полное сканирование таблицы, несмотря на то что по полю taxi_id создан индекс
```console
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
```
Создадим составной индекс по двум полям используемых для группировки в предыдущем запросе taxi_id и trip_end_timestamp
```console
taxi=# create index idx_taxi_id_end_date on taxi_trips(taxi_id, trip_end_timestamp);
CREATE INDEX
Time: 295943.727 ms (04:55.944)
```
Посмотрим размер индекса, его размер достаточно ресурсоемок и составляет от исходного размера больше трети таблицы
```console
taxi=# select pg_size_pretty(pg_table_size('idx_taxi_id_end_date'));
 pg_size_pretty
----------------
 1054 MB
(1 row)
```
Проверяем план запроса и убеждаемся что теперь используется сканирование созданного составного индекса
```console
taxi=# explain
taxi-# select taxi_id, trip_end_timestamp::date as trip_end_date, count(*) from taxi_trips
taxi-# group by taxi_id, trip_end_timestamp::date;
                                                                QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------------------
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
```
>Написать комментарии к каждому из индексов
>Описать что и как делали и с какими проблемами столкнулись 

Полнотекстовым поиском раньше не привелось на практике пользоваться, пришлось изучать  
С датами долго провозился, не мог понять почему при построении плана запроса count(*) возвращал одну запись :)  


# 2 вариант: 
>В результате выполнения ДЗ вы научитесь пользоваться различными вариантами соединения таблиц. В данном задании тренируются навыки:
>написания запросов с различными типами соединений Необходимо:

Будем использовать для создания тестовых таблиц созданную в первом варианте таблицу taxi_trips
Для увеличения скорости выборки создадим индекс
```console
create index idx_company on taxi_trips(company);
```
Создадим таблицу company1 с нумерацией по строкам от 1 до 10
```console
taxi=# create table company1 as select row_number () over () num, t.company from (select distinct company from taxi_trips where company is not null limit 10) t;
SELECT 10
Time: 6.785 ms
```
Создадим таблицу company2 с нумерацией по строкам от 6 до 15, она пересекается данными с первой таблицей
```console
taxi=# create table company2 as select (row_number () over ())::int+5 num, t.company from (select distinct company from taxi_trips where company is not null offset 
5 limit 10) t;
SELECT 10
Time: 5.492 ms
```
Создадим таблицу, непересекающуюся данными ни с какой из таблиц, company3 с нумерацией по строкам от 16 до 25
```console
taxi=# create table company3 as select (row_number () over ())::int+15 num, t.company from (select distinct company from taxi_trips where company is not null offset
 15 limit 10) t;
SELECT 10
Time: 10.556 ms
```console
Посмотрим содержимое созданных таблиц. Все должно быть как задумано.
```console
taxi=# select * from company1;
 num        |             company             
------------+---------------------------------
          1 | 0118 - 42111 Godfrey S.Awir
          2 | 0118 - Godfrey S.Awir
          3 | 0694 - 59280 Chinesco Trans Inc
          4 | 0694 - Chinesco Trans Inc
          5 | 1085 - 72312 N and W Cab Co
          6 | 1085 - N and W Cab Co
          7 | 1247 - 72807 Daniel Ayertey
          8 | 1247 - Daniel Ayertey
          9 | 1408 - 89599 Donald Barnes
         10 | 1408 - Donald Barnes
(10 rows)
taxi=# select * from company2;
 num      |                  company                  
----------+-------------------------------------------
        6 | 1085 - N and W Cab Co
        7 | 1247 - 72807 Daniel Ayertey
        8 | 1247 - Daniel Ayertey
        9 | 1408 - 89599 Donald Barnes
       10 | 1408 - Donald Barnes
       11 | 2092 - 61288 Sbeih company
       12 | 2092 - Sbeih company
       13 | 2192 - 73487 Zeymane Corp
       14 | 2192 - Zeymane Corp
       15 | 2241 - 44667 - Felman Corp, Manuel Alonso
(10 rows)
taxi=# select * from company3;
 num      |            company             
----------+--------------------------------
       16 | 2241 - 44667 Manuel Alonso
       17 | 2241 - Manuel Alonso
       18 | 24 Seven Taxi
       19 | 2733 - 74600 Benny Jona
       20 | 2733 - Benny Jona
       21 | 2767 - Sayed M Badri
       22 | 2809 - 95474 C & D Cab Co Inc.
       23 | 2809 - 95474 C&D Cab Co Inc.
       24 | 2823 - 73307 Lee Express Inc
       25 | 2823 - 73307 Seung Lee
(10 rows)
```console
>Реализовать прямое соединение двух или более таблиц

Выполним селект прямого соединения двух таблиц (join при этом можно не указывать). В обеих таблицах есть одинаковые данные по полю num с 6 до 10.
```console
taxi=# select * from company1 a, company2 b  where a.num=b.num;
 num |           company           | num |           company           
-----+-----------------------------+-----+-----------------------------
   6 | 1085 - N and W Cab Co       |   6 | 1085 - N and W Cab Co
   7 | 1247 - 72807 Daniel Ayertey |   7 | 1247 - 72807 Daniel Ayertey
   8 | 1247 - Daniel Ayertey       |   8 | 1247 - Daniel Ayertey
   9 | 1408 - 89599 Donald Barnes  |   9 | 1408 - 89599 Donald Barnes
  10 | 1408 - Donald Barnes        |  10 | 1408 - Donald Barnes
(5 rows)
```console
Проделаем туже операцию с тремя таблицами, так как в третьей таблице нет пересечения по полю num ни с одной таблицей, получаем нулевой редзультат.
```console
taxi=# select * from company1 a, company2 b, company3 c where a.num=b.num and a.num=c.num;
 num | company | num | company | num | company 
-----+---------+-----+---------+-----+---------
(0 rows)
```
>Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

Выполняем запрос левостороннего соединения двух первых таблиц, так как соединение "левое" выводятся все строки первой таблицы и данные со второй у которых есть совпадения по полю num
```console
taxi=# select * from company1 a left join company2 b on a.num=b.num;
 num |             company             | num |           company           
-----+---------------------------------+-----+-----------------------------
   1 | 0118 - 42111 Godfrey S.Awir     |     | 
   2 | 0118 - Godfrey S.Awir           |     | 
   3 | 0694 - 59280 Chinesco Trans Inc |     | 
   4 | 0694 - Chinesco Trans Inc       |     | 
   5 | 1085 - 72312 N and W Cab Co     |     | 
   6 | 1085 - N and W Cab Co           |   6 | 1085 - N and W Cab Co
   7 | 1247 - 72807 Daniel Ayertey     |   7 | 1247 - 72807 Daniel Ayertey
   8 | 1247 - Daniel Ayertey           |   8 | 1247 - Daniel Ayertey
   9 | 1408 - 89599 Donald Barnes      |   9 | 1408 - 89599 Donald Barnes
  10 | 1408 - Donald Barnes            |  10 | 1408 - Donald Barnes
(10 rows)
```
Сделаем аналогичный запрос по трем таблицам. Та как в третьей нет пересекающихся данных по полю num, выведуться пустые значения.
```console
taxi=# select * from company1 a left join company2 b on a.num=b.num left join company3 c on a.num=c.num;
 num |             company             | num |           company           | num | company 
-----+---------------------------------+-----+-----------------------------+-----+---------
   1 | 0118 - 42111 Godfrey S.Awir     |     |                             |     | 
   2 | 0118 - Godfrey S.Awir           |     |                             |     | 
   3 | 0694 - 59280 Chinesco Trans Inc |     |                             |     | 
   4 | 0694 - Chinesco Trans Inc       |     |                             |     | 
   5 | 1085 - 72312 N and W Cab Co     |     |                             |     | 
   6 | 1085 - N and W Cab Co           |   6 | 1085 - N and W Cab Co       |     | 
   7 | 1247 - 72807 Daniel Ayertey     |   7 | 1247 - 72807 Daniel Ayertey |     | 
   8 | 1247 - Daniel Ayertey           |   8 | 1247 - Daniel Ayertey       |     | 
   9 | 1408 - 89599 Donald Barnes      |   9 | 1408 - 89599 Donald Barnes  |     | 
  10 | 1408 - Donald Barnes            |  10 | 1408 - Donald Barnes        |     | 
(10 rows)
```
Выполним запрос с таким же левым соединением, но без вывода пустых значений по второй таблице. Вывод данные получился аналогичным прямому соединению.
```console
taxi=# select * from company1 a left join company2 b on a.num=b.num where b is not null;
 num |           company           | num |           company           
-----+-----------------------------+-----+-----------------------------
   6 | 1085 - N and W Cab Co       |   6 | 1085 - N and W Cab Co
   7 | 1247 - 72807 Daniel Ayertey |   7 | 1247 - 72807 Daniel Ayertey
   8 | 1247 - Daniel Ayertey       |   8 | 1247 - Daniel Ayertey
   9 | 1408 - 89599 Donald Barnes  |   9 | 1408 - 89599 Donald Barnes
  10 | 1408 - Donald Barnes        |  10 | 1408 - Donald Barnes
(5 rows)
```
Если мы скажем в запросе выводить все нулевые значения в результате соединеня (по полю num) выбранные по второй таблице, получим все строги непересекающиеся первой таблице со второй
```console
taxi=# select * from company1 a left join company2 b on a.num=b.num where b is  null;
 num |             company             | num | company 
-----+---------------------------------+-----+---------
   1 | 0118 - 42111 Godfrey S.Awir     |     | 
   2 | 0118 - Godfrey S.Awir           |     | 
   3 | 0694 - 59280 Chinesco Trans Inc |     | 
   4 | 0694 - Chinesco Trans Inc       |     | 
   5 | 1085 - 72312 N and W Cab Co     |     | 
(5 rows)
```console
>Реализовать кросс соединение двух или более таблиц

Соединённую таблицу образуют все возможные сочетания строк из T1 и T2 (т. е. их декартово произведение), а набор её столбцов объединяет в себе столбцы T1 со следующими за ними столбцами T2. Если таблицы содержат N и M строк, соединённая таблица будет содержать N * M строк.
При кросс соединении выборка образует все возможные сочетания строк из всех соединяемых таблиц, т е вывод такого запроса будет содержать количество строк равное перемноженному количеству строк всех участвующих в соединении таблиц
Соединяем две первых таблицы, получаем 10*10=100 строк.
  select * from company1 a cross join company2 b ;
```console
   9 | 1408 - 89599 Donald Barnes      |  12 | 2092 - Sbeih company
  10 | 1408 - Donald Barnes            |  12 | 2092 - Sbeih company
   1 | 0118 - 42111 Godfrey S.Awir     |  13 | 2192 - 73487 Zeymane Corp
   2 | 0118 - Godfrey S.Awir           |  13 | 2192 - 73487 Zeymane Corp
   3 | 0694 - 59280 Chinesco Trans Inc |  13 | 2192 - 73487 Zeymane Corp
...
   3 | 0694 - 59280 Chinesco Trans Inc |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   4 | 0694 - Chinesco Trans Inc       |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   5 | 1085 - 72312 N and W Cab Co     |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   6 | 1085 - N and W Cab Co           |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   7 | 1247 - 72807 Daniel Ayertey     |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   8 | 1247 - Daniel Ayertey           |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   9 | 1408 - 89599 Donald Barnes      |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
(100 rows)
```
Соединяем все три таблицы, получаем 10*10*10=1000 строк
```console
axi=# select * from company1 a cross join company2 b cross join company2 c;
 num |             company             | num |                  company                  | num |                  company                  
-----+---------------------------------+-----+-------------------------------------------+-----+-------------------------------------------
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |   6 | 1085 - N and W Cab Co
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |   7 | 1247 - 72807 Daniel Ayertey
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |   8 | 1247 - Daniel Ayertey
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |   9 | 1408 - 89599 Donald Barnes
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |  10 | 1408 - Donald Barnes
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |  11 | 2092 - 61288 Sbeih company
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |  12 | 2092 - Sbeih company
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |  13 | 2192 - 73487 Zeymane Corp
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |  14 | 2192 - Zeymane Corp
   1 | 0118 - 42111 Godfrey S.Awir     |   6 | 1085 - N and W Cab Co                     |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |   6 | 1085 - N and W Cab Co
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |   7 | 1247 - 72807 Daniel Ayertey
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |   8 | 1247 - Daniel Ayertey
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |   9 | 1408 - 89599 Donald Barnes
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |  10 | 1408 - Donald Barnes
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |  11 | 2092 - 61288 Sbeih company
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |  12 | 2092 - Sbeih company
   2 | 0118 - Godfrey S.Awir           |   6 | 1085 - N and W Cab Co                     |  13 | 2192 - 73487 Zeymane Corp
...
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |  10 | 1408 - Donald Barnes
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |  11 | 2092 - 61288 Sbeih company
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |  12 | 2092 - Sbeih company
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |  13 | 2192 - 73487 Zeymane Corp
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |  14 | 2192 - Zeymane Corp
  10 | 1408 - Donald Barnes            |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
(1000 rows)
```
>Реализовать полное соединение двух или более таблиц

Выполняем полное соединение двух первых таблиц по полю num, получаем все записи с данными из обоих с пересечением по указанному полю
```console
axi=# select * from company1 a full outer join company2 b on a.num=b.num;
 num |             company             | num |                  company                  
-----+---------------------------------+-----+-------------------------------------------
   1 | 0118 - 42111 Godfrey S.Awir     |     | 
   2 | 0118 - Godfrey S.Awir           |     | 
   3 | 0694 - 59280 Chinesco Trans Inc |     | 
   4 | 0694 - Chinesco Trans Inc       |     | 
   5 | 1085 - 72312 N and W Cab Co     |     | 
   6 | 1085 - N and W Cab Co           |   6 | 1085 - N and W Cab Co
   7 | 1247 - 72807 Daniel Ayertey     |   7 | 1247 - 72807 Daniel Ayertey
   8 | 1247 - Daniel Ayertey           |   8 | 1247 - Daniel Ayertey
   9 | 1408 - 89599 Donald Barnes      |   9 | 1408 - 89599 Donald Barnes
  10 | 1408 - Donald Barnes            |  10 | 1408 - Donald Barnes
     |                                 |  11 | 2092 - 61288 Sbeih company
     |                                 |  12 | 2092 - Sbeih company
     |                                 |  13 | 2192 - 73487 Zeymane Corp
     |                                 |  14 | 2192 - Zeymane Corp
     |                                 |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso
(15 rows)
```
Сделаем запрос полного соединения трех таблиц, получим вывод все строк таблиц, в том числе и третьей, хоть у нее данные не пересекаются по полю num
```console
taxi=# select * from company1 a full outer join company2 b on a.num=b.num full outer join company3 c on a.num=c.num;
 num |             company             | num |                  company                  | num |            company             
-----+---------------------------------+-----+-------------------------------------------+-----+--------------------------------
   1 | 0118 - 42111 Godfrey S.Awir     |     |                                           |     | 
   2 | 0118 - Godfrey S.Awir           |     |                                           |     | 
   3 | 0694 - 59280 Chinesco Trans Inc |     |                                           |     | 
   4 | 0694 - Chinesco Trans Inc       |     |                                           |     | 
   5 | 1085 - 72312 N and W Cab Co     |     |                                           |     | 
   6 | 1085 - N and W Cab Co           |   6 | 1085 - N and W Cab Co                     |     | 
   7 | 1247 - 72807 Daniel Ayertey     |   7 | 1247 - 72807 Daniel Ayertey               |     | 
   8 | 1247 - Daniel Ayertey           |   8 | 1247 - Daniel Ayertey                     |     | 
   9 | 1408 - 89599 Donald Barnes      |   9 | 1408 - 89599 Donald Barnes                |     | 
  10 | 1408 - Donald Barnes            |  10 | 1408 - Donald Barnes                      |     | 
     |                                 |  11 | 2092 - 61288 Sbeih company                |     | 
     |                                 |  12 | 2092 - Sbeih company                      |     | 
     |                                 |  13 | 2192 - 73487 Zeymane Corp                 |     | 
     |                                 |  14 | 2192 - Zeymane Corp                       |     | 
     |                                 |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |     | 
     |                                 |     |                                           |  16 | 2241 - 44667 Manuel Alonso
     |                                 |     |                                           |  17 | 2241 - Manuel Alonso
     |                                 |     |                                           |  18 | 24 Seven Taxi
     |                                 |     |                                           |  19 | 2733 - 74600 Benny Jona
     |                                 |     |                                           |  20 | 2733 - Benny Jona
     |                                 |     |                                           |  21 | 2767 - Sayed M Badri
     |                                 |     |                                           |  22 | 2809 - 95474 C & D Cab Co Inc.
     |                                 |     |                                           |  23 | 2809 - 95474 C&D Cab Co Inc.
     |                                 |     |                                           |  24 | 2823 - 73307 Lee Express Inc
     |                                 |     |                                           |  25 | 2823 - 73307 Seung Lee
(25 rows)
```console
>Реализовать запрос, в котором будут использованы разные типы соединений


taxi=# select * from company1 a right join company2 b on a.num=b.num full outer join company3 c on a.num=c.num ;
 num |           company           | num |                  company                  | num |            company             
-----+-----------------------------+-----+-------------------------------------------+-----+--------------------------------
   6 | 1085 - N and W Cab Co       |   6 | 1085 - N and W Cab Co                     |     | 
   7 | 1247 - 72807 Daniel Ayertey |   7 | 1247 - 72807 Daniel Ayertey               |     | 
   8 | 1247 - Daniel Ayertey       |   8 | 1247 - Daniel Ayertey                     |     | 
   9 | 1408 - 89599 Donald Barnes  |   9 | 1408 - 89599 Donald Barnes                |     | 
  10 | 1408 - Donald Barnes        |  10 | 1408 - Donald Barnes                      |     | 
     |                             |  11 | 2092 - 61288 Sbeih company                |     | 
     |                             |  12 | 2092 - Sbeih company                      |     | 
     |                             |  13 | 2192 - 73487 Zeymane Corp                 |     | 
     |                             |  14 | 2192 - Zeymane Corp                       |     | 
     |                             |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |     | 
     |                             |     |                                           |  16 | 2241 - 44667 Manuel Alonso
     |                             |     |                                           |  17 | 2241 - Manuel Alonso
     |                             |     |                                           |  18 | 24 Seven Taxi
     |                             |     |                                           |  19 | 2733 - 74600 Benny Jona
     |                             |     |                                           |  20 | 2733 - Benny Jona
     |                             |     |                                           |  21 | 2767 - Sayed M Badri
     |                             |     |                                           |  22 | 2809 - 95474 C & D Cab Co Inc.
     |                             |     |                                           |  23 | 2809 - 95474 C&D Cab Co Inc.
     |                             |     |                                           |  24 | 2823 - 73307 Lee Express Inc
     |                             |     |                                           |  25 | 2823 - 73307 Seung Lee
(20 rows)

taxi=# select * from company1 a right join company2 b on a.num=b.num full outer join company3 c on a.num=c.num where b is not null;
 num |           company           | num |                  company                  | num | company 
-----+-----------------------------+-----+-------------------------------------------+-----+---------
   6 | 1085 - N and W Cab Co       |   6 | 1085 - N and W Cab Co                     |     | 
   7 | 1247 - 72807 Daniel Ayertey |   7 | 1247 - 72807 Daniel Ayertey               |     | 
   8 | 1247 - Daniel Ayertey       |   8 | 1247 - Daniel Ayertey                     |     | 
   9 | 1408 - 89599 Donald Barnes  |   9 | 1408 - 89599 Donald Barnes                |     | 
  10 | 1408 - Donald Barnes        |  10 | 1408 - Donald Barnes                      |     | 
     |                             |  11 | 2092 - 61288 Sbeih company                |     | 
     |                             |  12 | 2092 - Sbeih company                      |     | 
     |                             |  13 | 2192 - 73487 Zeymane Corp                 |     | 
     |                             |  14 | 2192 - Zeymane Corp                       |     | 
     |                             |  15 | 2241 - 44667 - Felman Corp, Manuel Alonso |     | 
(10 rows)

taxi=# select * from company1 a left join company2 b on a.num=b.num full outer join company3 c on a.num=c.num where b is null;
 num |             company             | num | company | num |            company             
-----+---------------------------------+-----+---------+-----+--------------------------------
   1 | 0118 - 42111 Godfrey S.Awir     |     |         |     | 
   2 | 0118 - Godfrey S.Awir           |     |         |     | 
   3 | 0694 - 59280 Chinesco Trans Inc |     |         |     | 
   4 | 0694 - Chinesco Trans Inc       |     |         |     | 
   5 | 1085 - 72312 N and W Cab Co     |     |         |     | 
     |                                 |     |         |  16 | 2241 - 44667 Manuel Alonso
     |                                 |     |         |  17 | 2241 - Manuel Alonso
     |                                 |     |         |  18 | 24 Seven Taxi
     |                                 |     |         |  19 | 2733 - 74600 Benny Jona
     |                                 |     |         |  20 | 2733 - Benny Jona
     |                                 |     |         |  21 | 2767 - Sayed M Badri
     |                                 |     |         |  22 | 2809 - 95474 C & D Cab Co Inc.
     |                                 |     |         |  23 | 2809 - 95474 C&D Cab Co Inc.
     |                                 |     |         |  24 | 2823 - 73307 Lee Express Inc
     |                                 |     |         |  25 | 2823 - 73307 Seung Lee

>Сделать комментарии на каждый запрос
>К работе приложить структуру таблиц, для которых выполнялись соединения
>Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке

