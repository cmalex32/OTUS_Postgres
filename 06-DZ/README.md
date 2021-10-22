# Настройка autovacuum с учетом оптимальной производительности  

>Цель:  
>запустить нагрузочный тест pgbench с профилем нагрузки DWH  
>настроить параметры autovacuum для достижения максимального уровня устойчивой производительности  
>создать GCE инстанс типа e2-medium и standard disk 10GB  
>установить на него PostgreSQL 13 с дефолтными настройками  

Используем установленный ранее RHEL 7 и PostgreSQL 14, запускаем, поключаемся
```console
 gcloud beta compute ssh --zone "us-central1-a" "instance-2"  --project "clever-muse-328410"
```
>применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла  

Применяем параметры и запускаем сервис БД
```console
-bash-4.2$ mv postgresql.conf postgresql.base.conf
-bash-4.2$ cat postgresql.conf
include 'postgresql.base.conf'

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

[root@instance-2 ~]# systemctl start postgresql-14.service
```
>выполнить pgbench -i postgres  

```console
-bash-4.2$ ./pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.42 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.26 s, vacuum 0.08 s, primary keys 0.07 s).
```
>запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres  
>дать отработать до конца  
>зафиксировать среднее значение tps в последней ⅙ части работы  

Среднее значение в последней 1/6 части работы около 530 tps
```console
-bash-4.2$ ./pgbench -c8 -P 60 -T 3600 -U postgres postgres
pgbench (14.0)
starting vacuum...end.
progress: 60.0 s, 854.5 tps, lat 9.347 ms stddev 5.379
progress: 120.0 s, 850.0 tps, lat 9.404 ms stddev 5.253
progress: 180.0 s, 866.2 tps, lat 9.227 ms stddev 5.243
progress: 240.0 s, 842.3 tps, lat 9.489 ms stddev 5.558
progress: 300.0 s, 595.2 tps, lat 13.421 ms stddev 21.701
progress: 360.0 s, 529.9 tps, lat 15.083 ms stddev 24.853
progress: 420.0 s, 528.1 tps, lat 15.128 ms stddev 24.864
progress: 480.0 s, 528.2 tps, lat 15.131 ms stddev 24.735
progress: 540.0 s, 524.7 tps, lat 15.233 ms stddev 25.036
progress: 600.0 s, 523.2 tps, lat 15.272 ms stddev 24.563
progress: 660.0 s, 523.3 tps, lat 15.276 ms stddev 24.876
progress: 720.0 s, 523.5 tps, lat 15.269 ms stddev 24.529
progress: 780.0 s, 526.8 tps, lat 15.174 ms stddev 25.124
progress: 840.0 s, 515.3 tps, lat 15.506 ms stddev 24.317
progress: 900.0 s, 530.7 tps, lat 15.055 ms stddev 24.918
progress: 960.0 s, 532.6 tps, lat 15.007 ms stddev 24.602
progress: 1020.0 s, 529.8 tps, lat 15.088 ms stddev 25.418
progress: 1080.0 s, 439.7 tps, lat 18.168 ms stddev 36.524
progress: 1140.0 s, 405.9 tps, lat 19.689 ms stddev 39.641
progress: 1200.0 s, 485.8 tps, lat 16.453 ms stddev 24.385
progress: 1260.0 s, 500.4 tps, lat 15.964 ms stddev 24.103
progress: 1320.0 s, 518.6 tps, lat 15.414 ms stddev 24.584
progress: 1380.0 s, 521.9 tps, lat 15.306 ms stddev 24.747
progress: 1440.0 s, 521.0 tps, lat 15.332 ms stddev 24.330
progress: 1500.0 s, 514.4 tps, lat 15.541 ms stddev 24.317
progress: 1560.0 s, 515.8 tps, lat 15.498 ms stddev 24.700
progress: 1620.0 s, 519.9 tps, lat 15.372 ms stddev 24.561
progress: 1680.0 s, 529.1 tps, lat 15.102 ms stddev 24.826
progress: 1740.0 s, 528.0 tps, lat 15.137 ms stddev 24.345
progress: 1800.0 s, 518.0 tps, lat 15.430 ms stddev 24.658
progress: 1860.0 s, 538.5 tps, lat 14.841 ms stddev 24.172
progress: 1920.0 s, 532.7 tps, lat 15.006 ms stddev 24.305
progress: 1980.0 s, 537.2 tps, lat 14.879 ms stddev 24.400
progress: 2040.0 s, 527.8 tps, lat 15.145 ms stddev 24.476
progress: 2100.0 s, 531.4 tps, lat 15.044 ms stddev 24.440
progress: 2160.0 s, 536.9 tps, lat 14.891 ms stddev 24.391
progress: 2220.0 s, 539.8 tps, lat 14.807 ms stddev 24.362
progress: 2280.0 s, 532.4 tps, lat 15.009 ms stddev 24.923
progress: 2340.0 s, 539.5 tps, lat 14.816 ms stddev 24.396
progress: 2400.0 s, 527.7 tps, lat 15.145 ms stddev 24.219
progress: 2460.0 s, 537.3 tps, lat 14.874 ms stddev 24.375
progress: 2520.0 s, 535.0 tps, lat 14.939 ms stddev 24.066
progress: 2580.0 s, 539.3 tps, lat 14.817 ms stddev 24.883
progress: 2640.0 s, 538.0 tps, lat 14.843 ms stddev 24.242
progress: 2700.0 s, 531.1 tps, lat 15.054 ms stddev 24.380
progress: 2760.0 s, 524.2 tps, lat 15.248 ms stddev 24.132
progress: 2820.0 s, 516.9 tps, lat 15.467 ms stddev 24.154
progress: 2880.0 s, 527.9 tps, lat 15.136 ms stddev 24.243
progress: 2940.0 s, 533.1 tps, lat 14.995 ms stddev 24.687
progress: 3000.0 s, 528.9 tps, lat 15.114 ms stddev 24.081
progress: 3060.0 s, 522.9 tps, lat 15.288 ms stddev 24.318
progress: 3120.0 s, 532.1 tps, lat 15.017 ms stddev 24.198
progress: 3180.0 s, 531.4 tps, lat 15.043 ms stddev 24.247
progress: 3240.0 s, 522.0 tps, lat 15.306 ms stddev 24.753
progress: 3300.0 s, 526.1 tps, lat 15.194 ms stddev 24.500
progress: 3360.0 s, 517.5 tps, lat 15.442 ms stddev 24.366
progress: 3420.0 s, 525.3 tps, lat 15.216 ms stddev 24.238
progress: 3480.0 s, 537.8 tps, lat 14.864 ms stddev 24.285
progress: 3540.0 s, 532.2 tps, lat 15.014 ms stddev 24.411
progress: 3600.0 s, 531.5 tps, lat 15.039 ms stddev 24.097
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1966643
latency average = 14.630 ms
latency stddev = 23.744 ms
initial connection time = 25.240 ms
tps = 546.289833 (without initial connection time)
```
>а дальше настроить autovacuum максимально эффективно  
>так чтобы получить максимально ровное значение tps на горизонте час

Меняем параметры
```console
autovacuum_max_workers = 6
autovacuum_naptime = 5s
autovacuum_vacuum_threshold = 25
autovacuum_vacuum_scale_factor = 0.1
autovacuum_vacuum_cost_delay = 10ms
autovacuum_vacuum_cost_limit = 1000
```
Перезапускаем сервис
```console
systemctl restart postgresql-14.service
```
Запускаем



