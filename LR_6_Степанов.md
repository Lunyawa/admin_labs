# Отчет по лабораторной работе №6
# Надежность: Блокировки и мониторинг

**Дата:** 2025-11-30  
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Степанов Дмитрий Александрович 

## Цель работы

Изучить систему блокировок в PostgreSQL и методы мониторинга активности сервера. Получить практические навыки анализа статистики, диагностики блокировок и взаимоблокировок, использования инструментов мониторинга.

## Теоретическая часть

**Блокировки:** Механизм, обеспечивающий согласованность данных при параллельном доступе. Бывают разных уровней: объекты, строки, буферы в памяти.

**Мониторинг:** Набор представлений и функций для отслеживания активности сервера, статистики использования объектов и блокировок.

**Взаимоблокировка (Deadlock):** Ситуация, когда две или более транзакции ожидают друг друга, освобождения ресурсов.

## Практическая часть

### Часть 1. Мониторинг активности

**Выполненные задачи:**   
1. В новой базе данных `lr06_db` создал таблицу `monitor_test (id INT)`. Вставил несколько строк, затем удалил все. Изучил статистику обращений к таблице в `pg_stat_all_tables` (`n_tup_ins`, `n_tup_del`, `n_live_tup`, `n_dead_tup`). Выполнил `VACUUM`. Вставки и удаления отражаются в `n_tup_ins` и `n_tup_del` соответственно. После `DELETE` растёт `n_dead_tup`. `VACUUM` уменьшает «мёртвые» строки (`n_dead_tup`) и увеличивает `vacuum_count`.

2. Создал ситуацию взаимоблокировки двух транзакций. Изучил, какая информация записывается в журнал сообщений сервера при обнаружении взаимоблокировки.

3. Установил и настроил расширение `pg_stat_statements`. Выполнил несколько произвольных запросов. Изучил информацию в представлении `pg_stat_statements` (топ запросов, время выполнения и т.д.).

**Команды:**

**Задача 1:**
```sql
CREATE DATABASE lab06_db;
\c lab06_db
SELECT pg_stat_reset();
CREATE TABLE monitor_test(id INT PRIMARY KEY);
INSERT INTO monitor_test VALUES (1),(2),(3),(4);
DELETE FROM monitor_test;
SELECT relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup
FROM pg_stat_all_tables
WHERE relname='monitor_test';
VACUUM monitor_test;
SELECT relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup, vacuum_count
FROM pg_stat_all_tables
WHERE relname='monitor_test';
```

**Задача 2:**
```sql
-- Сеанс 1
CREATE TABLE dead_demo(id INT PRIMARY KEY, val INT);
INSERT INTO dead_demo VALUES (1,10),(2,20);
BEGIN;
UPDATE dead_demo SET val=11 WHERE id=1;
```
```sql
-- Сеанс 2
BEGIN;
UPDATE dead_demo SET val=21 WHERE id=2;
```
```sql
-- Сеанс 1
UPDATE dead_demo SET val=12 WHERE id=2;
```
```sql
-- Сеанс 2
UPDATE dead_demo SET val=22 WHERE id=1; 
ROLLBACK;
```
```sql
-- Сеанс 1
ROLLBACK;
```
```bash
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```

**Задача 3:**
```bash
    sudo apt-get update
    sudo apt-get install postgresql-contrib
    sudo systemctl restart postgresql
```
```sql
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
SELECT pg_reload_conf();
CREATE EXTENSION pg_stat_statements;
SELECT count(*) FROM dead_demo;
SELECT * FROM dead_demo WHERE id=1;
SELECT * FROM dead_demo WHERE id=2;
INSERT INTO dead_demo VALUES (9,32),(10,45);
SELECT query, calls, total_plan_time, total_exec_time, 
mean_plan_time, mean_exec_time, rows
FROM pg_stat_statements;
```

**Фрагменты вывода:** 

**Задача 1:**
```text
 pg_stat_reset 
---------------
 
(1 row)

CREATE TABLE

INSERT 0 4

DELETE 4

   relname    | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
--------------+-----------+-----------+------------+------------
 monitor_test |         4 |         4 |          0 |          4
(1 row)

VACUUM

   relname    | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup | vacuum_count 
--------------+-----------+-----------+------------+------------+--------------
 monitor_test |         4 |         4 |          0 |          0 |            1
(1 row)
```

**Задача 2:**
```text
-- Сеанс 1
CREATE TABLE
INSERT 0 2
BEGIN
UPDATE 1
```
```text
-- Сеанс 2
BEGIN
UPDATE 1
```
```text
-- Сеанс 1
UPDATE 1
```
```text
-- Сеанс 2
ERROR:  deadlock detected
DETAIL:  Process 97289 waits for ShareLock on transaction 366333; blocked by process 94625.
Process 94625 waits for ShareLock on transaction 366334; blocked by process 97289.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "dead_demo"

ROLLBACK
```
```text
-- Сеанс 1
ROLLBACK
```
```text
2025-10-09 13:20:42.866 MSK [97289] student@lab06_db ERROR:  deadlock detected
2025-10-09 13:20:42.866 MSK [97289] student@lab06_db DETAIL:  Process 97289 waits for ShareLock on transaction 366333; blocked by process 94625.
	Process 94625 waits for ShareLock on transaction 366334; blocked by process 97289.
	Process 97289: UPDATE dead_demo SET val=22 WHERE id=1;
	Process 94625: UPDATE dead_demo SET val=12 WHERE id=2;
```

**Задача 3:**
```text
ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)

CREATE EXTENSION

 count 
-------
     8
(1 row)

 id | val 
----+-----
  1 |  10
(1 row)

 id | val 
----+-----
  2 |  20
(1 row)

INSERT 0 2
```
```text
 INSERT INTO dead_demo VALUES ($1,$2),($3,$4)                     |     1 |               0 |            0.042891 |              0 |            0.042891 |    2
 SELECT calls, total_plan_time, total_exec_time,                 +|     1 |               0 | 0.10167699999999999 |              0 | 0.10167699999999999 |    9
 mean_plan_time, mean_exec_time, rows, query                     +|       |                 |                     |                |                     | 
 FROM pg_stat_statements                                          |       |                 |                     |                |                     | 
 SELECT * FROM dead_demo WHERE id=$1                              |     2 |               0 |            0.031036 |              0 |            0.015518 |    2
 SELECT query, calls                                             +|     1 |               0 | 0.08881399999999999 |              0 | 0.08881399999999999 |    7
         FROM pg_stat_statements                                  |       |                 |                     |                |                     | 
 SELECT pg_reload_conf()                                          |     1 |               0 |            0.060778 |              0 |            0.060778 |    1
 SHOW shared_preload_libraries                                    |     1 |               0 |            0.004164 |              0 |            0.004164 |    0
 SELECT * FROM pg_stat_statements                                 |     3 |               0 |            0.437348 |              0 | 0.14578266666666667 |   21
 SELECT count(*) FROM dead_demo                                   |     1 |               0 |            0.064402 |              0 |            0.064402 |    1
 SELECT * FROM pg_stat_statements LIMIT $1                        |     1 |               0 |            0.125855 |              0 |            0.125855 |    8
 ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements' |     1 |               0 |            7.280041 |              0 |            7.280041 |    0
(10 rows)
```

---

### Часть 2. Блокировки объектов
**Выполненные задачи:**   
1.	На уровне изоляции `Read Committed` прочитал одну строку таблицы по первичному ключу. Изучил удерживаемые блокировки в `pg_locks`. `relation | AccessShareLock | lock_obj` — обычная блокировка таблицы при SELECT, чтобы защитить от DDL (`DROP/TRUNCATE/REWRITE`). `relation | AccessShareLock | lock_obj_pkey` — та же лёгкая блокировка на используемом индексе (PK). `virtualxid | ExclusiveLock` — служебная «своя виртуальная транзакция», есть у каждого сеанса. Остальные `AccessShareLock` на `pg_stat_activity`, `pg_locks`, `pg_authid_*`, `pg_database_*` — это «шум», который появляется из-за просмотра блокировки из того же сеанса: сам запрос к `pg_locks/pg_stat_activity` читает системные каталоги и берёт на них `AccessShareLock`. Пустые `wait_event_type/wait_event` означают, что ожиданий нет.

2.	Воспроизвёл ситуацию автоматического повышения уровня предикатных блокировок при чтении строк по индексу. Показал, что это может привести к ложной ошибке сериализации.

3.	Настроил запись в журнал сообщений о ожиданиях блокировок > 100 мс (`log_lock_waits = on`, `deadlock_timeout = 100ms`). Создал ситуацию длительного ожидания блокировки. Убедися, что сообщение появилось в логе.

**Команды:**

**Задача 1:**
```sql
CREATE TABLE lock_obj(id INT PRIMARY KEY, v INT);
INSERT INTO lock_obj SELECT g, g FROM generate_series(1,10) g;
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT * FROM lock_obj WHERE id=1;
SELECT l.pid, l.locktype, l.mode, l.granted, l.relation::regclass AS rel, a.wait_event_type, a.wait_event, a.state
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE a.pid = pg_backend_pid();
COMMIT;
```

**Задача 2:**
```sql
-- Сеанс 1
CREATE TABLE pred_demo(id INT PRIMARY KEY, v INT);
INSERT INTO pred_demo SELECT g, g FROM generate_series(1,500) g;
CREATE INDEX ON pred_demo(v);
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM pred_demo WHERE v BETWEEN 100 AND 400;
INSERT INTO pred_demo VALUES (2000, 200);
```
```sql
-- Сеанс 2
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM pred_demo WHERE v BETWEEN 100 AND 400;
INSERT INTO pred_demo VALUES (2001, 200);
COMMIT;
```
```sql
-- Сеанс 1
COMMIT;
```

**Задача 3:**
```sql
-- Сеанс 1
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '100ms';
SELECT pg_reload_conf();
BEGIN;
UPDATE lock_obj SET v = v+1 WHERE id=1;
```
```sql
-- Сеанс 2
BEGIN;
UPDATE lock_obj SET v = v+1 WHERE id=1;
```
```sql
ROLLBACK;
```
```bash
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```


**Фрагменты вывода:** 

**Задача 1:**
```text
CREATE TABLE

INSERT 0 10

BEGIN

 id | v 
----+---
  1 | 1
(1 row)


  pid   |  locktype  |      mode       | granted |            rel            | wait_event_type | wait_event | state  
--------+------------+-----------------+---------+---------------------------+-----------------+------------+--------
 104958 | relation   | AccessShareLock | t       | pg_stat_activity          |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_locks                  |                 |            | active
 104958 | relation   | AccessShareLock | t       | lock_obj_pkey             |                 |            | active
 104958 | relation   | AccessShareLock | t       | lock_obj                  |                 |            | active
 104958 | virtualxid | ExclusiveLock   | t       |                           |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_authid_oid_index       |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_database_oid_index     |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_authid_rolname_index   |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_database_datname_index |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_database               |                 |            | active
 104958 | relation   | AccessShareLock | t       | pg_authid                 |                 |            | active
(11 rows)

COMMIT
```

**Задача 2:**
```text
-- Сеанс 1
CREATE TABLE

INSERT 0 500

CREATE INDEX

BEGIN

SET

 count 
-------
   301
(1 row)

INSERT 0 1
```
```text
-- Сеанс 2
BEGIN

SET

 count 
-------
   301
(1 row)

INSERT 0 1

COMMIT
```
```text
-- Сеанс 1
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

**Задача 3:**
```text
-- Сеанс 1
ALTER SYSTEM
ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)

BEGIN
UPDATE 1
```
```text
-- Сеанс 2
BEGIN
```
```text
-- Сеанс 1
ROLLBACK
```
```text
2025-10-09 17:05:59.839 MSK [141471] student@lab06_db LOG:  process 141471 acquired ShareLock on transaction 366374 after 310068.379 ms
2025-10-09 17:05:59.839 MSK [141471] student@lab06_db CONTEXT:  while updating tuple (0,1) in relation "lock_obj"
2025-10-09 17:05:59.839 MSK [141471] student@lab06_db STATEMENT:  UPDATE lock_obj SET v = v+1 WHERE id=1;
```
---



## Результаты выполнения

**1. Мониторинг активности и статистики**
В результате экспериментов с таблицей `monitor_test` подтверждено, что системные представления `pg_stat_all_tables` и `pg_stat_statements` позволяют отслеживать как активность DML-операций, так и историю выполнения запросов. После вставки и удаления строк корректно увеличиваются счётчики `n_tup_ins` и `n_tup_del`, а выполнение `VACUUM` обнуляет `n_dead_tup`, демонстрируя актуализацию статистики.  
Подключение расширения `pg_stat_statements` дало возможность собирать статистику по времени планирования и выполнения запросов, количеству вызовов и возвращаемых строк, что обеспечивает инструментальный контроль за производительностью.

**2. Анализ взаимоблокировок**
Была воспроизведена ситуация классического взаимного ожидания между двумя транзакциями (`UPDATE` двух строк в разном порядке). В журнале сообщений сервера зафиксированы строки:
Проведён анализ: причина — перекрёстное ожидание блокировок строк в одной таблице. PostgreSQL корректно обнаруживает цикл зависимостей и прерывает одну из транзакций, предотвращая зависание системы.  
Дополнительно были выполнены опыты с тремя транзакциями, демонстрирующие построение «треугольного» графа ожиданий (A → B → C → A). Логи показали, что сервер детально регистрирует идентификаторы транзакций и соответствующие `pid`, что позволяет по журналу точно реконструировать структуру deadlock-графа.

## Выводы
1. Изучил систему блокировок в PostgreSQL и методы мониторинга активности сервера. 
2. Получил практические навыки анализа статистики, диагностики блокировок и взаимоблокировок, использования инструментов мониторинга.
