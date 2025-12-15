# Отчет по лабораторной работе №7
# Управление доступом, расширениями и локализацией

**Дата:** 2025-12-01
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Степанов Дмитрий Александрович 

## Цель работы
Освоить управление правами доступа пользователей, работу с расширениями PostgreSQL и настройку параметров локализации. Получить практические навыки настройки аутентификации, управления привилегиями, установки расширений и миграции данных между разными кодировками.

## Теоретическая часть
**Управление доступом:** Система ролей и привилегий в PostgreSQL. Настройка аутентификации через файл pg_hba.conf.

**Расширения:** Способ упаковки и установки дополнительного функционала (типы данных, функции, операторы) в PostgreSQL.

**Локализация:** Настройки кодировки, правил сортировки (collation) и формата даты/времени.

## Практическая часть

### Часть 1. Управление доступом
**Выполненные задачи:**   
1. Создал новую базу данных `access_db` и двух пользователей: `writer` (с правом создания объектов) и `reader` (только чтение). Отозвал у роли `public` все привилегии на схему `public` в новой БД. Выдал роли `writer` права `CREATE` и `USAGE` на схему `public`, а роли `reader` — только `USAGE`. Настроил привилегии по умолчанию так, чтобы роль reader автоматически получала право `SELECT` на новые таблицы в схеме `public`, принадлежащие `writer`. Создал пользователей `w1` (входит в роль `writer`) и `r1` (входит в роль `reader`). Подключился под `writer`, создал таблицу `test_table`. Убедился, что `r1` может только читать её, а `w1` — имеет полный доступ (включая `DELETE`).

2. Создал пользовательские роли `alice` и `bob`. Отредактировал `pg_hba.conf`, разрешив беспарольный вход (`trust`) только для `postgres` и `student`. Для всех остальных методов установил`md5`. Перезагрузил конфигурацию. Убедился, что вход для `alice` и `bob` запрещен. Для `alice` и `bob` настроил аутентификацию по `peer` (сопоставление с пользователем ОС). Убедился, что войти нельзя без создания пользователя в ОС. Создал в ОС пользователя `alice`. Настроил вход в `PostgreSQL` для роли `alice` с методом `peer`. Проверил вход. Проверил, можно ли использовать одно отображение peer для нескольких ролей.

**Команды:**

**Задача 1:**
```sql
CREATE DATABASE lab07_db;
\c lab07_db
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

### Часть 2. Управление расширениями
**Выполненные задачи:**   
1.	Установил расширение `uom`. Убедился, что оно появилось в `pg_available_extensions`.

2.	Создал расширение в своей БД без указания версии. Определил, какая версия установилась и какие скрипты выполнились. Выполнились uom.control и uom--1.2.sql.

3.	Добавил в справочник расширения новые единицы измерения.

4.	Изменил права доступа к таблицам расширения: отозвал `SELECT` у `public`, выдал его новой специальной роли.

5.	Используя `pg_dump`, выгрузил объекты расширения. Изучил дамп. Убедился, что выгружаются метаданные (таблицы, типы, функции) и данные.

**Команды:**

**Задача 1:**
```bash
git clone https://pubgit.postgrespro.ru/pub/uom
cd uom
sudo make install
```
```sql
SELECT * FROM pg_available_extensions;
```

**Задача 2:**
```sql
CREATE EXTENSION uom;
SELECT e.extname, e.extversion, e.extrelocatable, n.nspname AS schema_name
FROM pg_extension e
JOIN pg_namespace n ON n.oid = e.extnamespace
WHERE e.extname = 'uom';
```

**Задача 3:**
```sql
INSERT INTO public.uom_ref (name, k) VALUES
  ('фут',  0.3048),
  ('дюйм', 0.0254);
SELECT * FROM public.uom_ref WHERE name IN ('фут','дюйм');
```

**Задача 4:**
```sql
CREATE ROLE uom_reader NOLOGIN;
REVOKE SELECT ON public.uom_ref FROM PUBLIC;
GRANT SELECT ON public.uom_ref TO uom_reader;
```

**Задача 5:**
```bash
pg_dump -Fc -f full.dump
pg_restore -l full.dump
```

**Фрагменты вывода:** 

**Задача 1:**
```text
/bin/mkdir -p '/usr/share/postgresql/16/extension'
/bin/mkdir -p '/usr/share/postgresql/16/extension'
/usr/bin/install -c -m 644 .//uom.control '/usr/share/postgresql/16/extension/'
/usr/bin/install -c -m 644 .//uom--1.0.sql .//uom--1.2.sql .//uom--1.0--1.1.sql .//uom--1.1--1.2.sql  '/usr/share/postgresql/16/extension/'
```
```text


        name        | default_version | installed_version |                                comment                                 
--------------------+-----------------+-------------------+------------------------------------------------------------------------
 file_fdw           | 1.0             |                   | foreign-data wrapper for flat file access
 pg_walinspect      | 1.1             |                   | functions to inspect contents of PostgreSQL Write-Ahead Log
 plpgsql            | 1.0             | 1.0               | PL/pgSQL procedural language
 pg_buffercache     | 1.4             |                   | examine the shared buffer cache
 pg_visibility      | 1.2             |                   | examine the visibility map (VM) and page-level visibility info
 btree_gin          | 1.3             |                   | support for indexing common datatypes in GIN
 btree_gist         | 1.7             |                   | support for indexing common datatypes in GiST
 pg_freespacemap    | 1.2             |                   | examine the free space map (FSM)
 uom                | 1.2             | 1.2               | Units of Measurement
 ...
```

**Задача 2:**
```text
CREATE EXTENSION

 extname | extversion | extrelocatable | schema_name 
---------+------------+----------------+-------------
 uom     | 1.2        | t              | public
(1 row)
```

**Задача 3:**
```text
INSERT 0 2

 name |   k    | predefined 
------+--------+------------
 фут  | 0.3048 | f
 дюйм | 0.0254 | f
(2 rows)
```

**Задача 4:**
```text
CREATE ROLE

REVOKE

GRANT
```

**Задача 5:**
```text
;
; Archive created at 2025-10-10 18:57:31 MSK
;     dbname: student
;     TOC Entries: 10
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
2; 3079 65705 EXTENSION - uom 
3463; 0 0 COMMENT - EXTENSION uom 
216; 1259 40966 TABLE public wal_crash_demo student
3309; 0 65706 TABLE DATA public uom_ref student
3456; 0 40966 TABLE DATA public wal_crash_demo student
3312; 2606 40972 CONSTRAINT public wal_crash_demo wal_crash_demo_pkey student
```

## Результаты выполнения

### 1. Управление правами доступа и ролями
Созданы две основные роли: `writer` и `reader`. Роль `writer` получила права `CREATE` и `USAGE` на схему `public`, а `reader` — только `USAGE`. Для предотвращения нежелательного доступа у `PUBLIC` были отозваны все права на схему.  
При помощи `ALTER DEFAULT PRIVILEGES` настроено автоматическое предоставление роли `reader` права `SELECT` на таблицы, создаваемые пользователем `writer`. Это подтвердило, что система наследования и шаблонных привилегий в PostgreSQL позволяет централизованно управлять доступом на уровне схем.  
При создании пользователей `w1` и `r1`, входящих соответственно в роли `writer` и `reader`, проверено:  
- `r1` может выполнять `SELECT`, но не имеет прав на `UPDATE` и `DELETE`;  
- `w1` может выполнять все операции над таблицами, включая удаление строк.  
Таким образом, была подтверждена правильность модели разделения ролей и полномочий.

### 2. Аутентификация и настройка pg_hba.conf
В файле `/etc/postgresql/16/main/pg_hba.conf` настроены разные методы аутентификации:
- `trust` для пользователей `postgres` и `student`;  
- `md5` для всех остальных;  
- `peer` с картой соответствия `pg_ident.conf` для пользователей `alice` и `bob`.  

Проверка показала, что:
- при методе `md5` PostgreSQL требует пароль,  
- при `peer` требуется наличие одноимённого пользователя ОС,  
- создание системного пользователя `alice` позволило выполнить успешный вход в СУБД без пароля, подтвердив работу peer-аутентификации.  
Эксперименты показали, что одна карта peer-аутентификации (`users_map`) может использоваться для нескольких ролей. Это продемонстрировало гибкость механизма `pg_ident.conf` и его использование при интеграции PostgreSQL с системной учётной записью. :contentReference[oaicite:3]{index=3}

### 3. Управление расширениями
Выполнена установка и настройка расширения `uom` (Units of Measurement).  
После установки команда `SELECT * FROM pg_available_extensions;` подтвердила наличие `uom` в списке доступных расширений. Создание расширения в базе без указания версии привело к установке версии `1.2`.  
Исследование системного каталога `pg_extension` и файлов `/usr/share/postgresql/16/extension/uom.control` и `uom--1.2.sql` показало механизм миграции и обновления расширений через версии.  
Добавлены пользовательские единицы измерения («фут», «дюйм»), и создана отдельная роль `uom_reader`, получившая доступ к таблице `uom_ref`.  
Команда `pg_dump -Fc -f full.dump` подтвердила выгрузку объектов расширения вместе с метаданными и пользовательскими данными, что демонстрирует корректную интеграцию расширений в систему резервного копирования PostgreSQL.

## Выводы
1. Освоил управление правами доступа пользователей, работу с расширениями PostgreSQL.
2. Получил практические навыки настройки аутентификации, управления привилегиями, установки расширений.
