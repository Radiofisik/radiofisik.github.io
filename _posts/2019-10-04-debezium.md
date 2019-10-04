---
title: Синхронизация баз Debezium
description: Kafka, Postgres, Debezium
---

В микросервисной архитектуре часто возникает задача синхронизации баз данных например для организации поиска по сущностям разных микросервисов одновременно. Как вариант решения - проект debezium

### Топология

```
                   +-------------+
                   |             |
                   | PostgreSQL  |
                   |             |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |  (Debezium, JDBC connectors)     |
          |                                  |
          +---------------+------------------+
                          |
                          |
                          |
                          |
                  +-------v--------+
                  |                |
                  |   PostgreSQL   |
                  |                |
                  +----------------+


```
Используем компоненты
* Kafka
  * ZooKeeper
  * Kafka Broker
  * Kafka Connect with [Debezium](http://debezium.io/) and  [JDBC](https://github.com/confluentinc/kafka-connect-jdbc) Connectors
* PostgreSQL

### Использование

Запустим синхронизацию

```shell
# Start the application
export DEBEZIUM_VERSION=1.0
docker-compose up -d

# Start PostgreSQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://docker:8083/connectors/ -d @sink.json

# Start source connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://docker:8083/connectors/ -d @source.json

```

в базе источнике

```shell
docker-compose exec postgressrc bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from inventory.customers"'
```

в базе назначения

```bash
docker-compose exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
```

список топиков

```bash
docker-compose exec kafka bash -c 'bin/kafka-topics.sh --zookeeper zookeeper:2181 --list'
```

послушать изменения

```bash
docker-compose exec kafka bash -c 'bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic customers --from-beginning'
```

чтобы все почистить

```bash
#delete
curl -i -X DELETE http://docker:8083/connectors/inventory-connector

curl -i -X DELETE http://docker:8083/connectors/jdbc-sink

docker-compose exec kafka bash -c 'bin/kafka-topics.sh --zookeeper zookeeper:2181 --delete --topic customers'

docker-compose exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "drop table customers"'
```

> Репозиторий на github https://github.com/Radiofisik/DbSynk