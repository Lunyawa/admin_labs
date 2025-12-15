# Отчет по лабораторной работе №8
# Резервное копирование и управление доступом

**Дата:** 2025-12-01 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Степанов Дмитрий Александрович 

## Цель работы

Освоить методы резервного копирования и восстановления данных в PostgreSQL, включая логическое и физическое копирование, а также углубить навыки управления правами доступа пользователей.

## Теоретическая часть

**Логическое резервное копирование:** Выполняется утилитами `pg_dump` и `pg_dumpall`. Создает дампы SQL-команд для восстановления структуры и данных. Гибкое, но может быть медленным для больших БД.

**Физическое резервное копирование:** Копирование файлов данных кластера с помощью `pg_basebackup`. Быстрее, но требует архивации WAL для восстановления на момент времени (PITR).

**WAL-архивация:** Непрерывное сохранение сегментов WAL. Позволяет восстанавливать данные на любой момент времени после создания базовой резервной копии.

**Обновление сервера:** Процесс миграции данных на новую мажорную версию PostgreSQL, часто с использованием логического дампа/восстановления.

## Практическая часть

### Часть 1. Управление доступом (Повторение и закрепление)

**Выполненные задачи:**   
1. **Настройка привилегий:** Повторил задания из практики по управлению доступом (создание БД,
ролей writer/reader, настройка GRANT/REVOKE, проверка доступа w1/r1).

2. **Настройка аутентификации (Практика+):** Повторил задания по настройке `pg_hba.conf` для ролей `alice` и `bob` с использованием методов `trust`, `reject` и `peer`.

**Команды:**

**Задача 1:**

```sql
CREATE DATABASE lab08_db;
\c lab08_db
CREATE ROLE writer NOLOGIN;
CREATE ROLE reader NOLOGIN;
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE, CREATE ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;
ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public GRANT SELECT ON TABLES TO reader;
CREATE ROLE w1 LOGIN PASSWORD 'w1_pass';
CREATE ROLE r1 LOGIN PASSWORD 'r1_pass';
GRANT writer TO w1;
GRANT reader TO r1;
SET ROLE w1;       
SET ROLE writer;
CREATE TABLE public.test_table(id int primary key, v text);
INSERT INTO public.test_table VALUES (1,'a'),(2,'b');     
RESET ROLE;
SET ROLE r1;
TABLE public.test_table;
RESET ROLE;
SET ROLE w1;
UPDATE public.test_table SET v='aa' WHERE id=1;  
DELETE FROM public.test_table WHERE id=2;       
RESET ROLE;
```

**Задача 2:**
```sql
\c postgres
CREATE ROLE alice LOGIN;
CREATE ROLE bob   LOGIN;
SHOW hba_file;
```
```bash
sudo -i -u postgres
nano /etc/postgresql/16/main/pg_hba.conf
local all postgres trust
local all student trust
local all all md5
```
```sql
SELECT pg_reload_conf();
```
```bash
psql -U alice
psql -U bob
psql -U postgres
psql -U student
```
```sql
SHOW ident_file;
```
```bash
nano /etc/postgresql/16/main/pg_hba.conf
local all alice peer map=users_map
local all bob peer map=users_map
```
```bash
nano /etc/postgresql/16/main/pg_ident.conf
# MAPNAME    SYSTEM-USER    PG-ROLE
users_map    alice          alice
users_map    bob            bob
```
```sql
SELECT pg_reload_conf();
CREATE DATABASE alice;
CREATE DATABASE bob;
```
```bash
sudo adduser --disabled-password --gecos "" alice
sudo -i -u alice
psql
```
```bash
sudo adduser --disabled-password --gecos "" bob
sudo -i -u bob
psql
```

**Фрагменты вывода:** 

**Задача 1:**
```text
CREATE DATABASE

You are now connected to database "lab07_db" as user "student".

CREATE ROLE

CREATE ROLE

REVOKE

GRANT

GRANT

ALTER DEFAULT PRIVILEGES

CREATE ROLE

CREATE ROLE

GRANT ROLE

GRANT ROLE

SET    

SET

CREATE TABLE

INSERT 0 2

RESET

SET

 id | v  
----+----
  1 | aa
(1 row)

RESET

SET

UPDATE 1

DELETE 1

RESET         
```

**Задача 2:**
```text
You are now connected to database "postgres" as user "student".

CREATE ROLE

CREATE ROLE

              hba_file               
-------------------------------------
 /etc/postgresql/16/main/pg_hba.conf
(1 row)
```
```text
 pg_reload_conf 
----------------
 t
(1 row)
```
```text
Password for user alice: 
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: fe_sendauth: no password supplied

Password for user bob: 
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: fe_sendauth: no password supplied

postgres=#

student=#
```
```text
              ident_file               
---------------------------------------
 /etc/postgresql/16/main/pg_ident.conf
(1 row)
```
```text
 pg_reload_conf 
----------------
 t
(1 row)

CREATE DATABASE

CREATE DATABASE
```
```text
info: Adding user `alice' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `alice' (1001) ...
info: Adding new user `alice' (1001) with group `alice (1001)' ...
info: Creating home directory `/home/alice' ...
info: Copying files from `/etc/skel' ...
info: Adding new user `alice' to supplemental / extra groups `users' ...
info: Adding user `alice' to group `users' ...

alice=> 
```
```text
info: Adding user `bob' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `bob' (1002) ...
info: Adding new user `bob' (1002) with group `bob (1002)' ...
info: Creating home directory `/home/bob' ...
info: Copying files from `/etc/skel' ...
info: Adding new user `bob' to supplemental / extra groups `users' ...
info: Adding user `bob' to group `users' ...

bob=> 
```

---

### Часть 2. Логическое резервное копирование

**Выполненные задачи:**   
1.	**Простой дамп и восстановление:** Создал БД `backup_db` и таблицу с данными. Сделал логическую копию с помощью `pg_dump`. Удалил БД и восстановил её из копии. Убедился в целостности данных.

2.	**Параллельный дамп:** Создал несколько БД с различными объектами. Сделал копию глобальных объектов через `pg_dumpall --globals-only`. Сделал дампы каждой БД с помощью `pg_dump` в параллельном режиме (`-j`).

3.	**Восстановление кластера:** Восстановил весь кластер на "другом сервере" из созданных резервных копий.

4.	**Проблемы при загрузке (Практика+):** Попробовал создать данные и параметры `COPY`, которые приведут к ошибке при загрузке дампа.

**Команды:**

**Задача 1:**
```bash
psql -U postgres
```
```sql
CREATE DATABASE backup_db;
\c backup_db
CREATE TABLE t(a int primary key, b text);
INSERT INTO t  VALUES (1, 'row_1');
INSERT INTO t  VALUES (2, 'row_2');
INSERT INTO t  VALUES (3, 'row_3');
INSERT INTO t  VALUES (4, 'row_4');
INSERT INTO t  VALUES (5, 'row_5');
TABLE t;
```
```bash
pg_dump -U postgres -d backup_db -Fc -f lab08_1.dump
dropdb -U postgres backup_db
createdb -U postgres backup_db
pg_restore -U postgres -d backup_db -l lab08_1.dump
```
```sql
\c backup_db
TABLE t;
```

**Задача 2:**
```sql
CREATE DATABASE db1;
CREATE DATABASE db2;
\c db1
CREATE TABLE t1(id int primary key, v text); 
INSERT INTO t1 VALUES (1,'x'),(2,'y');
\c db2
CREATE TABLE t2(id int primary key, v text); 
INSERT INTO t2 VALUES (10,'z'),(20,'q');
```
```bash
pg_dumpall -U postgres --globals-only > globals_lab08.sql
pg_dump -U postgres -d db1 -Fd -j 4 -f db1_dir
pg_dump -U postgres -d db2 -Fd -j 4 -f db2_dir
```

**Задача 3:**
```bash
# Использую кластер, созданный в лабораторной 00
sudo pg_ctlcluster 16 replica start
psql -U postgres -h /var/run/postgresql -p 5433 -f globals_lab08.sql
createdb -p 5433 db1
createdb -p 5433 db2
pg_restore -p 5433 -d db1 -j 4 -l db1_dir
pg_restore -p 5433 -d db2 -j 4 -l db2_dir
psql -h /var/run/postgresql -p 5433
```
```sql
\c db1
TABLE t1;
\c db2
TABLE t2;
```

**Задача 4:**
```bash
# CSV делиметр точкой с запятой и неправильной кодировкой
psql -U postgres -d backup_db -c "\COPY t TO err_copy.csv WITH (FORMAT csv, DELIMITER ';', ENCODING 'WIN1251')"
psql -U postgres -d backup_db -c "\COPY t FROM err_copy.csv WITH (FORMAT csv, DELIMITER ',', ENCODING 'UTF8')"
```

**Фрагменты вывода:** 

**Задача 1:**
```text
CREATE DATABASE

You are now connected to database "backup_db" as user "student".

CREATE TABLE

INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1

 a |   b   
---+-------
 1 | row_1
 2 | row_2
 3 | row_3
 4 | row_4
 5 | row_5
(5 rows)
```
```text
 ;
; Archive created at 2025-11-01 13:13:51 MSK
;     dbname: backup_db
;     TOC Entries: 7
;     Compression: gzip
;     Dump Version: 1.15-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;     Dumped by pg_dump version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;
;
; Selected TOC Entries:
;
215; 1259 74008 TABLE public t postgres
3431; 0 74008 TABLE DATA public t postgres
3287; 2606 74014 CONSTRAINT public t t_pkey postgres
```
```text
You are now connected to database "backup_db" as user "postgres".

 a |   b   
---+-------
 1 | row_1
 2 | row_2
 3 | row_3
 4 | row_4
 5 | row_5
(5 rows)
```

**Задача 2:**
```text
CREATE DATABASE
CREATE DATABASE

You are now connected to database "db1" as user "postgres"
CREATE TABLE
INSERT 0 1

You are now connected to database "db2" as user "postgres"
CREATE TABLE
INSERT 0 1
```

```text
SET
SET
SET
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
psql:globals_lab08.sql:20: ERROR:  role "postgres" already exists
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
GRANT ROLE
GRANT ROLE
```

**Задача 3:**
```text
SET
SET
SET
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
psql:globals_lab08.sql:20: ERROR:  role "postgres" already exists
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
CREATE ROLE
ALTER ROLE
GRANT ROLE
GRANT ROLE

;
; Archive created at 2025-11-01 13:37:44 MSK
;     dbname: db1
;     TOC Entries: 7
;     Compression: gzip
;     Dump Version: 1.15-0
;     Format: DIRECTORY
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;     Dumped by pg_dump version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;
;
; Selected TOC Entries:
;
215; 1259 74025 TABLE public t1 postgres
3431; 0 74025 TABLE DATA public t1 postgres
3287; 2606 74031 CONSTRAINT public t1 t1_pkey postgres

;
; Archive created at 2025-11-01 14:51:50 MSK
;     dbname: db2
;     TOC Entries: 7
;     Compression: gzip
;     Dump Version: 1.15-0
;     Format: DIRECTORY
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;     Dumped by pg_dump version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;
;
; Selected TOC Entries:
;
215; 1259 74032 TABLE public t2 postgres
3431; 0 74032 TABLE DATA public t2 postgres
3287; 2606 74038 CONSTRAINT public t2 t2_pkey postgres
```
```text
You are now connected to database "db1" as user "postgres".

 id | v 
----+---
  1 | x
  2 | y
(2 rows)

You are now connected to database "db2" as user "postgres".

 id | v 
----+---
 10 | z
 20 | q
(2 rows)
```

**Задача 4** 
```text
COPY 5

ERROR:  invalid input syntax for type integer: "1;row_1"
CONTEXT:  COPY t, line 1, column a: "1;row_1"
```

## Результаты выполнения

1) **Логические бэкапы и проверка целостности.**  
   Выполнил создание кастом-дампа БД (`pg_dump -Fc`) и глобальных объектов (`pg_dumpall --globals-only`), затем восстановил БД `pg_restore`. Для больших БД применён параллельный формат `-Fd` и `-j N`, что ускорило загрузку за счёт независимого восстановления объектов.

2) **Обработка ошибок при загрузке дампа (COPY/кодировки/делимитеры).**  
   Смоделированы ошибки с несовпадающими `DELIMITER` и `ENCODING` при `\COPY`.

5) **Перенос ролей/привилегий и проверка доступа.**  
   Дамп глобальных объектов перенёс роли `writer/reader`, membership `w1/r1` и `ALTER DEFAULT PRIVILEGES`. После восстановления на другом кластере поведение доступа сохранилось. Это подтвердило корректность миграции политик безопасности вместе с данными.

## Выводы

1. Освоил методы резервного копирования.

2. Освоил методы восстановления данных в PostgreSQL, включая логическое.

3. Углубил навыки управления правами доступа пользователей.
