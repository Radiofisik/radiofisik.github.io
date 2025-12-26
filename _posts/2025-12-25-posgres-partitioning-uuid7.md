---
name: Секционирование в PostgreSQL по UUID V7
description: Время идет и придумывают все более совершенные решения, одним из них является идея запихнуть в uuid время чтобы базе было проще, и еще и секционировать по нему можно
---
В postgress v18 есть функции для работы с uuidv7 

## Развертывание
как и раньше развернем через docker-compose
```yml
version: '3.8'

services:
  postgres:
    build: .
    container_name: postgres18
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 123
      POSTGRES_DB: db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

```

dockerfile
```bash
FROM postgres:18

# Install pg_partman
RUN apt-get update && \
    apt-get install -y postgresql-18-partman && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

```

init.sql
```sql
-- Enable pg_partman extension
CREATE SCHEMA IF NOT EXISTS partman;
CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;

-- Grant usage on partman schema
GRANT USAGE ON SCHEMA partman TO postgres;
GRANT ALL ON ALL TABLES IN SCHEMA partman TO postgres;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA partman TO postgres;
GRANT EXECUTE ON ALL PROCEDURES IN SCHEMA partman TO postgres;

```

## Заполнение данными
```sql
CREATE TABLE mytable (  
    "Id" UUID PRIMARY KEY DEFAULT uuidv7(),  
    "CreatedDate" TIMESTAMPTZ DEFAULT NOW()  
);  
  
  
WITH cte AS (  
  SELECT  
    random() * INTERVAL '10 years' AS shift  
  FROM generate_series(1, 100000)  
)  
INSERT INTO mytable ("Id", "CreatedDate")  
SELECT  
  uuidv7(-shift),  
  NOW()-shift  
FROM cte;
```

## Секционирование
```sql
ALTER TABLE public.mytable RENAME to old_nonpartitioned_mytable;  
  
CREATE TABLE mytable (  
    "Id" UUID DEFAULT uuidv7() PRIMARY KEY,  
    "CreatedDate" TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE ("Id");  
  
CREATE INDEX IDX_EX_Primary_mytable on public.mytable("Id");  


SELECT partman.create_parent(  
    p_parent_table := 'public.mytable',
    p_control := 'Id',  
    p_interval := '6 month',  
    p_start_partition := '2000-01-01',
    p_time_encoder := 'partman.uuid7_time_encoder',
    p_time_decoder := 'partman.uuid7_time_decoder'
);
  
-- error tdue to the bug https://github.com/pgpartman/pg_partman/issues/795  
CALL partman.partition_data_proc(  
    p_parent_table := 'public.mytable'  
    , p_loop_count := 200  
    , p_interval := '6 month'  
    , p_source_table := 'public.old_nonpartitioned_mytable'
    );
```

create_parent создает сами партиции вида
```sql
create table public.mytable_p20000101
    partition of public.mytable
        FOR VALUES FROM ('00dc6acf-ac00-0000-0000-000000000000') TO ('00e01415-1400-0000-0000-000000000000');
```

partition_data_proc должен перекачивать данные, но из- за бага данные нужно тащить самостоятельно
```sql
DO $$  
DECLARE  
    BATCH_SIZE      integer := 1000;  
    rows_processed  integer := 0;  
    batch_count     integer := 0;  
    total_rows      integer := 0;  
    processed_ids   uuid[];  
BEGIN  
    SELECT count(*) INTO total_rows FROM old_nonpartitioned_mytable;  
    RAISE NOTICE 'Total rows to migrate: %', total_rows;  
  
    WHILE TRUE LOOP  
        -- Gather the Ids for this batch  
        SELECT ARRAY(  
            SELECT "Id"  
            FROM old_nonpartitioned_mytable  
            ORDER BY "Id"  
            LIMIT BATCH_SIZE  
            FOR UPDATE SKIP LOCKED  
        ) INTO processed_ids;  
  
        rows_processed := array_length(processed_ids, 1);  
  
        IF rows_processed IS NULL OR rows_processed = 0 THEN  
            EXIT;  
        END IF;  
  
        -- Insert rows in this batch  
        INSERT INTO mytable("Id", "CreatedDate")  
        SELECT "Id", "CreatedDate"  
        FROM old_nonpartitioned_mytable  
        WHERE "Id" = ANY(processed_ids);  
  
        -- Delete those rows from the old table  
        DELETE FROM old_nonpartitioned_mytable  
        WHERE "Id" = ANY(processed_ids);  
  
        batch_count := batch_count + rows_processed;  
  
        RAISE NOTICE 'Processed %/% rows (%.2f%%)', batch_count, total_rows, (batch_count::numeric/total_rows*100);  
    END LOOP;  
    RAISE INFO 'Migration complete: % rows migrated.', batch_count;  
END $$;

```

## Partition prunning

Если делать простые запросы вида
```sql
select *  
from mytable  
where "Id"='0151f2c1-63fd-7e5f-ade9-4a10d0f715bd';
```
планировщик понимает и в плане есть только нужная партиция
```sql
Seq Scan on mytable_p20150701 mytable  (cost=0.00..4.06 rows=1 width=24)
"  Filter: (""Id"" = '0151f2c1-63fd-7e5f-ade9-4a10d0f715bd'::uuid)"
```

однако допустим мы знаем дату и на эту дату может быть сгенерировано много id которые должны быть в одной партиции, но сами id заранее не знаем
так запрос вида
```sql
select *  
from mytable  
where uuid_extract_timestamp("Id")=uuid_extract_timestamp('0151f2c1-63fd-7e5f-ade9-4a10d0f715bd');
```
перебирает все партиции
```sql
Append  (cost=0.00..3359.74 rows=788 width=24)
  ->  Seq Scan on mytable_p20000101 mytable_1  (cost=0.00..33.55 rows=8 width=24)
"        Filter: (uuid_extract_timestamp(""Id"") = '2015-12-30 11:58:59.069+00'::timestamp with time zone)"
  ->  Seq Scan on mytable_p20000701 mytable_2  (cost=0.00..33.55 rows=8 width=24)
"        Filter: (uuid_extract_timestamp(""Id"") = '2015-12-30 11:58:59.069+00'::timestamp with time zone)"
  ->  Seq Scan on mytable_p20010101 mytable_3  (cost=0.00..33.55 rows=8 width=24)
"        Filter: (uuid_extract_timestamp(""Id"") = '2015-12-30 11:58:59.069+00'::timestamp with time zone)"
  ->  Seq Scan on mytable_p20010701 mytable_4  (cost=0.00..33.55 rows=8 width=24)
...

```

и даже так prunning не работает
```sql
SELECT *  
FROM mytable  
WHERE "Id" >= uuidv7('2015-12-30 11:58:59.069000+00:00' - now())  
  AND "Id" < uuidv7('2015-12-30 11:59:00.000000+00:00' - now());
```
работает только если совсем просто на уровне приложения вычислить границы и их передать
```sql
SELECT * FROM mytable  
WHERE "Id" >= '019b8ea3-0ca7-7000-8453-c5dea83a830c'::uuid  
  AND "Id" <  '019b8ea3-12d5-7000-82a7-1e869a7dc1bb'::uuid;
```
тогда получим простой план
```sql
Bitmap Heap Scan on mytable_p20260101 mytable  (cost=4.23..14.41 rows=8 width=24)
"  Recheck Cond: ((""Id"" >= '019b8ea3-0ca7-7000-8453-c5dea83a830c'::uuid) AND (""Id"" < '019b8ea3-12d5-7000-82a7-1e869a7dc1bb'::uuid))"
"  ->  Bitmap Index Scan on ""mytable_p20260101_Id_idx""  (cost=0.00..4.23 rows=8 width=0)"
"        Index Cond: ((""Id"" >= '019b8ea3-0ca7-7000-8453-c5dea83a830c'::uuid) AND (""Id"" < '019b8ea3-12d5-7000-82a7-1e869a7dc1bb'::uuid))"

```

> Использованные материалы
> https://www.postgresql.org/docs/18/functions-uuid.html
> https://minervadb.xyz/uuidv7-and-time-based-partitioning-in-postgresql-18/

