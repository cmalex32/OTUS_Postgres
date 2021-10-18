# Установка и настройка PostgteSQL в контейнере Docker

>Цель:  
>установить PostgreSQL в Docker контейнере  
>настроить контейнер для внешнего подключения  
>сделать в GCE инстанс с Ubuntu 20.04  

Запускаем созданный ранее инстанс с RHEL7  
>поставить на нем Docker Engine  

Устанавливаем docker
```console
yum install docker -y
```
И запускаем
```console
[root@instance-2 ~]# systemctl start docker
[root@instance-2 ~]# systemctl enable docker
```

>сделать каталог /var/lib/postgres  

```console
mkdir -p /var/lib/postgres  
```
>развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres  

```console
docker network create pg-net
docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data:z postgres:14
```
>развернуть контейнер с клиентом postgres

```console
docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
Password for user postgres:
psql (14.0 (Debian 14.0-1.pgdg110+1))
Type "help" for help.

postgres=#
```
>подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк  

```console
create table test (id int);
CREATE TABLE
postgres=# insert into test values (1),(2),(3),(4),(5);
INSERT 0 5
```
>подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP  

Добавляем правило файрвола в гугл клоуд на порт 5432 от всех адресов 0.0.0.0/0
Пробуем подключиться
```console
-bash-4.2$ psql -h 34.68.248.207 -U postgres
Password for user postgres:
psql (12.4, server 14.0 (Debian 14.0-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

postgres=# select * from test;
 id
----
  1
  2
  3
  4
  5
(5 rows)
```
>удалить контейнер с сервером  

Останавливаем контейнер
```console
[root@instance-2 ~]# docker stop pg-docker
pg-docker
```
Удаляем
```console
[root@instance-2 ~]# docker rm pg-docker
pg-docker
```
>создать его заново  

Создаем
```console
docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data:z postgres:14
dffd2c7a2820055bdc4789611666ac2f28cd98da93a1fc3a11c5a40b42403a5f
```
>подключится снова из контейнера с клиентом к контейнеру с сервером  

```console
[root@instance-2 ~]# docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
Password for user postgres:
psql (14.0 (Debian 14.0-1.pgdg110+1))
Type "help" for help.
```
>проверить, что данные остались на месте  

Проверяем
```console
postgres=# select * from test;
 id
----
  1
  2
  3
  4
  5
(5 rows)
```
