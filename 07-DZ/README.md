# Работа с журналами

>Цель:  
>уметь работать с журналами и контрольными точками  
>уметь настраивать параметры журналов  
>Настройте выполнение контрольной точки раз в 30 секунд.  

Запускаем экземпляр. Подключаемся к БД. Выставляем параметр в базе данных.
```console
 gcloud beta compute ssh --zone "us-central1-a" "instance-2"  --project "clever-muse-328410"
 
 postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)
```
Включаем логирование checkpoint и проверяем
```console
postgres=# alter system set log_checkpoints = 'on';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
postgres=# show log_checkpoints;
 log_checkpoints
-----------------
 on
(1 row)
```
>10 минут c помощью утилиты pgbench подавайте нагрузку. 

```console
-bash-4.2$ /usr/pgsql-14/bin/pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.48 s (drop tables 0.04 s, create tables 0.03 s, client-side generate 0.23 s, vacuum 0.10 s, primary keys 0.08 s).
```
Чтобы не вычислять статистику сбросим ее
```console
postgres=# select pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared
----------------------

(1 row)

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+-----------------------------
checkpoints_timed     | 0
checkpoints_req       | 0
checkpoint_write_time | 0
checkpoint_sync_time  | 0
buffers_checkpoint    | 443
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 0
buffers_backend_fsync | 0
buffers_alloc         | 0
stats_reset           | 2021-10-29 14:41:09.22286+00
```
Запомним lsn
```console
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/EB462548
(1 row)
```
Запускаем нагрузку
```console
-bash-4.2$ /usr/pgsql-14/bin/pgbench -c8 -P 60 -T 600 -U postgres postgres
pgbench (14.0)
starting vacuum...end.
progress: 60.0 s, 761.1 tps, lat 10.495 ms stddev 5.820
progress: 120.0 s, 589.1 tps, lat 13.548 ms stddev 17.661
progress: 180.0 s, 460.1 tps, lat 17.369 ms stddev 25.337
progress: 240.0 s, 474.0 tps, lat 16.858 ms stddev 25.031
progress: 300.0 s, 472.9 tps, lat 16.904 ms stddev 25.597
progress: 360.0 s, 472.0 tps, lat 16.930 ms stddev 24.994
progress: 420.0 s, 471.1 tps, lat 16.962 ms stddev 25.481
progress: 480.0 s, 461.9 tps, lat 17.309 ms stddev 25.652
progress: 540.0 s, 489.5 tps, lat 16.320 ms stddev 24.793
progress: 600.0 s, 491.5 tps, lat 16.259 ms stddev 24.999
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 308623
latency average = 15.538 ms
latency stddev = 22.718 ms
initial connection time = 29.910 ms
tps = 514.354672 (without initial connection time)

```
>Измерьте, какой объем журнальных файлов был сгенерирован за это время. 

Вычисляем объем журнальных файлов. Объем 443Мб.
```console
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 1/6F46F48
(1 row)
postgres=# select pg_size_pretty('1/6F46F48'::pg_lsn - '0/EB462548'::pg_lsn);
 pg_size_pretty
----------------
 443 MB
(1 row)
```
>Оцените, какой объем приходится в среднем на одну контрольную точку.  

Смотрим контрольные точки.
```console
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+-----------------------------
checkpoints_timed     | 20
checkpoints_req       | 0
checkpoint_write_time | 539521
checkpoint_sync_time  | 149
buffers_checkpoint    | 39976
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 2021
buffers_backend_fsync | 0
buffers_alloc         | 2011
stats_reset           | 2021-10-29 14:41:09.22286+00
```

Контрольных точек было сделано 20. Объем в среднем на контрольную точку 443 Мб / 20 = 22.15 Мб
>Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло? 

Да, все точки выполнялись по расписанию, они все checkpoints_timed
```console
postgres=# show max_wal_size;
 max_wal_size
--------------
 16GB
(1 row)
```
Так как max_wal_size не достигался, все контрольные точки выполнялись по времени.
>Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.  
>Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?  
