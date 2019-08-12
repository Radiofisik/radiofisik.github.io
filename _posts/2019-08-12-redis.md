---
title: Redis
description: Кэширование
---

Для ускорения работы приложений применяют кеширование данных, получение которых ресурсозатратно в связи с частотой запросов к ним или трудоемкостью вычислений. Для облегчения этой задачи существует несколько продуктов для хранения кэшируемых данных это своего рода базы данных хранящие данные в оперативной памяти. Наиболее известные из них Redis и Memcached. При этом Redis обладает большими возможностями. так он поддерживает следующие структуры данных

- String - на самом деле может хранить не только строку, но и число и число с плавающей точкой, причем поддерживаются атомарные операции инкремента и декремента
- List - список строк - поддерживает операции добавления с обоих концов, нахождение и удаление элемента по значению, обрезка по смещению
- Set - неупорядоченное множество - поддерживает операции добавления, удаления, пересечения, объединения...
- Hash - хеш таблица ключ - значение
- ZSET (Sorted set) - отображение строк на float score упорядоченное по score

Также поддерживает режим pub/sub и сохранение на диск. В то время как memcached это простое key-value хранилище.

## Запуск в docker

для запуска используем doccker-compose (`docker-compose up -d`) . Файл docker-compose.yml имеет вид

```yml
version: '3.5'

services:
 redis:
    restart: always
    image: redis:latest
    ports:
      - 6379:6379
```

Получить доступ к консоли контейнера можно командой `docker exec -it redis_redis_1 bash`

## Работа через консоль

### Работа со строками

```bash
root@5c3f83a1f4ef:/data# redis-cli
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> get hello
"world"
127.0.0.1:6379> del hello
(integer) 1
127.0.0.1:6379> get hello
(nil)
```

### Работа со списками

```bash
127.0.0.1:6379> rpush list-key 1
(integer) 1
127.0.0.1:6379> rpush list-key 2
(integer) 2
127.0.0.1:6379> rpush list-key item
(integer) 3
127.0.0.1:6379> lrange list-key 0 -1
1) "1"
2) "2"
3) "item"
127.0.0.1:6379> lindex list-key 1
"2"
127.0.0.1:6379> lpop list-key
"1"
127.0.0.1:6379> lrange list-key 0 -1
1) "2"
2) "item"
```

### Работа с множествами

```bash
127.0.0.1:6379> sadd set-key item
(integer) 1
127.0.0.1:6379> sadd set-key item2
(integer) 1
127.0.0.1:6379> sadd set-key item3
(integer) 1
127.0.0.1:6379> sadd set-key item
(integer) 0
127.0.0.1:6379> smembers set-key
1) "item3"
2) "item2"
3) "item"
127.0.0.1:6379> sismember set-key item4
(integer) 0
127.0.0.1:6379> sismember set-key item2
(integer) 1
127.0.0.1:6379> srem set-key item
(integer) 1
127.0.0.1:6379> smembers set-key
1) "item3"
2) "item2"
```

### Работа с Hash таблицей

```bash
127.0.0.1:6379> hset hash-key key1 value1
(integer) 1
127.0.0.1:6379> hset hash-key key2 value2
(integer) 1
127.0.0.1:6379> hset hash-key key3 value3
(integer) 1
127.0.0.1:6379> hgetall hash-key
1) "key1"
2) "value1"
3) "key2"
4) "value2"
5) "key3"
6) "value3"
127.0.0.1:6379> hdel hash-key key3
(integer) 1
127.0.0.1:6379> hdel hash-key key2
(integer) 1
127.0.0.1:6379> hget hash-key key1
"value1"
127.0.0.1:6379> hgetall hash-key
1) "key1"
2) "value1"
```

### Работа с сортированным множеством

```bash
127.0.0.1:6379> zadd zset-key 367 member0
(integer) 1
127.0.0.1:6379> zadd zset-key 777 member1
(integer) 1
127.0.0.1:6379> zadd zset-key 777 member1
(integer) 0
127.0.0.1:6379> zrange zset-key 0 -1 withscores
1) "member0"
2) "367"
3) "member1"
4) "777"
127.0.0.1:6379> zrangebyscore zset-key 0 400 withscores
1) "member0"
2) "367"
127.0.0.1:6379> zrem zset-key member1
(integer) 1
127.0.0.1:6379> zrange zset-key 0 -1 withscores
1) "member0"
2) "367"
```

### Чистка

Часто требуется почистить кэш, удалить все что там есть. для этого есть две команды `flushdb` - удаляет текущую БД и `flushall` - удаляет все

```bash
127.0.0.1:6379> info keyspace
# Keyspace
db0:keys=5,expires=0,avg_ttl=0
127.0.0.1:6379> keys *
1) "list-key"
2) "set-key"
3) "listKey"
4) "hash-key"
5) "zset-key"
127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> flushall
```

## C# клиент

Создадим простое приложение, которое использует Redis для временного хранения данных. Для начала установим пакеты

```bash
PM> Install-Package StackExchange.Redis
```

Простейшая операция - сохранение и получение строки

```c#
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("docker");
IDatabase db = redis.GetDatabase();

string value = "hello";
//положим на два часа
db.StringSet("world", value, TimeSpan.FromHours(2));

string valueFromRedis = db.StringGet("world");
Console.WriteLine(valueFromRedis);

//удалим
db.KeyDelete("world");
string valueFromRedis2 = db.StringGet("world");
Console.WriteLine(valueFromRedis2);
```

Таким образом можно сохранить любой объект, предварительно сериализовав его с помощью `Newton.Json`

> Репозиторий проекта https://github.com/Radiofisik/SimplestRedis