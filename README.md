# Занятие №7: Блокировки

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.  
**Выполнение:** Поменял значения параметров - *log_lock_waits = on* и *deadlock_timeout = 200*. Ситуацию воспроизвел - в первой сессии начал транзакцию и выполнил *lock table l_test;*, а во второй выполнил *select * from l_test;*, после чего в логе появились следующие сообщения:  
*2021-10-31 08:27:25.578 UTC [10480] LOG:  process 10480 still waiting for AccessShareLock on relation 16384 of database 13434 after 200.217 ms at character 15  
2021-10-31 08:27:25.578 UTC [10480] DETAIL:  Process holding the lock: 10783. Wait queue: 10480.  
2021-10-31 08:27:25.578 UTC [10480] STATEMENT:  select * from l_test;*

2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.  
**Выполнение:** Список блокировок следующий:  
  
|  pid  |   locktype    |  lockid  |       mode       | granted |  
|-------|---------------|----------|------------------|---------|  
 13600 | relation      | l_test   | RowExclusiveLock | t  
 10783 | relation      | l_test   | RowExclusiveLock | t  
 14906 | relation      | l_test   | RowExclusiveLock | t  
 10783 | tuple         | l_test:3 | ExclusiveLock    | t  
 10783 | transactionid | 499      | ShareLock        | f  
 10783 | transactionid | 500      | ExclusiveLock    | t  
 13600 | tuple         | l_test:3 | ExclusiveLock    | f  
 13600 | transactionid | 501      | ExclusiveLock    | t  
 14906 | transactionid | 499      | ExclusiveLock    | t  

Рассмотрим их по порядку:  
***Первая сессия***  

|  pid  |   locktype    | lockid |       mode       | granted |
|-------|---------------|--------|------------------|---------|
 14906 | relation      | l_test | RowExclusiveLock | t
 14906 | transactionid | 499    | ExclusiveLock    | t

В данной сессии было создано две блокировки: в первой строке идет непосредственно блокировка строки (т.к. mode - RowExclusiveLock) таблицы l_test (т.к. lockid - l_test); во второй строке, насколько я понимаю, указывается, что транзакция 499 получает эксклюзивную блокировку по своему идентификатору  
  
***Вторая сессия***

|  pid  |   locktype    |  lockid  |       mode       | granted |
|-------|---------------|----------|------------------|---------|
 10783 | relation      | l_test   | RowExclusiveLock | t
 10783 | tuple         | l_test:3 | ExclusiveLock    | t
 10783 | transactionid | 499      | ShareLock        | f
 10783 | transactionid | 500      | ExclusiveLock    | t

К данной сессии имеет отношение 4 блокировки:
- в первой строке идет непосредственно блокировка строки (т.к. mode - RowExclusiveLock) таблицы l_test (т.к. lockid - l_test);
- во второй строке была получена эксклюзивная блокировка версии строки (т.к. locktype - tuple)
- в третьей строке указано, что транзакция ожидает получение блокировки ShareLock (эта блокировка защищает строки от одновременного изменения), которая её не выдается, т.к. с этим же lockid выдана эксклюзивная блокировка транзакции 499. Эта блокировка и не дает на текущий момент получить ShareLock транзакции 500.
- в четвертой строке, указывается, что транзакция 500 получила эксклюзивную блокировку по своему id.


***Третья сессия***
  pid  |   locktype    |  lockid  |       mode       | granted
|-------|---------------|----------|------------------|---------
 13600 | relation      | l_test   | RowExclusiveLock | t
 13600 | tuple         | l_test:3 | ExclusiveLock    | f
 13600 | transactionid | 501      | ExclusiveLock    | t

К данной сессии имеют отношение три блокировки:  
- в первой строке идет непосредственно блокировка строки (т.к. mode - RowExclusiveLock) таблицы l_test (т.к. lockid - l_test);
- во второй строке указано, что транзакция запросила эксклюзивную блокировку версии строки с id l_test:3, но еще ожидает еще получения, т.к. на текущий момент эта блокировка занята транзакцией 500.
- в третьей строке, указывается, что транзакция 501 получила эксклюзивную блокировку по своему id.
  
3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?  
**Выполнение:** Воспроизвел, по итогу в логе можно увидеть следующее
Сначала мы видим каждую из блокировок транзакций с указанием того, какой транзакцией (процессом) они блокируется, а также их порядок, что помогает понять, как позникла проблема
Блокировка первой транзакции:
```
2021-10-31 10:30:27.428 UTC [14906] LOG:  process 14906 still waiting for ShareLock on transaction 505 after 200.186 ms  
2021-10-31 10:30:27.428 UTC [14906] DETAIL:  Process holding the lock: 13600. Wait queue: 14906.  
2021-10-31 10:30:27.428 UTC [14906] CONTEXT:  while updating tuple (0,10) in relation "l_test"  
2021-10-31 10:30:27.428 UTC [14906] STATEMENT:  update l_test set c2 = 13 where c1 = 3;
```
Блокировка второй транзакции:
```  
2021-10-31 10:30:30.720 UTC [10783] LOG:  process 10783 still waiting for ShareLock on transaction 507 after 200.172 ms  
2021-10-31 10:30:30.720 UTC [10783] DETAIL:  Process holding the lock: 14906. Wait queue: 10783.  
2021-10-31 10:30:30.720 UTC [10783] CONTEXT:  while updating tuple (0,1) in relation "l_test"  
2021-10-31 10:30:30.720 UTC [10783] STATEMENT:  update l_test set c2 = 12 where c1 = 1;  
```
Блокировка третьей транзакции, где уже идет информация, что обнаружена взаимная блокировка:
```  
2021-10-31 10:30:34.217 UTC [13600] LOG:  process 13600 detected deadlock while waiting for ShareLock on transaction 504 after 200.222 ms  
2021-10-31 10:30:34.217 UTC [13600] DETAIL:  Process holding the lock: 10783. Wait queue: .  
2021-10-31 10:30:34.217 UTC [13600] CONTEXT:  while updating tuple (0,2) in relation "l_test"  
2021-10-31 10:30:34.217 UTC [13600] STATEMENT:  update l_test set c2 = 13 where c1 = 2;  
```
И далее сама информация о взаимной блокировке с указанием, какая транзакция какую заблокировала, а также используемые при этом запросы:
```  
2021-10-31 10:30:34.217 UTC [13600] ERROR:  deadlock detected  
2021-10-31 10:30:34.217 UTC [13600] DETAIL:  Process 13600 waits for ShareLock on transaction 504; blocked by process 10783.  
        Process 10783 waits for ShareLock on transaction 507; blocked by process 14906.  
        Process 14906 waits for ShareLock on transaction 505; blocked by process 13600.  
        Process 13600: update l_test set c2 = 13 where c1 = 2;  
        Process 10783: update l_test set c2 = 12 where c1 = 1;  
        Process 14906: update l_test set c2 = 13 where c1 = 3;  
```
На этом всё. По идее, более подробную информацию о произошедшем можно было бы получить, включив логирование самих запросов, тогда было бы дополнительно видно, какие запросы предшествовали возникновению блокировок.
  
4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?  
**Выполнение:** Возможно с помощью *select for update*
5. Попробуйте воспроизвести такую ситуацию.  
**Выполнение:**  

Транзакция 1:
```  
begin;  
select * from l_test limit 1 for update;  
```
Транзакция 2 (раз уж update без where, из select'ов тоже его уберем):
```  
begin;  
select * from l_test limit 2 for update skip locked;  
update l_test set c2=22;
```
Снова транзакция 1:
```
update l_test set c2=11;
```
По итогу в транзакции 1 выходит ошибка:
```
ERROR:  deadlock detected
DETAIL:  Process 14906 waits for ShareLock on transaction 516; blocked by process 10783.
Process 10783 waits for ShareLock on transaction 515; blocked by process 14906.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "l_test"
```
