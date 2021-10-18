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
postgres=# insert into test values (1),(2),(3),(4),(5);
INSERT 0 5
```
>подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP  
>удалить контейнер с сервером  
>создать его заново  
>подключится снова из контейнера с клиентом к контейнеру с сервером  
>проверить, что данные остались на месте  
