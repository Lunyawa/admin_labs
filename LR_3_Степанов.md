# Отчет по лабораторной работе №3
# Модель многопользовательского доступа: MVCC

**Дата:** 2025-11-27  
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Степанов Дмитрий Александрович 

## Цель работы
Изучить принципы многоверсионного управления конкурентным доступом (MVCC) в PostgreSQL. Получить практические навыки наблюдения за работой MVCC, анализа версий строк, снимков данных и уровней изоляции транзакций. Освоить использование расширений и системных представлений для исследования внутренней структуры данных.

## Теоретическая часть
**MVCC (Multiversion Concurrency Control)** — механизм, позволяющий нескольким транзакциям работать с одними и теми же данными одновременно, минимизируя блокировки. Каждая транзакция видит согласованный «снимок» данных на момент своего
начала. 
1. Версии строк: При изменении строки создается ее новая версия. Старая версия остается в таблице до очистки.
2. Системные поля:
- xmin – идентификатор транзакции, создавшей версию строки.
- xmax — идентификатор транзакции, удалившей версию строки (или заблокировавшей ее для обновления).
- ctid — физическое расположение версии строки в таблице (номер страницы и позиции в ней).
3. Уровни изоляции: Определяют, какие аномалии параллелизма допустимы:
-Read Committed (По умолчанию): Виден только зафиксированный данные. Возможны неповторяемое чтение и фантомное чтение.
- Repeatable Read: Гарантирует, что данные, прочитанные в транзакции, не изменятся. Предотвращает неповторяемое чтение, возможны фантомы.
- Serializable: Самый строгий уровень, предотвращает все аномалии.
4. Снимок данных (Snapshot): Набор идентификаторов транзакций, активных на момент начала текущей транзакции. Определяет, какие версии строк видимы текущей транзакции.

## Практическая часть

### Часть 1. Уровни изоляции и аномалии
**Выполненные задачи:**   
1. Создал таблицу `iso_test (id INT, data TEXT)` и вставил одну строку. В сеансе 1 начал транзакцию с уровнем `READ COMMITTED` и выполнил `SELECT iso_test;. * FROM iso_test;`. В сеансе 2 удалил строку и зафиксировал изменения (`DELETE ...; COMMIT;`). В сеансе 1 выполнил тот же `SELECT` повторно. Вывело 0 строк. Завершил транзакцию в сеансе 1.

2. Повторил предыдущий эксперимент, но в сеансе 1 начал транзакцию с `BEGIN ISOLATION LEVEL REPEATABLE READ;`. `READ COMMITTED` берёт новый снимок на каждый запрос. `REPEATABLE READ` фиксирует снимок данных и держит его неизменным на всю транзакцию.

3. В сеансе 1 начал транзакцию и создал новую таблицу `new_table`, вставил в нее строку. Не фиксировал. В сеансе 2 выполнил `SELECT * FROM new_table;`. Выдало ошибку о несуществовании таблицы. Зафиксировал транзакцию в сеансе 1. Повторил запрос в сеансе 2, таблица появилась. Повторил процесс, но вместо фиксации откатил транзакцию в сеансе 1. После `ROLLBACK;` таблица также нету в сеансе 2.

4. В сеансе 1 начал транзакцию и выполнил `SELECT * FROM iso_test;`. Попытался в сеансе 2 выполнить `DROP TABLE iso_test;`. Удалить таблицу не получится, пока не завершим транзакцию в первом сеансе.

**Команды:**

**Задача 1:**
```sql
-- Сеанс 1
CREATE DATABASE lab03_db;
\c lab03_db
CREATE TABLE iso_test(id INT, data TEXT);
INSERT INTO iso_test VALUES (1,'row1');

BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT * FROM iso_test;
```
```sql
-- Сеанс 2
\c lab03_db
DELETE FROM iso_test WHERE id=1;
COMMIT;
```
```sql
-- Сеанс 1
SELECT * FROM iso_test;
COMMIT;
```

**Задача 2:**
```sql
-- Сеанс 1
INSERT INTO iso_test VALUES (1,'row1');
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM iso_test;
```
```sql
-- Сеанс 2
DELETE FROM iso_test WHERE id=1;
COMMIT;
```
```sql
-- Сеанс 1
SELECT * FROM iso_test;
COMMIT;
```

**Задача 3:**
```sql
-- Сеанс 1
BEGIN;
CREATE TABLE new_table(id int);
INSERT INTO new_table VALUES (10);
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```sql
-- Сеанс 1
COMMIT;
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```sql
-- Сеанс 1
DROP TABLE new_table;
BEGIN;
CREATE TABLE new_table(id int);
INSERT INTO new_table VALUES (10);
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```sql
-- Сеанс 1
ROLLBACK;
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
**Задача 4:**
```sql
-- Сеанс 1
BEGIN;
SELECT * FROM iso_test;
```

```sql
-- Сеанс 2
DROP TABLE iso_test;
```

```sql
-- Сеанс 1
COMMIT;
```

**Фрагменты вывода:** 

**Задача 1:**
```text
CREATE DATABASE
You are now connected to database "lab03_db" as user "student".

CREATE TABLE

INSERT 0 1

BEGIN

 id | data 
----+------
  1 | row1
(1 row)
```
```text
You are now connected to database "lab03_db" as user "student".

DELETE 1

WARNING:  there is no transaction in progress
COMMIT

```
```text
 id | data 
----+------
  1 | row1
(1 row)

COMMIT
```

**Задача 2:**
```text
INSERT 0 1

BEGIN

 id | data 
----+------
  1 | row1
(1 row)

```
```text
DELETE 1

WARNING:  there is no transaction in progress
COMMIT
lab03_db=# 
```

**Задача 3:**
```text
BEGIN
CREATE TABLE
INSERT 0 1
```
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
```
```text
COMMIT
```
```text
 id 
----
 10
(1 row)
```
```text
DROP TABLE
BEGIN
CREATE TABLE
INSERT 0 1
```
```text
ROLLBACK
```
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
                      ^
```
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
                      ^
```

**Задача 4:**
```text
-- Сеанс 1
BEGIN
 id | data 
----+------
(0 rows)
```

```text
-- Сеанс 1
COMMIT
```
```text
-- Сеанс 2
DROP TABLE
```

---

### Часть 2. Фантомное чтение и снимки
**Выполненные задачи:**   
1.	Создал пустую таблицу `phantom_test (id INT)`. Продемонстрировал на уровне `Read Committed`, что аномалия "фантомное чтение" не предотвращается (вставка новых строк в другом сеансе становится видимой).

2.	В сеансе 1 начал транзакцию с уровнем `Repeatable Read` (пока без запросов). В сеансе 2 удалил все строки из `phantom_test` и зафиксировал. В сеансе 1 выполнил `SELECT * FROM phantom_test;`. Удалённые стркои вывелись. Выполнил в сеансе 1 запрос `SELECT * FROM pg_database;` (не касаясь `phantom_test`). При последующем запросе также вывело удалённые строки.

3.	Убедился, что `DROP TABLE` является транзакционной операцией (можно откатить).

**Команды:**

**Задача 1:**
```sql
-- Сеанс 1
CREATE TABLE phantom_test(id INT);
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT count(*) FROM phantom_test;  -- 0
```
```sql
-- Сеанс 2
INSERT INTO phantom_test VALUES (1),(2),(3);
COMMIT;
```
```sql
-- Сеанс 1
SELECT count(*) FROM phantom_test;
COMMIT;
```

**Задача 2:**
```sql
-- Сеанс 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
```
```sql
-- Сеанс 2
DELETE FROM phantom_test;
COMMIT;
```
```sql
-- Сеанс 1
SELECT * FROM phantom_test; 
SELECT * FROM pg_database;   
SELECT * FROM phantom_test;
COMMIT;
```

**Задача 3:**
```sql
CREATE TABLE drop_demo(id int);
BEGIN;
DROP TABLE drop_demo;
ROLLBACK;

SELECT * FROM drop_demo;
```


**Фрагменты вывода:** 

**Задача 1:**
```text
-- Сеанс 1
CREATE TABLE

BEGIN

 count 
-------
     0
(1 row)
```
```text
-- Сеанс 2
INSERT 0 3

WARNING:  there is no transaction in progress
COMMIT
```
```text
-- Сеанс 1
 count 
-------
     3
(1 row)

COMMIT
```

**Задача 2:**
```text
-- Сеанс 1
BEGIN
```
```text
-- Сеанс 2
DELETE 3

WARNING:  there is no transaction in progress
COMMIT
```
```text
 id 
----
  1
  2
  3
(3 rows)
```

**Задача 3:**
```text
CREATE TABLE

BEGIN

DROP TABLE

ROLLBACK

 id 
----
(0 rows)
```


## Результаты выполнения
1. **Изучение уровней изоляции и поведения транзакций.**  
   На практике были сравнены уровни `READ COMMITTED` и `REPEATABLE READ`.  
   - При `READ COMMITTED` каждая команда SQL работает с новым снимком данных, поэтому после удаления строки в другом сеансе повторный `SELECT` возвращал пустое множество.  
   - При `REPEATABLE READ` снимок фиксируется на момент начала транзакции, и повторный запрос возвращал те же данные, даже если строка была удалена в другом сеансе.  
   Эти эксперименты подтвердили, что PostgreSQL реализует согласованное чтение без блокировок, используя снимки транзакций.

2. **Транзакционность DDL и контроль видимости объектов.**  
   Создание таблицы в одной транзакции не делает её видимой другим сеансам до фиксации (`COMMIT`) и устраняется при откате (`ROLLBACK`). Это демонстрирует, что DDL-операции подчиняются общим правилам транзакционной изоляции, а объекты БД участвуют в MVCC наряду с данными. 


## Выводы
1. Изучил принципы многоверсионного управления конкурентным доступом (MVCC) в PostgreSQL. 
2. Получил практические навыки наблюдения за работой MVCC, анализа версий строк, снимков данных и уровней изоляции транзакций. 
3. Освоил использование расширений и системных представлений для исследования внутренней структуры данных.