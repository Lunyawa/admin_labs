# Отчет по лабораторной работе №5
# Надежность: Журнал предзаписи (WAL)

**Дата:** 2025-11-27 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Степанов Дмитрий Александрович 

## Цель работы

Изучить работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Получить практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев.

## Теоретическая часть

**Буферный кеш:** Область общей памяти для кэширования страниц данных, считываемых с диска. Измененные ("грязные") буферы периодически сбрасываются на диск.

**Контрольная точка (Checkpoint):** Процесс принудительной записи всех "грязных" буферов на диск. Ограничивает объем WAL, необходимый для восстановления.

**Журнал предзаписи (WAL):** Циклический журнал, в который записываются все изменения данных перед тем, как они попадут в основные файлы данных. Обеспечивает надежность и возможность восстановления после сбоя.

**Восстановление:** Процесс применения WAL-записей, созданных после последней контрольной точки, к данным на диске для приведения их в согласованное состояние.

## Практическая часть

### Часть 1. Процессы и режимы остановки

**Выполненные задачи:**   
1. Средствами ОС нашёл процессы, отвечающие за работу буферного кеша (checkpointer, background writer) и журнала WAL (walwriter).

2. Остановил `PostgreSQL` в режиме `fast` (`sudo pg_ctlcluster 16 main stop`). Запустил сервер. Просмотрел журнал сообщений сервера (`/var/log/postgresql/postgresql-16-main.log`). Нашёл записи о контрольной точке, выполненной при завершении работы.

3. Остановил `PostgreSQL` в режиме immediate (`sudo pg_ctlcluster 16 main stop -m immediate`). Запустил сервер. Просмотрел журнал сообщений. Нашёл записи о восстановлении после сбоя (`recovery`). Появился `database system was not properly shut down; automatic recovery in progress`.
**Команды:**

**Задача 1:**
```bash
ps -ef | grep -E "post(gres|master)|checkpointer|writer|wal" | grep -v grep
```

**Задача 2:**
```bash
sudo pg_ctlcluster 16 main stop -m fast
sudo pg_ctlcluster 16 main start
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```

**Задача 3:**
```bash
sudo pg_ctlcluster 16 main stop -m immediate
sudo pg_ctlcluster 16 main start
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```

**Фрагменты вывода:** 

**Задача 1:**
```text
/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres     797     787  0 08:34 ?        00:00:00 postgres: 16/main: checkpointer 
postgres     798     787  0 08:34 ?        00:00:00 postgres: 16/main: background writer 
postgres     977     787  0 08:34 ?        00:00:00 postgres: 16/main: walwriter 
postgres     978     787  0 08:34 ?        00:00:02 postgres: 16/main: autovacuum launcher 
postgres     979     787  0 08:34 ?        00:00:00 postgres: 16/main: logical replication launcher 
student     6148    1601  0 08:46 pts/0    00:00:00 /usr/lib/postgresql/16/bin/psql
```

**Задача 2:**
```text
2025-10-09 08:53:20.243 MSK [797] LOG:  checkpoint starting: shutdown immediate
2025-10-09 08:53:20.250 MSK [797] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.009 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=22877 kB; lsn=0/64E8C0F0, redo lsn=0/64E8C0F0
```

**Задача 3:**
```text
2025-10-09 10:06:08.573 MSK [38667] LOG:  database system was not properly shut down; automatic recovery in progress
2025-10-09 10:06:08.786 MSK [38667] LOG:  redo starts at 0/5837B058
2025-10-09 10:06:08.984 MSK [38667] LOG:  invalid record length at 0/5837B058: expected at least 24, got 0
2025-10-09 10:06:09.345 MSK [38667] LOG:  redo done at 0/5837B058 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
2025-10-09 10:06:09.563 MSK [38666] LOG:  checkpoint starting: end-of-recovery immediate wait
2025-10-09 10:06:12.351 MSK [39813] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.003 s, sync=0.002 s, total=0.013 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB; lsn=0/5837B058, redo lsn=0/5837B058
```

---

### Часть 2. Буферный кеш и контрольные точки

**Выполненные задачи:**   
1.	Создал таблицу `wal_test (id INT, data TEXT)`. Вставил в неё достаточное количество строк. Определил, сколько страниц на диске занимает таблица. Определил, сколько буферов занимает таблица в кеше (запрос к `pg_buffercache`).

2.	Узнал общее количество "грязных" буферов в кеше Выполнил команду `CHECKPOINT;`. Снова проверил количество "грязных" буферов. Количество "грязны" буферов упало до нуля, так как они были записаны на диск.

3.	Подключил расширение `pg_prewarm`. Загрузил свою таблицу в кеш с помощью `pg_prewarm(...)`. Перезапустил сервер. Таблица не осталась в кеше, это недостаточно удобно.

**Команды:**

**Задача 1:**
```sql
CREATE DATABASE lab05_db;
\c lab05_db
CREATE TABLE wal_test(id int PRIMARY KEY, data text);
INSERT INTO wal_test
SELECT g, repeat('x', 200)
FROM generate_series(1, 100000) AS g;
SELECT pg_relation_size('wal_test');
SELECT current_setting('block_size')::int;
WITH rel AS (
  SELECT oid, pg_relation_filenode(oid) AS fn
  FROM pg_class WHERE relname = 'wal_test'
)
SELECT
  count(*) AS buffers
FROM pg_buffercache b
JOIN rel r
  ON b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
 AND b.relfilenode = r.fn
 AND b.relforknumber = 0;

```

**Задача 2:**
```sql
SELECT count(*) FROM pg_buffercache WHERE isdirty IS TRUE;
CHECKPOINT;
SELECT count(*) FROM pg_buffercache WHERE isdirty IS TRUE;
```

**Задача 3:**
```sql
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('wal_test');
```
```bash
sudo pg_ctlcluster 16 main restart
```
```sql
WITH rel AS (
  SELECT oid, pg_relation_filenode(oid) AS fn
  FROM pg_class WHERE relname = 'wal_test'
)
SELECT count(*) AS buffers
FROM pg_buffercache b
JOIN rel r
  ON b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
 AND b.relfilenode = r.fn
 AND b.relforknumber = 0;
```


**Фрагменты вывода:** 

**Задача 1:**
```text
CREATE TABLE
INSERT 0 100000
 pg_relation_size 
------------------
         24100864
(1 row)

 current_setting 
-----------------
            8192
(1 row)

 buffers 
---------
    2942
(1 row)

```

**Задача 2:**
```text
 count 
-------
  3291
(1 row)

CHECKPOINT

 count 
-------
     0
(1 row)

```

**Задача 3:**
```text
 pg_prewarm 
------------
       2942
(1 row)
```
```text
 buffers 
---------
       0
```


## Результаты выполнения

1. **Процессы и режимы завершения работы.**  
   В лабораторных экспериментах определены фоновые процессы PostgreSQL, ответственные за буферизацию и надёжность — `checkpointer`, `background writer` и `walwriter`.  
   При остановке в режиме `fast` в журнале фиксируется сообщение о штатной контрольной точке, после которой при перезапуске восстановление не требуется.  
   При остановке в режиме `immediate` сервер стартует с этапа `crash recovery`: в логах появляются строки `database system was not properly shut down; automatic recovery in progress`, `redo starts at ...`, `redo done at ...`. Это демонстрирует автоматическое восстановление базы до последней контрольной точки и применения всех WAL-записей после неё.

2. **Буферный кеш и контрольные точки.**  
   Используя расширения `pg_buffercache` и `pg_stat_bgwriter`, измерено количество «грязных» буферов до и после выполнения `CHECKPOINT;`. До контрольной точки их было более 3000, после — 0, что подтверждает успешный сброс данных на диск.  
   Эксперименты с расширением `pg_prewarm` показали, что таблица успешно загружается в буферный кеш (около 2900 буферов), но после перезапуска кеш полностью очищается, что подтверждает его оперативную, а не постоянную природу.

## Выводы

1. Изучил работу буферного кеша в PostgreSQL. 

2. Получил практические навыки управления контрольными точками
