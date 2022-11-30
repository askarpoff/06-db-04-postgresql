# Домашнее задание к занятию "6.4. PostgreSQL" - Карпов Андрей, devops-21

## Задача 1

Используя docker поднимите инстанс PostgreSQL (версию 13). Данные БД сохраните в volume.

Подключитесь к БД PostgreSQL используя `psql`.

Воспользуйтесь командой `\?` для вывода подсказки по имеющимся в `psql` управляющим командам.

**Найдите и приведите** управляющие команды для:
- вывода списка БД
- подключения к БД
- вывода списка таблиц
- вывода описания содержимого таблиц
- выхода из psql

### Ответ:
- вывода списка БД
```
\l[+]   [PATTERN]      list databases
```
- подключения к БД
```
\c[onnect] {[DBNAME|- USER|- HOST|- PORT|-] | conninfo}
                         connect to new database (currently "postgres")
```
- вывода списка таблиц
```
\d[S+]                 list tables, views, and sequences
```
- вывода описания содержимого таблиц
```
\d[S+]  NAME           describe table, view, sequence, or index
```
- выхода из psql
```
\q                     quit psql
```

## Задача 2

Используя `psql` создайте БД `test_database`.

Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/virt-11/06-db-04-postgresql/test_data).

Восстановите бэкап БД в `test_database`.

Перейдите в управляющую консоль `psql` внутри контейнера.

Подключитесь к восстановленной БД и проведите операцию ANALYZE для сбора статистики по таблице.

Используя таблицу [pg_stats](https://postgrespro.ru/docs/postgresql/12/view-pg-stats), найдите столбец таблицы `orders` 
с наибольшим средним значением размера элементов в байтах.

**Приведите в ответе** команду, которую вы использовали для вычисления и полученный результат.

### Ответ:
```
postgres@53e13bf22bee:~$ psql test_database
psql (13.9 (Debian 13.9-1.pgdg110+1))
Type "help" for help.

test_database=# ANALYZE;
ANALYZE
test_database=# select attname, avg_width from pg_stats where tablename = 'orders' order by avg_width desc limit 1;
 attname | avg_width
---------+-----------
 title   |        16
(1 row)
```
## Задача 3

Архитектор и администратор БД выяснили, что ваша таблица orders разрослась до невиданных размеров и
поиск по ней занимает долгое время. Вам, как успешному выпускнику курсов DevOps в нетологии предложили
провести разбиение таблицы на 2 (шардировать на orders_1 - price>499 и orders_2 - price<=499).

Предложите SQL-транзакцию для проведения данной операции.
### Транзакция
```sql
begin;

CREATE TABLE new_orders (like orders including defaults including constraints including indexes);
CREATE TABLE orders_1 (CHECK (price > 499)) INHERITS (new_orders);
CREATE TABLE orders_2 (CHECK (price <= 499)) INHERITS (new_orders);


CREATE OR REPLACE FUNCTION orders_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF ( NEW.price > 499 ) THEN
        INSERT INTO orders_1 VALUES (NEW.*);
    ELSIF ( NEW.price <= 499) THEN
        INSERT INTO orders_2 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Date out of range.';
    END IF;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER insert_orders_trigger
    BEFORE INSERT ON new_orders
    FOR EACH ROW EXECUTE PROCEDURE orders_insert_trigger();


insert into new_orders (id,title,price) select * from orders;

ALTER SEQUENCE orders_id_seq OWNED BY new_orders.id;
drop table orders;

alter table new_orders rename to orders;

commit;
```
Можно ли было изначально исключить "ручное" разбиение при проектировании таблицы orders?

### Ответ:
Я думаю, что "декларативным разбиением"(Declarative Partitioning):
```sql
CREATE TABLE orders (
    id integer NOT NULL,
    title character varying(80) NOT NULL,
    price integer DEFAULT 0
);
CREATE TABLE orders_1 (
    check (price>499)
) inherits (public.orders);

CREATE TABLE orders_2 (
    check (price<=499)
) inherits (public.orders);
CREATE RULE orders_1_insert as on INSERT TO orders WHERE (price>499) DO INSTEAD INSERT INTO orders_1 values(NEW.*);
CREATE RULE orders_2_insert as on INSERT TO orders WHERE (price<=499) DO INSTEAD INSERT INTO orders_2 values(NEW.*);
```

## Задача 4

Используя утилиту `pg_dump` создайте бекап БД `test_database`.

Как бы вы доработали бэкап-файл, чтобы добавить уникальность значения столбца `title` для таблиц `test_database`?

### Ответ:
```bash
postgres@53e13bf22bee:~$ pg_dump test_database > backup.sql
```
Дописать в SQL огрничение при создании таблицы - **UNIQUE**
```
 CREATE TABLE orders (
    id integer NOT NULL,
    title character varying(80) NOT NULL UNIQUE,
    price integer DEFAULT 0
);
```
