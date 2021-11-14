# Нагрузочное тестирование и тюнинг PostgreSQL

>Цель:  
>сделать нагрузочное тестирование PostgreSQL  
>настроить параметры PostgreSQL для достижения максимальной производительности  
>сделать проект ---10  
>сделать инстанс Google Cloud Engine типа e2-medium с ОС Ubuntu 20.04  
>поставить на него PostgreSQL 13 из пакетов собираемых postgres.org  

Используем уже установленный instance Red Hat Enterprise Linux Server release 7.9 (Maipo) + PostgreSQL 14
Пересоздадим БД и запустим
```console
-bash-4.2$ rm -rf /var/lib/pgsql/14/data
-bash-4.2$ /usr/pgsql-14/bin/initdb --locale en_US.UTF-8
systemctl start postgresql-14.service
```
Создадим тестовую базу
```console
postgres=# create database testdb;
CREATE DATABASE
```
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
sysbench-tpcc не удалось нормально запустить, можно воспользоваться советом от Vladislav Fedotov использовать sysbench напрямую
делаем prepare
```console
[root@instance-2 ~]# sysbench --pgsql-db=testdb --db-driver=pgsql --pgsql-user=postgres --table_size=2000000 --tables=25 /usr/share/sysbench/oltp_read_write.lua prepare
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Creating table 'sbtest1'...
Inserting 2000000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 2000000 records into 'sbtest2'
Creating a secondary index on 'sbtest2'...
...
Creating table 'sbtest25'...
Inserting 2000000 records into 'sbtest25'
Creating a secondary index on 'sbtest25'...
```
 запускаем с базой где установлены параметры по умолчанию
 ```console
 [root@instance-2 ~]# sysbench --pgsql-db=testdb --db-driver=pgsql --pgsql-user=postgres --table_size=2000000 --tables=25 --report-interval=10 --threads=64 --time=600 --verbosity=4 /usr/share/sysbench/oltp_read_write.lua run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 64
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 64 tps: 265.99 qps: 5389.50 (r/w/o: 3784.94/1066.24/538.33) lat (ms,95%): 383.33 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 64 tps: 57.01 qps: 1143.58 (r/w/o: 801.70/227.85/114.03) lat (ms,95%): 2045.74 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 64 tps: 56.20 qps: 1123.53 (r/w/o: 784.92/226.01/112.60) lat (ms,95%): 1973.38 err/s: 0.10 reconn/s: 0.00
[ 40s ] thds: 64 tps: 62.70 qps: 1259.89 (r/w/o: 882.79/251.70/125.40) lat (ms,95%): 1869.60 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 64 tps: 66.60 qps: 1327.62 (r/w/o: 928.81/265.60/133.20) lat (ms,95%): 1903.57 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 64 tps: 73.70 qps: 1473.29 (r/w/o: 1032.49/293.40/147.40) lat (ms,95%): 1561.52 err/s: 0.00 reconn/s: 0.00
[ 70s ] thds: 64 tps: 73.30 qps: 1468.90 (r/w/o: 1027.80/294.50/146.60) lat (ms,95%): 1533.66 err/s: 0.00 reconn/s: 0.00
...
[ 580s ] thds: 64 tps: 73.40 qps: 1460.28 (r/w/o: 1019.59/293.90/146.80) lat (ms,95%): 1648.20 err/s: 0.00 reconn/s: 0.00
[ 590s ] thds: 64 tps: 65.20 qps: 1329.03 (r/w/o: 931.75/266.89/130.39) lat (ms,95%): 1708.63 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 64 tps: 69.50 qps: 1383.18 (r/w/o: 970.86/273.62/138.71) lat (ms,95%): 1803.47 err/s: 0.00 reconn/s: 0.00
Time limit exceeded, exiting...
(last message repeated 63 times)
Done.

SQL statistics:
    queries performed:
        read:                            655942
        write:                           187409
        other:                           93707
        total:                           937058
    transactions:                        46852  (78.01 per sec.)
    queries:                             937058 (1560.18 per sec.)
    ignored errors:                      1      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.6060s
    total number of events:              46852

Latency (ms):
         min:                                    4.30
         avg:                                  820.02
         max:                                 4005.85
         95th percentile:                     1589.90
         sum:                             38419355.01

Threads fairness:
    events (avg/stddev):           732.0625/10.57
    execution time (avg/stddev):   600.3024/0.21
 ```
>написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему  

Меняем параметры сервиса БД, для максимальной производительности
```console
# Делаем минимальное количество возможных подключений, запуск будет в 64 коннекта плюс системные
max_connections = '80'
# Выставляем 50 процентов от всего объема памяти для использования сервисом БД
shared_buffers = '2GB'
# Устанавливаем параметры для записи в файловую систему максимально небезопасными, но быстрыми
synchronous_commit = 'off'
full_page_writes = 'off'
fsync = 'off'
# Устанавливаем максимально возможный параметр использования памяти для подключений к базе исходя из количества сессий для теста (64)
work_mem = '30MB'
# Устанавливаем минимально возможный параметр использования памяти для служебных процессов
maintenance_work_mem = '1MB'
# Автовакуум полностью выключаем
autovacuum = 'off'
# Максимально уменьшаем количество записываемой информации в WAL файлы
wal_level = 'minimal'требуе
# Если этот параметр больше 0 не будет работать предыдущий, так как при передачи WAL файлов требуется минимум replica
max_wal_senders = '0'
# Так как у нас база горазда больше размеров оперативной памяти и диски при этом SSD, то можно этот параметр произвольного доступа к диску уменьшить, по умолчанию от # 4.0
random_page_cost = '1.1'
```
Запукаем тест
```console
[root@instance-2 ~]# sysbench --pgsql-db=testdb --db-driver=pgsql --pgsql-user=postgres --table_size=2000000 --tables=25 --report-interval=10 --threads=64 --time=600 --verbosity=4 /usr/share/sysbench/oltp_read_write.lua run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 64
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 64 tps: 272.73 qps: 5525.37 (r/w/o: 3879.54/1094.21/551.63) lat (ms,95%): 356.70 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 64 tps: 239.72 qps: 4782.01 (r/w/o: 3346.21/956.16/479.63) lat (ms,95%): 707.07 err/s: 0.00 reconn/s: 0.00
...
[ 590s ] thds: 64 tps: 152.00 qps: 3036.72 (r/w/o: 2126.12/606.60/304.00) lat (ms,95%): 909.80 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 64 tps: 127.80 qps: 2551.55 (r/w/o: 1785.27/510.69/255.60) lat (ms,95%): 1069.86 err/s: 0.00 reconn/s: 0.00
Time limit exceeded, exiting...
(last message repeated 63 times)
Done.

SQL statistics:
    queries performed:
        read:                            1141154
        write:                           326043
        other:                           163023
        total:                           1630220
    transactions:                        81511  (135.61 per sec.)
    queries:                             1630220 (2712.22 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          601.0641s
    total number of events:              81511

Latency (ms):
         min:                                    2.98
         avg:                                  471.35
         max:                                 5975.90
         95th percentile:                     1069.86
         sum:                             38419994.61

Threads fairness:
    events (avg/stddev):           1273.6094/20.42
    execution time (avg/stddev):   600.3124/0.18
```
В результате средний tps увеличился от 78.01 до 135.61. Тест был проделан для максимальной производительности, на продуктивных средах такие параметры неприемлимы.
Создается много рисков потерять данные, также на продуктиве обязательно еще включить data_checksums, что тоже не придаст скорости.


