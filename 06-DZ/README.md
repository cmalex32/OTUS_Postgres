# Настройка autovacuum с учетом оптимальной производительности  

>Цель:  
>запустить нагрузочный тест pgbench с профилем нагрузки DWH  
>настроить параметры autovacuum для достижения максимального уровня устойчивой производительности  
>создать GCE инстанс типа e2-medium и standard disk 10GB  
>установить на него PostgreSQL 13 с дефолтными настройками  

Создаем инстанс RHEL 7 и  устанавливаем на него PostgreSQL 14, запускаем, поключаемся
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

Среднее значение в последней 1/6 части работы около 610 tps
```console
-bash-4.2$ ./pgbench -c8 -P 60 -T 3600 -U postgres postgres
pgbench (14.0)
starting vacuum...end.
progress: 60.0 s, 948.8 tps, lat 8.420 ms stddev 4.406
progress: 120.0 s, 945.0 tps, lat 8.458 ms stddev 4.502
progress: 180.0 s, 965.8 tps, lat 8.275 ms stddev 4.414
progress: 240.0 s, 954.0 tps, lat 8.378 ms stddev 4.838
progress: 300.0 s, 983.1 tps, lat 8.130 ms stddev 4.239
progress: 360.0 s, 766.7 tps, lat 10.423 ms stddev 15.619
progress: 420.0 s, 616.4 tps, lat 12.958 ms stddev 21.293
progress: 480.0 s, 630.3 tps, lat 12.683 ms stddev 20.959
progress: 540.0 s, 627.0 tps, lat 12.750 ms stddev 21.172
progress: 600.0 s, 635.2 tps, lat 12.572 ms stddev 20.920
progress: 660.0 s, 621.6 tps, lat 12.855 ms stddev 21.218
progress: 720.0 s, 627.2 tps, lat 12.741 ms stddev 20.924
progress: 780.0 s, 621.5 tps, lat 12.855 ms stddev 21.304
progress: 840.0 s, 614.2 tps, lat 13.013 ms stddev 21.498
progress: 900.0 s, 508.5 tps, lat 15.714 ms stddev 45.837
progress: 960.0 s, 648.7 tps, lat 12.318 ms stddev 19.887
progress: 1020.0 s, 617.4 tps, lat 12.944 ms stddev 21.115
progress: 1080.0 s, 625.4 tps, lat 12.781 ms stddev 20.944
progress: 1140.0 s, 623.2 tps, lat 12.824 ms stddev 20.818
progress: 1200.0 s, 615.2 tps, lat 12.995 ms stddev 21.114
progress: 1260.0 s, 627.2 tps, lat 12.746 ms stddev 20.680
progress: 1320.0 s, 622.8 tps, lat 12.837 ms stddev 20.801
progress: 1380.0 s, 629.3 tps, lat 12.700 ms stddev 21.005
progress: 1440.0 s, 632.1 tps, lat 12.639 ms stddev 20.739
progress: 1500.0 s, 617.5 tps, lat 12.938 ms stddev 21.039
progress: 1560.0 s, 622.0 tps, lat 12.850 ms stddev 21.166
progress: 1620.0 s, 609.3 tps, lat 13.115 ms stddev 21.205
progress: 1680.0 s, 615.7 tps, lat 12.977 ms stddev 20.990
progress: 1740.0 s, 618.8 tps, lat 12.915 ms stddev 20.889
progress: 1800.0 s, 599.4 tps, lat 13.338 ms stddev 21.342
progress: 1860.0 s, 619.2 tps, lat 12.905 ms stddev 20.897
progress: 1920.0 s, 618.2 tps, lat 12.929 ms stddev 20.688
progress: 1980.0 s, 614.6 tps, lat 12.997 ms stddev 20.914
progress: 2040.0 s, 605.7 tps, lat 13.198 ms stddev 21.222
progress: 2100.0 s, 615.2 tps, lat 12.997 ms stddev 21.115
progress: 2160.0 s, 604.0 tps, lat 13.228 ms stddev 21.126
progress: 2220.0 s, 625.1 tps, lat 12.791 ms stddev 20.764
progress: 2280.0 s, 618.9 tps, lat 12.912 ms stddev 21.090
progress: 2340.0 s, 627.1 tps, lat 12.742 ms stddev 20.672
progress: 2400.0 s, 629.2 tps, lat 12.705 ms stddev 20.544
progress: 2460.0 s, 617.0 tps, lat 12.951 ms stddev 20.645
progress: 2520.0 s, 621.3 tps, lat 12.866 ms stddev 20.731
progress: 2580.0 s, 609.8 tps, lat 13.104 ms stddev 21.232
progress: 2640.0 s, 617.0 tps, lat 12.953 ms stddev 20.857
progress: 2700.0 s, 620.0 tps, lat 12.891 ms stddev 20.813
progress: 2760.0 s, 607.6 tps, lat 13.150 ms stddev 20.967
progress: 2820.0 s, 617.9 tps, lat 12.934 ms stddev 20.884
progress: 2880.0 s, 613.6 tps, lat 13.022 ms stddev 21.192
progress: 2940.0 s, 619.3 tps, lat 12.908 ms stddev 20.719
progress: 3000.0 s, 616.2 tps, lat 12.971 ms stddev 20.950
progress: 3060.0 s, 616.9 tps, lat 12.958 ms stddev 20.702
progress: 3120.0 s, 608.5 tps, lat 13.132 ms stddev 21.022
progress: 3180.0 s, 605.8 tps, lat 13.185 ms stddev 20.908
progress: 3240.0 s, 602.7 tps, lat 13.259 ms stddev 21.593
progress: 3300.0 s, 613.0 tps, lat 13.040 ms stddev 21.210
progress: 3360.0 s, 611.0 tps, lat 13.073 ms stddev 21.026
progress: 3420.0 s, 600.0 tps, lat 13.323 ms stddev 21.316
progress: 3480.0 s, 612.0 tps, lat 13.060 ms stddev 21.384
progress: 3540.0 s, 614.8 tps, lat 13.000 ms stddev 21.196
progress: 3600.0 s, 607.4 tps, lat 13.154 ms stddev 21.670
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 2329112
latency average = 12.352 ms
latency stddev = 20.234 ms
initial connection time = 22.962 ms
tps = 646.976461 (without initial connection time)
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
```console
-bash-4.2$ ./pgbench -c8 -P 60 -T 3600 -U postgres postgres
pgbench (14.0)
starting vacuum...end.
progress: 60.0 s, 909.9 tps, lat 8.779 ms stddev 4.999
progress: 120.0 s, 932.0 tps, lat 8.576 ms stddev 4.818
progress: 180.0 s, 939.9 tps, lat 8.504 ms stddev 4.697
progress: 240.0 s, 938.8 tps, lat 8.513 ms stddev 4.459
progress: 300.1 s, 946.1 tps, lat 8.435 ms stddev 4.415
progress: 360.1 s, 586.3 tps, lat 13.627 ms stddev 22.448
progress: 420.1 s, 517.5 tps, lat 15.445 ms stddev 26.141
progress: 480.1 s, 582.6 tps, lat 13.721 ms stddev 22.339
progress: 540.1 s, 586.4 tps, lat 13.631 ms stddev 22.330
progress: 600.1 s, 585.1 tps, lat 13.657 ms stddev 22.333
progress: 660.1 s, 588.4 tps, lat 13.582 ms stddev 22.645
progress: 720.1 s, 577.7 tps, lat 13.830 ms stddev 22.503
progress: 780.1 s, 589.8 tps, lat 13.553 ms stddev 22.379
progress: 840.1 s, 579.0 tps, lat 13.793 ms stddev 22.832
progress: 900.1 s, 588.0 tps, lat 13.587 ms stddev 22.698
progress: 960.1 s, 591.0 tps, lat 13.525 ms stddev 22.378
progress: 1020.1 s, 575.4 tps, lat 13.888 ms stddev 22.831
progress: 1080.1 s, 590.9 tps, lat 13.531 ms stddev 22.483
progress: 1140.1 s, 593.1 tps, lat 13.469 ms stddev 22.609
progress: 1200.1 s, 594.7 tps, lat 13.446 ms stddev 22.401
progress: 1260.1 s, 592.3 tps, lat 13.499 ms stddev 22.341
progress: 1320.1 s, 582.9 tps, lat 13.718 ms stddev 22.650
progress: 1380.1 s, 592.1 tps, lat 13.498 ms stddev 22.372
progress: 1440.1 s, 586.4 tps, lat 13.631 ms stddev 22.616
progress: 1500.1 s, 587.6 tps, lat 13.596 ms stddev 22.206
progress: 1560.1 s, 593.4 tps, lat 13.471 ms stddev 22.362
progress: 1620.1 s, 588.2 tps, lat 13.587 ms stddev 22.623
progress: 1680.1 s, 584.1 tps, lat 13.679 ms stddev 22.232
progress: 1740.1 s, 598.6 tps, lat 13.348 ms stddev 22.190
progress: 1800.1 s, 595.3 tps, lat 13.425 ms stddev 22.332
progress: 1860.1 s, 593.0 tps, lat 13.479 ms stddev 22.346
progress: 1920.1 s, 596.3 tps, lat 13.408 ms stddev 22.209
progress: 1980.1 s, 579.8 tps, lat 13.788 ms stddev 22.226
progress: 2040.1 s, 586.0 tps, lat 13.635 ms stddev 22.553
progress: 2100.1 s, 589.0 tps, lat 13.564 ms stddev 22.147
progress: 2160.1 s, 587.5 tps, lat 13.611 ms stddev 22.236
progress: 2220.1 s, 585.1 tps, lat 13.658 ms stddev 22.226
progress: 2280.1 s, 575.8 tps, lat 13.879 ms stddev 22.272
progress: 2340.1 s, 586.9 tps, lat 13.614 ms stddev 22.148
progress: 2400.1 s, 584.8 tps, lat 13.662 ms stddev 22.156
progress: 2460.1 s, 582.0 tps, lat 13.737 ms stddev 22.036
progress: 2520.1 s, 575.4 tps, lat 13.894 ms stddev 22.416
progress: 2580.1 s, 572.2 tps, lat 13.957 ms stddev 22.204
progress: 2640.1 s, 574.6 tps, lat 13.908 ms stddev 22.296
progress: 2700.1 s, 576.6 tps, lat 13.862 ms stddev 22.440
progress: 2760.1 s, 584.2 tps, lat 13.684 ms stddev 22.066
progress: 2820.1 s, 579.2 tps, lat 13.796 ms stddev 22.506
progress: 2880.1 s, 579.5 tps, lat 13.796 ms stddev 22.182
progress: 2940.1 s, 566.9 tps, lat 14.089 ms stddev 22.246
progress: 3000.1 s, 581.0 tps, lat 13.764 ms stddev 22.354
progress: 3060.1 s, 585.7 tps, lat 13.645 ms stddev 22.170
progress: 3120.1 s, 568.7 tps, lat 14.045 ms stddev 22.827
progress: 3180.1 s, 584.6 tps, lat 13.669 ms stddev 22.130
progress: 3240.1 s, 569.0 tps, lat 14.040 ms stddev 22.157
progress: 3300.1 s, 581.6 tps, lat 13.745 ms stddev 22.096
progress: 3360.1 s, 583.5 tps, lat 13.699 ms stddev 22.142
progress: 3420.1 s, 577.4 tps, lat 13.837 ms stddev 22.367
progress: 3480.1 s, 573.6 tps, lat 13.936 ms stddev 22.492
progress: 3540.1 s, 580.7 tps, lat 13.762 ms stddev 22.366
progress: 3600.1 s, 582.0 tps, lat 13.739 ms stddev 22.282
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 2203034
latency average = 13.060 ms
latency stddev = 21.091 ms
initial connection time = 27.645 ms
tps = 611.945127 (without initial connection time)
```
