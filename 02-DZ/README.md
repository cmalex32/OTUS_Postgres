Цель:
научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read
создать новый проект в Google Cloud Platform, например postgres2021-, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально на уровне GCP)
дать возможность доступа к этому проекту пользователю ifti@yandex.ru с ролью Project Editor
далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами

Выполнено

добавить свой ssh ключ в GCE metadata

Выполнено

зайти удаленным ssh (первая сессия), не забывайте про ssh-add

Выполнено

поставить PostgreSQL

Выполнено

зайти вторым ssh (вторая сессия)
запустить везде psql из под пользователя postgres
выключить auto commit

postgres@instance-1:~$ psql
psql (14.0 (Ubuntu 14.0-1.pgdg20.04+1))
Type "help" for help.
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF


сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;

postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT

посмотреть текущий уровень изоляции: show transaction isolation level

postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

postgres=# begin;
BEGIN

в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');

postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1

сделать select * from persons во второй сессии

postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

видите ли вы новую запись и если да то почему?

не видим эту запись так как она не зафиксирована в другой сессии, а грязное чтение не допускается

завершить первую транзакцию - commit;

postgres=*# commit;
COMMIT

сделать select * from persons во второй сессии

postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

видите ли вы новую запись и если да то почему?

видим эту запись, так как в этом режиме изоляции  read committed  при фиксации данных в другой сесии мы должны видеть зафиксированные данные

завершите транзакцию во второй сессии

postgres=*# commit;
COMMIT

начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;

postgres=# begin;
BEGIN
postgres=*# set transaction isolation level repeatable read;
SET
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read
(1 row)

в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');

postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1

сделать select * from persons во второй сессии

postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

видите ли вы новую запись и если да то почему?

не видим эту запись так как она не зафиксирована в другой сессии, а грязное чтение не допускается

завершить первую транзакцию - commit;

postgres=*# commit;
COMMIT

сделать select * from persons во второй сессии

postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

видите ли вы новую запись и если да то почему?

не видим эту запись, так как уровень Repeatable Read не допускает не только неповторяемое чтение, но и фантомное чтение

завершить вторую транзакцию

postgres=*# commit;
COMMIT

сделать select * from persons во второй сессии

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

видите ли вы новую запись и если да то почему?

новая запись видна, так как мы начали новую транзакцию уже после фиксации данных в первой транзакции и при любом уровне изоляции эти данные будут видны


