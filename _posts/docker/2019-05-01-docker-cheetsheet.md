---
title: Шпаргалка по docker
description: Часто используемые команды
---

`docker ps -a` - shows all docker processes including shutted down

`docker inspect container_name_or_id`

Часто логи начинают занимать много места, для проверки служит команда

```du -ch /var/lib/docker/containers/*/*-json.log | grep total```

Почистить можно с помощью команды

```truncate -s 0 path_to_log```

Не по докеру но не менее полезно преобразование путей из win в lin

```   cd "$(cygpath "C:\Users\Aleksandr\Desktop\docker-compose-master\example1")"```

