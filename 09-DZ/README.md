# Нагрузочное тестирование и тюнинг PostgreSQL

>Цель:  
>сделать нагрузочное тестирование PostgreSQL  
>настроить параметры PostgreSQL для достижения максимальной производительности  
>сделать проект ---10  
>сделать инстанс Google Cloud Engine типа e2-medium с ОС Ubuntu 20.04  
>поставить на него PostgreSQL 13 из пакетов собираемых postgres.org  

Используем уже установленный instance Red Hat Enterprise Linux Server release 7.9 (Maipo) + PostgreSQL 14
Пересоздадим БД и запустим
-bash-4.2$ rm -rf /var/lib/pgsql/14/data
-bash-4.2$ /usr/pgsql-14/bin/initdb --locale en_US.UTF-8
-bash-4.2$ mv postgresql.conf postgresql.base.conf
-bash-4.2$ vi postgresql.conf
systemctl start postgresql-14.service
postgres=# create database testdb;
CREATE DATABASE
[root@instance-2 sysbench-tpcc]# ./tpcc.lua --pgsql-user=postgres --pgsql-db=testdb --time=300 --threads=64 --report-interval=1 --tables=10 --scale=100 --db-driver=pgsql prepare


[root@instance-2 sysbench-tpcc]# ./tpcc.lua --pgsql-user=postgres --pgsql-db=testdb --time=300 --threads=64 --report-interval=1 --tables=10 --scale=100 --db-driver=pgsql run

include 'postgresql.base.conf'

max_connections = 1000
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 524kB
min_wal_size = 4GB
max_wal_size = 16GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1

>настроить кластер PostgreSQL 13 на максимальную производительность не  
>обращая внимание на возможные проблемы с надежностью в случае  
>аварийной перезагрузки виртуальной машины  


>нагрузить кластер через утилиту  
>https://github.com/Percona-Lab/sysbench-tpcc (требует установки https://github.com/akopytov/sysbench)  

устанавливаем sysbench
```console
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench
```
>написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему  
