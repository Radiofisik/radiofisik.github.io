---
title: Секционирование в PostgreSQL
description: Что делать если количество данных растет и начинает тормозить систему
---
Со временем работы любой системы в ней накапливается большое количество данных, при этом наиболее используемые свежие данные, а старые же как правило нельзя удалить и обращений к ним мало, это своего рода архив. Однако эти данные лежат в общей с иттенсивно используемыми таблицах, увеличивают размет индексов и замедляют получение данных. Такую проблему можно решить секционированием.

В сети есть примеры нескольких реализаций секционирования, есть несколько устаревший подход с наследованием таблиц и утилитой которая на триггерах сооздает нужные секции  [https://github.com/2gis/partition_magic/blob/master/_2gis_partition_magic.sql](https://github.com/2gis/partition_magic/blob/master/_2gis_partition_magic.sql) однако для нашших целей такой подход не подошел. 

По сути надо секционировать ряд таблиц по дате создания некоторой базовой сущности, когда мы вытягиваем все данные мы сначала вытягиваем базовую сущность отдельным запросом, потом используем дату из нее в where остальных запросов. таким образом в остальных запросах данные будут вытягиваться из одной партиции, что сократит время поиска.

В документации [https://postgrespro.ru/docs/postgresql/10/ddl-partitioning](https://postgrespro.ru/docs/postgresql/10/ddl-partitioning) описан пример
```sql
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);

CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');

CREATE TABLE measurement_y2006m03 PARTITION OF measurement
    FOR VALUES FROM ('2006-03-01') TO ('2006-04-01');

CREATE TABLE measurement_y2007m11 PARTITION OF measurement
    FOR VALUES FROM ('2007-11-01') TO ('2007-12-01');

CREATE TABLE measurement_y2007m12 PARTITION OF measurement
    FOR VALUES FROM ('2007-12-01') TO ('2008-01-01')
    TABLESPACE fasttablespace;

CREATE TABLE measurement_y2008m01 PARTITION OF measurement
    FOR VALUES FROM ('2008-01-01') TO ('2008-02-01')
    WITH (parallel_workers = 4)
    TABLESPACE fasttablespace;

CREATE INDEX ON measurement_y2006m02 (logdate);
CREATE INDEX ON measurement_y2006m03 (logdate);
CREATE INDEX ON measurement_y2007m11 (logdate);
CREATE INDEX ON measurement_y2007m12 (logdate);
CREATE INDEX ON measurement_y2008m01 (logdate);

```
```sql
-- Inserting data into the measurement_y2006m02 partition
INSERT INTO measurement (city_id, logdate, peaktemp, unitsales)
VALUES
    (101, '2006-02-05', 55, 200),
    (102, '2006-02-14', 60, 250),
    (103, '2006-02-20', 50, 150);

-- Inserting data into the measurement_y2006m03 partition
INSERT INTO measurement (city_id, logdate, peaktemp, unitsales)
VALUES
    (104, '2006-03-10', 45, 100),
    (105, '2006-03-15', 48, 180),
    (106, '2006-03-25', 50, 220);

-- Inserting data into the measurement_y2007m11 partition
INSERT INTO measurement (city_id, logdate, peaktemp, unitsales)
VALUES
    (107, '2007-11-03', 38, 300),
    (108, '2007-11-18', 40, 320),
    (109, '2007-11-27', 42, 310);

-- Inserting data into the measurement_y2007m12 partition
INSERT INTO measurement (city_id, logdate, peaktemp, unitsales)
VALUES
    (110, '2007-12-02', 32, 400),
    (111, '2007-12-14', 35, 450),
    (112, '2007-12-28', 30, 410);

-- Inserting data into the measurement_y2008m01 partition
INSERT INTO measurement (city_id, logdate, peaktemp, unitsales)
VALUES
    (113, '2008-01-05', 25, 500),
    (114, '2008-01-18', 28, 550),
    (115, '2008-01-29', 27, 520);

```

если попытаться вставить что-то для чего еще нет секции получим ошибку
`[23514] ERROR: no partition of relation "measurement" found for row Detail: Partition key of the failing row contains (logdate) = (2024-01-05).`

## PGpackman
В поисках варианта для партиционирования был рассмотрен скрипт от Скрипт секционирования от 2Gis и ручной вариант описаный выше. Скрипт оказался не подходящим потому что использует условия равенства по целочисленному полю, а вручную поддерживать индексы и партиции довольно  трудозатратно и главное черевато ошибками.

Для экспериментов поднимем докер контейнер [https://github.com/dbsystel/postgresql-partman-container](https://github.com/dbsystel/postgresql-partman-container)

docker run ghcr.io/dbsystel/postgresql-partman

```bash
ARG POSTGRESQL_VERSION="15"
FROM bitnami/postgresql:$POSTGRESQL_VERSION
LABEL org.opencontainers.image.source="https://github.com/dbsystel/postgresql-partman-container"
ARG PARTMAN_VERSION="v5.0.1"
LABEL de.dbsystel.partman-version=$PARTMAN_VERSION
ARG POSTGRESQL_VERSION
LABEL de.dbsystel.postgres-version=$POSTGRESQL_VERSION
USER root
RUN install_packages wget gcc make build-essential
RUN cd /tmp \
    && wget "https://github.com/pgpartman/pg_partman/archive/refs/tags/${PARTMAN_VERSION}.tar.gz" \
    && export C_INCLUDE_PATH=/opt/bitnami/postgresql/include/:/opt/bitnami/common/include/ \
    && export LIBRARY_PATH=/opt/bitnami/postgresql/lib/:/opt/bitnami/common/lib/ \
    && export LD_LIBRARY_PATH=/opt/bitnami/postgresql/lib/:/opt/bitnami/common/lib/ \
    && tar zxf ${PARTMAN_VERSION}.tar.gz && cd pg_partman-${PARTMAN_VERSION#v}\
    && make \
    && make install \
    && cd .. && rm -r pg_partman-${PARTMAN_VERSION#v} ${PARTMAN_VERSION}.tar.gz

USER 1001

```

```yaml
version: '3.8'  # or whichever version you need

services:
  postgresql-partman:
    image: ghcr.io/dbsystel/postgresql-partman:latest
    build:
      context: .
      dockerfile: Dockerfile
    container_name: my_postgresql_partman
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydatabase
    ports:
      - "5432:5432"
    volumes:
      - postgresql_data:/var/lib/postgresql/data  # Assuming you want to persist the database data

volumes:
  postgresql_data:

```

начнем эксперименты [https://github.com/pgpartman/pg_partman/blob/5.0.1/doc/pg_partman_howto.md](https://github.com/pgpartman/pg_partman/blob/5.0.1/doc/pg_partman_howto.md)
```sql
CREATE SCHEMA partman ;
CREATE EXTENSION pg_partman SCHEMA partman ;
```

таблица с тестовыми данными
```sql
CREATE TABLE mytable (  
    "Id" UUID PRIMARY KEY DEFAULT gen_random_uuid(),  
    "CreatedDate" TIMESTAMPTZ DEFAULT NOW()  
);  
  
  
INSERT INTO mytable ("Id", "CreatedDate")  
SELECT gen_random_uuid(), NOW() - (random() * INTERVAL '10 years')  
FROM generate_series(1, 100000);
```
допустим нам надо ее секционировать по дате с итервалом 6 месяцев
```sql

  
ALTER TABLE public.mytable RENAME to old_nonpartitioned_mytable;  
  
CREATE TABLE mytable (  
    "Id" UUID,  
    "CreatedDate" TIMESTAMPTZ DEFAULT NOW(),  
    PRIMARY KEY ("Id", "CreatedDate")  
) PARTITION BY RANGE ("CreatedDate");  
  
CREATE INDEX IDX_EX_Primary_mytable on public.mytable("Id");  
  
CREATE INDEX on public.mytable("CreatedDate");  
  
SELECT partman.create_parent(  
    p_parent_table := 'public.mytable'  
    , p_control := 'CreatedDate'  
    , p_interval := '6 month',  
    p_start_partition := '2000-01-01'  
);
  
CALL partman.partition_data_proc(  
    p_parent_table := 'public.mytable'  
    , p_loop_count := 200  
    , p_interval := '6 month'  
    , p_source_table := 'public.old_nonpartitioned_mytable'
```

```sql
INSERT INTO mytable ("Id", "CreatedDate")  
SELECT gen_random_uuid(), NOW() - (random() * INTERVAL '20 years')  
FROM generate_series(1, 100000);  
  
 -- проверка есть ли записи в таблице по умолчанию 
select * from partman.check_default();  

-- позволяет эти записи раскидать по партициям
select * from partman.partition_data_time('public.mytable', p_batch_count := 1000);  
  
-- должна запускаться по кроуну и созадвать новые партиции, но основывается на текущей дата, а не на данных в талбице  
select * from partman.run_maintenance('public.mytable');  

-- показывает разделы
select * from partman.show_partitions('public.mytable');
```

Добавление колонки в базовую таблицу
```sql
alter table mytable add column "SomeId" uuid;
update mytable set "SomeId" = gen_random_uuid();
```
проблемы нет колонка добавилась во все таблицы

Добавление индекса в базовую таблицу
```sql
create index on mytable("SomeId");
```

тоже проблемы нет, индексы создались на доченрих таблицах

Если выполнить запрос с условием по дате
```sql
select *  
from mytable  
where "CreatedDate" > now() - interval '2 day' and "CreatedDate" < now();
```
то план запроса будет включать только одну секцию
```sql
Append  (cost=1.34..327.24 rows=341 width=24) (actual time=0.025..0.050 rows=50 loops=1)
  Subplans Removed: 54
  ->  Bitmap Heap Scan on mytable_p20240701 mytable_1  (cost=1.89..14.99 rows=49 width=24) (actual time=0.025..0.047 rows=50 loops=1)
"        Recheck Cond: ((""CreatedDate"" > (now() - '2 days'::interval)) AND (""CreatedDate"" < now()))"
        Heap Blocks: exact=12
"        ->  Bitmap Index Scan on ""mytable_p20240701_CreatedDate_idx""  (cost=0.00..1.88 rows=49 width=0) (actual time=0.019..0.019 rows=50 loops=1)"
"              Index Cond: ((""CreatedDate"" > (now() - '2 days'::interval)) AND (""CreatedDate"" < now()))"
Planning Time: 2.718 ms
Execution Time: 0.113 ms

```

однако если в запросе убрать условие по дате придется пройтись по всем
```sql
Append  (cost=9.50..4286.76 rows=153380 width=24) (actual time=0.110..15.001 rows=100000 loops=1)
  ->  Bitmap Heap Scan on mytable_p20000101 mytable_1  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.009..0.009 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20000101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.005..0.005 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20000701 mytable_2  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20000701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20010101 mytable_3  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.004..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20010101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20010701 mytable_4  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20010701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20020101 mytable_5  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.004..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20020101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20020701 mytable_6  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20020701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20030101 mytable_7  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20030101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20030701 mytable_8  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.004..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20030701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20040101 mytable_9  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20040101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20040701 mytable_10  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.004..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20040701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20050101 mytable_11  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20050101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20050701 mytable_12  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20050701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20060101 mytable_13  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20060101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20060701 mytable_14  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20060701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20070101 mytable_15  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.004..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20070101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20070701 mytable_16  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20070701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20080101 mytable_17  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20080101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20080701 mytable_18  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20080701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20090101 mytable_19  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20090101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20090701 mytable_20  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.001 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20090701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20100101 mytable_21  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20100101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20100701 mytable_22  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.004 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20100701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20110101 mytable_23  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.001 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20110101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20110701 mytable_24  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.005..0.005 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20110701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20120101 mytable_25  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.001 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20120101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20120701 mytable_26  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20120701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20130101 mytable_27  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20130101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20130701 mytable_28  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20130701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20140101 mytable_29  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20140101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Index Only Scan using mytable_p20140701_pkey on mytable_p20140701 mytable_30  (cost=0.28..70.90 rows=3168 width=24) (actual time=0.024..0.358 rows=3168 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20150101_pkey on mytable_p20150101 mytable_31  (cost=0.28..116.21 rows=4942 width=24) (actual time=0.014..0.438 rows=4942 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20150701_pkey on mytable_p20150701 mytable_32  (cost=0.28..117.78 rows=4973 width=24) (actual time=0.013..0.506 rows=4973 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20160101_pkey on mytable_p20160101 mytable_33  (cost=0.28..116.96 rows=4992 width=24) (actual time=0.021..0.481 rows=4992 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20160701_pkey on mytable_p20160701 mytable_34  (cost=0.28..115.50 rows=4968 width=24) (actual time=0.024..0.464 rows=4968 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20170101_pkey on mytable_p20170101 mytable_35  (cost=0.28..114.84 rows=4997 width=24) (actual time=0.015..0.446 rows=4997 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20170701_pkey on mytable_p20170701 mytable_36  (cost=0.28..115.53 rows=4970 width=24) (actual time=0.016..0.450 rows=4970 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20180101_pkey on mytable_p20180101 mytable_37  (cost=0.28..115.18 rows=5020 width=24) (actual time=0.017..0.463 rows=5020 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20180701_pkey on mytable_p20180701 mytable_38  (cost=0.28..115.74 rows=4984 width=24) (actual time=0.013..0.460 rows=4984 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20190101_pkey on mytable_p20190101 mytable_39  (cost=0.28..116.27 rows=5019 width=24) (actual time=0.017..0.477 rows=5019 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20190701_pkey on mytable_p20190701 mytable_40  (cost=0.28..115.08 rows=5013 width=24) (actual time=0.017..0.490 rows=5013 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20200101_pkey on mytable_p20200101 mytable_41  (cost=0.28..117.38 rows=5093 width=24) (actual time=0.017..0.507 rows=5093 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20200701_pkey on mytable_p20200701 mytable_42  (cost=0.28..116.63 rows=4970 width=24) (actual time=0.015..0.451 rows=4970 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20210101_pkey on mytable_p20210101 mytable_43  (cost=0.28..117.42 rows=5096 width=24) (actual time=0.025..0.479 rows=5096 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20210701_pkey on mytable_p20210701 mytable_44  (cost=0.28..116.28 rows=5020 width=24) (actual time=0.023..0.465 rows=5020 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20220101_pkey on mytable_p20220101 mytable_45  (cost=0.28..116.92 rows=4916 width=24) (actual time=0.019..0.468 rows=4916 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20220701_pkey on mytable_p20220701 mytable_46  (cost=0.28..116.02 rows=5076 width=24) (actual time=0.021..0.589 rows=5076 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20230101_pkey on mytable_p20230101 mytable_47  (cost=0.28..116.30 rows=4948 width=24) (actual time=0.035..0.505 rows=4948 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20230701_pkey on mytable_p20230701 mytable_48  (cost=0.28..118.83 rows=5043 width=24) (actual time=0.020..0.470 rows=5043 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20240101_pkey on mytable_p20240101 mytable_49  (cost=0.28..117.47 rows=5026 width=24) (actual time=0.021..0.586 rows=5026 loops=1)
        Heap Fetches: 0
  ->  Index Only Scan using mytable_p20240701_pkey on mytable_p20240701 mytable_50  (cost=0.28..39.97 rows=1766 width=24) (actual time=0.027..0.178 rows=1766 loops=1)
        Heap Fetches: 0
  ->  Bitmap Heap Scan on mytable_p20250101 mytable_51  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20250101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.002..0.002 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20250701 mytable_52  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.002..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20250701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20260101 mytable_53  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20260101_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.003..0.003 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_p20260701 mytable_54  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.003..0.003 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_p20260701_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
  ->  Bitmap Heap Scan on mytable_default mytable_55  (cost=9.50..35.20 rows=1570 width=24) (actual time=0.001..0.002 rows=0 loops=1)
        ->  Bitmap Index Scan on mytable_default_pkey  (cost=0.00..9.10 rows=1570 width=0) (actual time=0.001..0.001 rows=0 loops=1)
Planning Time: 1.131 ms
Execution Time: 18.737 ms

```