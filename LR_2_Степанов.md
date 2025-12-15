# Отчет по лабораторной работе №2
# Организация данных и системный каталог

**Дата:** 2025-11-26  
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Степанов Дмитрий Александрович 

## Цель работы
Всестороннее изучение логической и физической структуры хранения данных в PostgreSQL. Получение практических навыков управления базами данных, схемами, табличными пространствами. Глубокое освоение работы с системным каталогом для извлечения метаинформации. Исследование низкоуровневых аспектов хранения, включая TOAST.

## Теоретическая часть
### 1. Логическая структура: 
Иерархия: Кластер БД -> Базы данных -> Схемы -> Объекты (таблицы, представления и т.д.).

### 2. Физическая структура: 
Табличные пространства связывают логические объекты с физическим расположением файлов на диске. Данные хранятся в файлах.

### 3. Системный каталог: 
Набор системных таблиц (`pg_class`, `pg_database`, `pg_namespace`) и представлений (`pg_tables`, `pg_views`), содержащих метаданные. 

### 4. Низкоуровневое хранение: 
Механизм TOAST для больших данных, нежурналируемые таблицы, стратегии хранения столбцов.

## Практическая часть

### Часть 1. Базы данных и схемы
**Выполненные задачи:**   
1. Создал новую базу данных `lab02_db`. Проверил ее начальный размер с помощью `pg_database_size('lab02_db')`.  

2. Подключился к `lab02_db`. Создал две схемы: `app` и схему с именем пользователя ОС. В каждой схеме создал по одной таблице и вставил в них данные.  

3. Снова проверил размер базы данных. Она изменилась, так как после создания схем/таблиц и вставки строк каталог базы получает новые файлы отношений, что увеличивает размер базы данных.

4. Настроил параметр `search_path` для текущего сеанса так, чтобы при обращении по неполному имени приоритет имела моя пользовательская схема, а затем схема `app`. Продемонстрировал работу, обратившись к таблицам без указания схемы.  

5. Для базы `lab02_db` установил значение параметра `temp_buffers` так, чтобы в каждом новом сеансе, подключенном к этой БД, оно было в 4 раза больше значения по умолчанию. Проверил работу.

**Команды:**

**Задача 1:**
```bash
psql
```
```sql
CREATE DATABASE lab02_db;
SELECT pg_database_size('lab02_db');
```

**Задача 2:**
```sql
\c lab02_db
CREATE SCHEMA app;
CREATE SCHEMA student;
CREATE TABLE app.products(id serial PRIMARY KEY, name text NOT NULL);
INSERT INTO app.products(name) VALUES ('Keyboard'), ('Mouse');
CREATE TABLE student.users(id serial PRIMARY KEY, login text UNIQUE);
INSERT INTO student.users(login) VALUES ('Nikolay'), ('Anton');
```

**Задача 3:**
```sql
SELECT pg_database_size('lab02_db');
```

**Задача 4:**
```sql
SET search_path TO student, app, public;
SELECT * FROM products;
```

```sql
SELECT * FROM users;
```

```sql
SELECT * FROM products;
```

**Задача 5:**
```sql
SHOW temp_buffers;
ALTER DATABASE lab02_db SET temp_buffers = '32MB';
```
```sql
\c lab02_db  --сделал новый сеанс
SHOW temp_buffers;
```

**Фрагменты вывода:** 

**Задача 1:**
```text
 pg_database_size 
------------------
          7602703
(1 row)
```
```text
                config_file
-----------------------------------------
 /etc/postgresql/16/main/postgresql.conf
(1 row)
```

**Задача 2:**
```text
CREATE SCHEMA
CREATE SCHEMA
CREATE TABLE
INSERT 0 2
CREATE TABLE
INSERT 0 2
```

**Задача 3:**
```text
 pg_database_size 
------------------
          7926243
(1 row)
```

**Задача 4:**
```text
SET
```

```text
 id |  login  
----+---------
  1 | Nikolay
  2 | Anton
(2 rows)
```

```text
 id |   name   
----+----------
  1 | Keyboard
  2 | Mouse
(2 rows)

```


**Задача 5:**
```text
 temp_buffers 
--------------
 8MB
(1 row)

ALTER DATABASE
```
```text
 temp_buffers 
--------------
 32MB
(1 row)
```

---

### Часть 2. Системный каталог
**Выполненные задачи:**   
1.	Получил описание системной таблицы `pg_class` (команда `\d pg_class`).  

2.	Получил подробное описание представления `pg_tables` (команда`\d+ pg_tables`). Таблица — физически хранимые строки в файле(ах) на диске. Представление — сохранённый запрос к данным, собственных строк не хранит, а при обращении выполняет базовый SQL. `pg_tables` — представление, агрегирующее метаданные из внутренних таблиц (например, `pg_class`, `pg_namespace`). Его `\d+` показывает столбцы и источники.

3.	В базе `lab02_db` создал временную таблицу. Получил полный список всех схем в этой БД, включая системные (`pg_catalog`, `information_schema`). Для каждой сессии PostgreSQL создаёт собственную временную схему вида `pg_temp_N`. Временные таблицы живут только в рамках соединения и физически размещаются именно там, поэтому в списке `pg_namespace` видим дополнительную схему `pg_temp_*`.

4.	Получил список всех представлений в схеме `information_schema`.

5.	Выполнил в `psql` команду `\d+ pg_views`. Эта метакоманда клиента psql читает структуру представления из каталога: имена/типы столбцов и, главное, — текст SQL-определения, который хранится в `pg_views.definition`.

**Команды:**

**Задача 1:**
```sql
\d+ pg_class
```

**Задача 2:**
```sql
\d+ pg_tables
```

**Задача 3:**
```sql
CREATE TEMP TABLE t_tmp(x int);
SELECT n.oid, n.nspname
FROM pg_namespace n
ORDER BY 1;
```

**Задача 4:**
```sql
SELECT table_name
FROM information_schema.views
WHERE table_schema = 'information_schema'
ORDER BY 1;
```

```sql
SELECT * FROM users;
```

```sql
SELECT * FROM products;
```

**Задача 5:**
```sql
\d+ pg_views
```


**Фрагменты вывода:** 

**Задача 1:**
```text
                                                 Table "pg_catalog.pg_class"
       Column        |     Type     | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
---------------------+--------------+-----------+----------+---------+----------+-------------+--------------+-------------
 oid                 | oid          |           | not null |         | plain    |             |              | 
 relname             | name         |           | not null |         | plain    |             |              | 
 relnamespace        | oid          |           | not null |         | plain    |             |              | 
...
```

**Задача 2:**
```text
                          View "pg_catalog.pg_tables"
   Column    |  Type   | Collation | Nullable | Default | Storage | Description 
-------------+---------+-----------+----------+---------+---------+-------------
 schemaname  | name    |           |          |         | plain   | 
 tablename   | name    |           |          |         | plain   | 
 tableowner  | name    |           |          |         | plain   | 
 tablespace  | name    |           |          |         | plain   | 
 hasindexes  | boolean |           |          |         | plain   | 
 hasrules    | boolean |           |          |         | plain   | 
 hastriggers | boolean |           |          |         | plain   | 
 rowsecurity | boolean |           |          |         | plain   | 
View definition:
 SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    pg_get_userbyid(c.relowner) AS tableowner,
    t.spcname AS tablespace,
    c.relhasindex AS hasindexes,
    c.relhasrules AS hasrules,
    c.relhastriggers AS hastriggers,
    c.relrowsecurity AS rowsecurity
   FROM pg_class c
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_tablespace t ON t.oid = c.reltablespace
  WHERE c.relkind = ANY (ARRAY['r'::"char", 'p'::"char"]);
```

**Задача 3:**
```text
  oid  |      nspname       
-------+--------------------
    11 | pg_catalog
    99 | pg_toast
  2200 | public
 13296 | information_schema
 16391 | app
 16392 | student
 16413 | pg_temp_19
 16414 | pg_toast_temp_19
(8 rows)
```

**Задача 4:**
```text
              table_name               
---------------------------------------
 _pg_foreign_data_wrappers
 _pg_foreign_servers
 _pg_foreign_table_columns
 _pg_foreign_tables
 _pg_user_mappings
 administrable_role_authorizations
 applicable_roles
 attributes
 character_sets
 check_constraint_routine_usage
 check_constraints
 collation_character_set_applicability
 collations
 column_column_usage
 column_domain_usage
 column_options
 column_privileges
 column_udt_usage
 columns
 constraint_column_usage
 constraint_table_usage
 data_type_privileges
 domain_constraints
 domain_udt_usage
 domains
 element_types
 enabled_roles
 foreign_data_wrapper_options
 foreign_data_wrappers
 foreign_server_options
 foreign_servers
 foreign_table_options
 foreign_tables
 information_schema_catalog_name
 key_column_usage
 parameters
 referential_constraints
 role_column_grants
 role_routine_grants
 role_table_grants
 role_udt_grants
 role_usage_grants
 routine_column_usage
 routine_privileges
 routine_routine_usage
 routine_sequence_usage
 routine_table_usage
 routines
 schemata
 sequences
 table_constraints
 table_privileges
 tables
 transforms
 triggered_update_columns
 triggers
 udt_privileges
 usage_privileges
 user_defined_types
 user_mapping_options
 user_mappings
 view_column_usage
 view_routine_usage
 view_table_usage
 views
(65 rows)
```

```text
 id |  login  
----+---------
  1 | Nikolay
  2 | Anton
(2 rows)
```

```text
 id |   name   
----+----------
  1 | Keyboard
  2 | Mouse
(2 rows)

```


**Задача 5:**
```text
                          View "pg_catalog.pg_views"
   Column   | Type | Collation | Nullable | Default | Storage  | Description 
------------+------+-----------+----------+---------+----------+-------------
 schemaname | name |           |          |         | plain    | 
 viewname   | name |           |          |         | plain    | 
 viewowner  | name |           |          |         | plain    | 
 definition | text |           |          |         | extended | 
View definition:
 SELECT n.nspname AS schemaname,
    c.relname AS viewname,
    pg_get_userbyid(c.relowner) AS viewowner,
    pg_get_viewdef(c.oid) AS definition
   FROM pg_class c
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'v'::"char";

```


## Результаты выполнения
1. **Исследование конфигурации и параметров сервера.**  
   Были определены пути к основным конфигурационным файлам (`postgresql.conf`, `postgresql.auto.conf`), проанализированы контексты параметров (`postmaster`, `sighup`, `user`) и их влияние на применение настроек. Использование `pg_settings` и `pg_file_settings` позволило зафиксировать источники параметров и отследить ошибки при загрузке конфигурации. Это подтвердило корректную структуру конфигурации и позволило различать параметры, требующие перезапуска сервера, от тех, что могут быть изменены динамически.

2. **Практическое управление настройками на уровне экземпляра и сеанса.**  
   Применение команд `ALTER SYSTEM`, `SET`, `SET LOCAL` позволило сравнить уровни влияния конфигурационных изменений: глобальные (для всех пользователей), сеансовые и транзакционные. Результаты экспериментов показали, что изменения через `ALTER SYSTEM` записываются в файл `postgresql.auto.conf`, тогда как `SET` и `SET LOCAL` ограничиваются текущим контекстом работы.

## Выводы
1. Всесторонне изучил логическую и физическую структуры хранения данных в PostgreSQL. 

2. Получил практические навыки управления базами данных, схемами, табличными пространствами. 
