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
>запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres  
>дать отработать до конца  
>зафиксировать среднее значение tps в последней ⅙ части работы  
>а дальше настроить autovacuum максимально эффективно  
>так чтобы получить максимально ровное значение tps на горизонте час  
