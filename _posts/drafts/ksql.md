---
title: KSQL
description: Агрегация таблиц через Kafka connect
---

Для начала экспериментов войдем в CLI ksql

```bash
docker-compose exec ksql-server ksql
```

Можно посмотреть список топиков

```bash
ksql> LIST TOPICS;

 Kafka Topic        | Registered | Partitions | Partition Replicas | Consumers | ConsumerGroups
------------------------------------------------------------------------------------------------
 _schemas           | false      | 1          | 1                  | 0         | 0
 customers          | false      | 1          | 1                  | 1         | 1
 my_connect_configs | false      | 1          | 1                  | 0         | 0
 my_connect_offsets | false      | 25         | 1                  | 0         | 0
 my_connect_status  | false      | 5          | 1                  | 0         | 0
------------------------------------------------------------------------------------------------

ksql> show tables;

 Table Name | Kafka Topic | Format | Windowed
----------------------------------------------
----------------------------------------------


```

Можно вывести топик

```sql
ksql> print 'customers';
Format:STRING
```



Скажем чтобы читал с начала

```sql
SET 'auto.offset.reset' = 'earliest';
```



У нас есть таблица customers и из нее производится топик customers. KSQL не работает напрямую с топиками, для запросов надо сначала создать стрим или таблицу.

```sql
CREATE STREAM DEVICE_RAW (payload VARCHAR) WITH (KAFKA_TOPIC='customers',VALUE_FORMAT='JSON');
SELECT EXTRACTJSONFIELD(payload,'$.email') FROM DEVICE_RAW;

CREATE STREAM customers AS SELECT EXTRACTJSONFIELD(payload,'$.id') "id", EXTRACTJSONFIELD(payload,'$.email') "email", EXTRACTJSONFIELD(payload,'$.first_name') "firstname"  FROM DEVICE_RAW;
```

Для проверки можно запустить

```sql
ksql> describe extended customers;
```

Убедимся что можем выбрать данные из потока

```sql
ksql> SELECT id, email, firstname FROM customers;
1001 | sally.thomas@acme.com | Sally
1002 | gbailey@foobar.com | George
1003 | ed@walker.com | Edward
1004 | annek@noanswer.org | Anne
1004 | annek@noanswer.org | Anne2

```

