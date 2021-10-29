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
Посмотрим статистику
```console
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 107
checkpoints_req       | 3
checkpoint_write_time | 6828208
checkpoint_sync_time  | 596
buffers_checkpoint    | 104737
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 30354
buffers_backend_fsync | 0
buffers_alloc         | 33796
stats_reset           | 2021-10-22 05:54:32.240908+00
```
Чтобы не вычислять статистику сбросим ее
```console
postgres=# select pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared
----------------------

(1 row)

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 0
checkpoints_req       | 0
checkpoint_write_time | 0
checkpoint_sync_time  | 0
buffers_checkpoint    | 0
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 0
buffers_backend_fsync | 0
buffers_alloc         | 8
stats_reset           | 2021-10-29 13:18:08.598558+00
```
Запомним lsn
```console
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/A4637BF0
(1 row)
```
Запускаем нагрузку
```console

```
>Измерьте, какой объем журнальных файлов был сгенерирован за это время. 
>Оцените, какой объем приходится в среднем на одну контрольную точку.  
>Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?  
>Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.  
>Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?  
