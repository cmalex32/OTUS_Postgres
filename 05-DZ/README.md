# Работа с базами данных, пользователями и правами

>Цель:  
>создание новой базы данных, схемы и таблицы  
>создание роли для чтения данных из созданной схемы созданной базы данных  
>создание роли для чтения и записи из созданной схемы созданной базы данных  
>1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)  

Запускаем существующий на RHEL7
>2 зайдите в созданный кластер под пользователем postgres  

```console
su - postgres
Last login: Tue Oct 19 10:40:53 MSK 2021 on pts/0
-bash-4.2$ psql
psql (14.0)
Type "help" for help.
postgres=#
```
>3 создайте новую базу данных testdb  

```console
postgres=# create database testdb;
CREATE DATABASE
```
>4 зайдите в созданную базу данных под пользователем postgres  

```console
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```
>5 создайте новую схему testnm  

```console
testdb=# create schema testnm;
CREATE SCHEMA
```
>6 создайте новую таблицу t1 с одной колонкой c1 типа integer  

```console
testdb=# create table t1 (c1 int);
CREATE TABLE
```
>7 вставьте строку со значением c1=1  

```console
testdb=# insert into t1 values (1);
INSERT 0 1
```
>8 создайте новую роль readonly  

```console
testdb=# create role testro with login;
```
CREATE ROLE
>9 дайте новой роли право на подключение к базе данных testdb  

```console
testdb=# grant connect on database testdb to testro;
GRANT
```
>10 дайте новой роли право на использование схемы testnm  

```console
testdb=# grant usage on schema testnm to testro;
GRANT
```
>11 дайте новой роли право на select для всех таблиц схемы testnm  

```console
testdb=# grant select on all tables in schema testnm to testro;
GRANT
```
>12 создайте пользователя testread с паролем test123  

```console
testdb=# create role testread with login password 'test123';
CREATE ROLE
```
>13 дайте роль readonly пользователю testread  

Роли readonly не существует, только что созданный пользователь и так не имеет никаких прав ни для чего.
>14 зайдите под пользователем testread в базу данных testdb  

```console
-bash-4.2$ psql -U testread  testdb
psql (14.0)
Type "help" for help.

testdb=>
```
>15 сделайте select * from t1;  

```console
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
>16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)  

нет
>17 напишите что именно произошло в тексте домашнего задания  

Конечно не получилоcь, у этого пользователя пока прав никаких нет для select этой таблицы
>18 у вас есть идеи почему? ведь права то дали?  

Права дали только для логина в базу
>19 посмотрите на список таблиц  

```console
testdb=> \dp
                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 public | t1   | table |                   |                   |
(1 row)
```
>20 подсказка в шпаргалке под пунктом 20  
>21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)  
>22 вернитесь в базу данных testdb под пользователем postgres  

```console
-bash-4.2$ psql
psql (14.0)
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
>23 удалите таблицу t1  

```console
testdb=# drop table t1;
DROP TABLE
```
>24 создайте ее заново но уже с явным указанием имени схемы testnm  

```console
testdb=# create table testnm.t1 (c1 int);
CREATE TABLE
```
>25 вставьте строку со значением c1=1  

```console
testdb=# insert into  testnm.t1 values (1);
INSERT 0 1
```
>26 зайдите под пользователем testread в базу данных testdb  

```console
-bash-4.2$ psql -U testread  testdb
psql (14.0)
Type "help" for help.

testdb=>
```
>27 сделайте select * from testnm.t1;  

```console
testdb=> select * from testnm.t1;
ERROR:  permission denied for schema testnm
LINE 1: select * from testnm.t1;
```
>28 получилось?  
>29 есть идеи почему? если нет - смотрите шпаргалку  

Конечно нет, у этой роли по прежнему нет никаких привелегий на select этой таблицы
>30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку  

Как и для первого пользователя нужно дать ему привелегии на select таблиц в этой схеме
```console
testdb=# grant usage on schema testnm to testread;
GRANT
testdb=# grant select on all tables in schema testnm to testread;
GRANT
```
>31 сделайте select * from testnm.t1;  

```console
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
>32 получилось?  

Да
>33 есть идеи почему? если нет - смотрите шпаргалку  
>31 сделайте select * from testnm.t1;  
>32 получилось?  
>33 ура!  
>34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2); 

```console
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```
>35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?  

Мы создали таблицу в схеме public, а в ней по умолчанию всем пользователям даются права create и usage, если попробовать сделать в нашей схеме
получим ожидаемый результат
```console
testdb=> create table testnm.t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema testnm
```
>36 есть идеи как убрать эти права? если нет - смотрите шпаргалку  

можно сделать так
```console
testdb=# revoke create on schema public from public;
REVOKE
```
теперь
```console
-bash-4.2$ psql -U testread  testdb
psql (14.0)
Type "help" for help.

testdb=> create table t5(c1 integer); insert into t5 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t5(c1 integer);
                     ^
ERROR:  relation "t5" does not exist
LINE 1: insert into t5 values (2);
                    ^
```

>37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды  
>38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);  
>39 расскажите что получилось и почему   
