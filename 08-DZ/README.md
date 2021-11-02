# Механизм блокировок

>Цель:  
>понимать как работает механизм блокировок объектов и строк  

>Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. 

Заходим на сервер, меняем параметры СУБД
```console
postgres=# alter system set log_lock_waits = on;
ALTER SYSTEM
postgres=# alter system set deadlock_timeout = 200;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)
```
>Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Создаем таблицу, и пытаемся изменить одну и ту же строку
```console
postgres=# create table t1 (id int, text text);
CREATE TABLE
postgres=# insert into t1 values (1,'text1'), (2,'text2'), (3,'text3'), (4,'text4'), (5,'text5'), (6,'text6');
INSERT 0 6
```
В первой сессии
```console
postgres=# begin;
BEGIN
postgres=*# update t1 set text = 'text40' where id = 4;
UPDATE 1
```
Во второй
```console
postgres=# begin;
BEGIN
postgres=*# update t1 set text = 'text4' where id = 4;
```
Вторая ожидает в очереди для обновления строки  
В первой
```console
postgres=*# commit;
COMMIT
```
Во второй тоже происходит изменение строки и фиксация
```console
UPDATE 1
postgres=*# commit;
COMMIT
```
Смотрим лог файл
```console
2021-11-02 11:46:09.354 UTC [1964] LOG:  process 1964 still waiting for ShareLock on transaction 5345542 after 200.190 ms
2021-11-02 11:46:09.354 UTC [1964] DETAIL:  Process holding the lock: 1933. Wait queue: 1964.
2021-11-02 11:46:09.354 UTC [1964] CONTEXT:  while updating tuple (0,4) in relation "t1"
2021-11-02 11:46:09.354 UTC [1964] STATEMENT:  update t1 set text = 'text4' where id = 4;
2021-11-02 11:47:23.221 UTC [1964] LOG:  process 1964 acquired ShareLock on transaction 5345542 after 74066.341 ms
2021-11-02 11:47:23.221 UTC [1964] CONTEXT:  while updating tuple (0,4) in relation "t1"
2021-11-02 11:47:23.221 UTC [1964] STATEMENT:  update t1 set text = 'text4' where id = 4;
```
Как и ожидалось в лог файле отображено ожидание блокировки более 200 мс
>Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 

Создаем представление для таблицы t1, для удобства просмотра блокировок по ней
```console
CREATE VIEW t1_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 't1'::regclass);
CREATE VIEW
```
Обновляем строку в первой сессии, предварительно запоминаем ее параметры, номер транзакции и пид процесса
```console
 postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      5345545 |           2313
(1 row)

postgres=*#
postgres=*# update t1 set text = 'text2' where id = 2;
UPDATE 1
```
Обновляем строку во второй сессии, предварительно запоминаем ее параметры, номер транзакции и пид процесса
 ```console
 postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      5345546 |           1933
(1 row)

postgres=*#  update t1 set text = 'text2' where id = 2;
```
Обновляем строку в третьей сессии, предварительно запоминаем ее параметры, номер транзакции и пид процесса
```console
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      5345547 |           1964
(1 row)

postgres=*#  update t1 set text = 'text2' where id = 2;
```
Естественно вторая и третья сессии "подвисают" так как первая заблокировала строку.
>Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.   

Смотрим блокировки в первой сессии, используя ранее созданное представление и pid каждой сессии
```console
postgres=*# select * from t1_v where pid='2313';
 pid  |   locktype    | lockid  |       mode       | granted
------+---------------+---------+------------------+---------
 2313 | relation      | t1      | RowExclusiveLock | t
 2313 | transactionid | 5345545 | ExclusiveLock    | t
(2 rows)

postgres=*# select * from t1_v where pid='1933';
 pid  |   locktype    | lockid  |       mode       | granted
------+---------------+---------+------------------+---------
 1933 | relation      | t1      | RowExclusiveLock | t
 1933 | tuple         | t1:2    | ExclusiveLock    | t
 1933 | transactionid | 5345545 | ShareLock        | f
 1933 | transactionid | 5345546 | ExclusiveLock    | t
(4 rows)

postgres=*# select * from t1_v where pid='1964';
 pid  |   locktype    | lockid  |       mode       | granted
------+---------------+---------+------------------+---------
 1964 | relation      | t1      | RowExclusiveLock | t
 1964 | tuple         | t1:2    | ExclusiveLock    | f
 1964 | transactionid | 5345547 | ExclusiveLock    | t
(3 rows)
```
>Пришлите список блокировок и объясните, что значит каждая.  

Для первой транзакции блокируется строка таблицы t1 в режиме RowExclusiveLock, остальные поэтому ждут эту строку
вторая блокировка transactionid, блокируется своя транзакция в режиме ExclusiveLock

Для второй транзакции также блокировка transactionid блокирует свою транзакцию в режиме ExclusiveLock
хочет также заблокировать строку таблицы t1 в режиме RowExclusiveLock, но ожидает от первой транзакции  блокировку transactionid в режиме ShareLock
блокировка tuple в режиме ExclusiveLock, говорит о том что эта транзакция будет первой в очереди.

Третья транзакция, также блокировка transactionid блокирует свою транзакцию в режиме ExclusiveLock
хочет заблокировать таблицу t1 в режиме RowExclusiveLock, но ждет блокировку tuple в режиме ExclusiveLock 

>Воспроизведите взаимоблокировку трех транзакций.   

Запускаем три консоли и выполняем последовательно обновления строк.  
Первая консоль 
```console
postgres=# begin;
BEGIN
postgres=*# update t1 set text = 'text10' where id = 1;
UPDATE 1
````
Вторая
```console
postgres=# begin;
BEGIN
postgres=*# update t1 set text = 'text20' where id = 2;
UPDATE 1
```
Третья
```console
postgres=# begin;
BEGIN
postgres=*# update t1 set text = 'text30' where id = 3;
UPDATE 1
```
Первая консоль 
```console
postgres=*# update t1 set text = 'text2' where id = 2;
```
Вторая
```console
postgres=*# update t1 set text = 'text3' where id = 3;
```
Третья
```console
postgres=*# update t1 set text = 'text1' where id = 1;
ERROR:  deadlock detected
DETAIL:  Process 1964 waits for ShareLock on transaction 5345548; blocked by process 2313.
Process 2313 waits for ShareLock on transaction 5345549; blocked by process 1933.
Process 1933 waits for ShareLock on transaction 5345550; blocked by process 1964.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "t1"
```
Первые две консоли подвисают, ожидая доступа к редактируемой строке, третья сессия выдает ошибку так как были обнаружены взаимные блокировки.
Далее первая и вторая сессии ожидаемо отрабатывают штатно.
>Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?  

Смотрим журнал событий
```console
2021-11-02 12:21:04.440 UTC [2313] LOG:  process 2313 still waiting for ShareLock on transaction 5345549 after 200.232 ms
2021-11-02 12:21:04.440 UTC [2313] DETAIL:  Process holding the lock: 1933. Wait queue: 2313.
2021-11-02 12:21:04.440 UTC [2313] CONTEXT:  while updating tuple (0,11) in relation "t1"
2021-11-02 12:21:04.440 UTC [2313] STATEMENT:  update t1 set text = 'text2' where id = 2;
2021-11-02 12:21:50.302 UTC [1933] LOG:  process 1933 still waiting for ShareLock on transaction 5345550 after 200.126 ms
2021-11-02 12:21:50.302 UTC [1933] DETAIL:  Process holding the lock: 1964. Wait queue: 1933.
2021-11-02 12:21:50.302 UTC [1933] CONTEXT:  while updating tuple (0,3) in relation "t1"
2021-11-02 12:21:50.302 UTC [1933] STATEMENT:  update t1 set text = 'text3' where id = 3;
2021-11-02 12:23:03.627 UTC [1964] LOG:  process 1964 detected deadlock while waiting for ShareLock on transaction 5345548 after 200.234 ms
2021-11-02 12:23:03.627 UTC [1964] DETAIL:  Process holding the lock: 2313. Wait queue: .
2021-11-02 12:23:03.627 UTC [1964] CONTEXT:  while updating tuple (0,1) in relation "t1"
2021-11-02 12:23:03.627 UTC [1964] STATEMENT:  update t1 set text = 'text1' where id = 1;
2021-11-02 12:23:03.627 UTC [1964] ERROR:  deadlock detected
2021-11-02 12:23:03.627 UTC [1964] DETAIL:  Process 1964 waits for ShareLock on transaction 5345548; blocked by process 2313.
        Process 2313 waits for ShareLock on transaction 5345549; blocked by process 1933.
        Process 1933 waits for ShareLock on transaction 5345550; blocked by process 1964.
        Process 1964: update t1 set text = 'text1' where id = 1;
        Process 2313: update t1 set text = 'text2' where id = 2;
        Process 1933: update t1 set text = 'text3' where id = 3;
2021-11-02 12:23:03.627 UTC [1964] HINT:  See server log for query details.
2021-11-02 12:23:03.627 UTC [1964] CONTEXT:  while updating tuple (0,1) in relation "t1"
2021-11-02 12:23:03.627 UTC [1964] STATEMENT:  update t1 set text = 'text1' where id = 1;
2021-11-02 12:23:03.627 UTC [1933] LOG:  process 1933 acquired ShareLock on transaction 5345550 after 73525.222 ms
2021-11-02 12:23:03.627 UTC [1933] CONTEXT:  while updating tuple (0,3) in relation "t1"
2021-11-02 12:23:03.627 UTC [1933] STATEMENT:  update t1 set text = 'text3' where id = 3;
2021-11-02 12:23:30.004 UTC [1751] LOG:  checkpoint starting: time
2021-11-02 12:23:30.117 UTC [1751] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.103 s, sync=0.003 s, total=0.113 s; sync files=2, longest=0.003 s, average=0.002 s; distance=1 kB, estimate=58 kB
2021-11-02 12:23:51.649 UTC [2313] LOG:  process 2313 acquired ShareLock on transaction 5345549 after 167409.440 ms
2021-11-02 12:23:51.649 UTC [2313] CONTEXT:  while updating tuple (0,11) in relation "t1"
2021-11-02 12:23:51.649 UTC [2313] STATEMENT:  update t1 set text = 'text2' where id = 2;
```
Из журнала событий также видно что процесс 1964, это наша третья консоль, зафиксировал deadlock и после времени большим чем deadlock_timeout, а он у нас 200мс, транзакция завершилась неудачно.
>Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?  

Могут, если будут обновлять набор данных с разной сортировкой строк.
>Попробуйте воспроизвести такую ситуацию. 

Посмотрим как сейчас у нас выглядит таблица.
```console
postgres=#  select * from t1;
 id |  text
----+--------
  5 | text5
  6 | text6
  4 | text4
  1 | text10
  3 | text3
  2 | text2
(6 rows)
```
Строки перемешаны, так как мы сделали несколько обновлений строк.  
Создадим курсор в первой сессии
```console
postgres=# begin;
BEGIN
postgres=*# declare cur1 cursor for select * from t1  for update;
DECLARE CURSOR
```
И курсор с сортировкой во второй
```console
postgres=# begin;
BEGIN
postgres=*# declare cur2 cursor for select * from t1 order by id for update;
DECLARE CURSOR
```
Будем поочередно выбирать строки из обоих сессий.
```console
postgres=*# fetch cur1;
 id | text
----+-------
  5 | text5
(1 row)

postgres=*# fetch cur2;
 id |  text
----+--------
  1 | text10
(1 row)

postgres=*# fetch cur1;
 id | text
----+-------
  6 | text6
(1 row)

postgres=*# fetch cur2;
 id | text
----+-------
  2 | text2
(1 row)

postgres=*# fetch cur1;
 id | text
----+-------
  4 | text4
(1 row)

postgres=*# fetch cur2;
 id | text
----+-------
  3 | text3
(1 row)

postgres=*# fetch cur1;
```
Первая сессия "подвисает" так как эта строка уже выбрана во второй
и на следующем шаге вторая вылетает с ошибкой
```console
postgres=*# fetch cur2;
ERROR:  deadlock detected
DETAIL:  Process 1964 waits for ShareLock on transaction 5345551; blocked by process 1933.
Process 1933 waits for ShareLock on transaction 5345552; blocked by process 1964.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,8) in relation "t1"
```
Первая продолжает работать
```console
 id |  text
----+--------
  1 | text10
(1 row)

postgres=*# commit;
COMMIT
```
А вторая "откатывается"
```console
postgres=!# commit;
ROLLBACK
```
