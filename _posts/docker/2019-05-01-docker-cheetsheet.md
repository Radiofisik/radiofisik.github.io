---
title: Шпаргалка по docker
description: Часто используемые команды. Возникающие проблемы и их решения
---

```bash
docker ps -a - shows all docker processes including shutted down
```

```bash
docker inspect container_name_or_id
```

```bash
docker diff container_name_or_id
```

## Логи

Просмотр логов

```bash
docker logs container_name_or_id
docker logs container_name_or_id --tail 1000
docker logs container_name_or_id --tail 1000 -f
```

Часто логи начинают занимать много места, для проверки служит команда

```bash
du -ch /var/lib/docker/containers/*/*-json.log | grep total
```

Почистить можно с помощью команды
```bash
truncate -s 0 path_to_log
```

Не по докеру но не менее полезно преобразование путей из win в lin
```bash
cd "$(cygpath "C:\Users\Aleksandr\Desktop\docker-compose-master\example1")"
```

## Остановка и чистка

Остановка контейнеров

```bash
docker stop $(docker ps -aq)
#избранно
docker stop $(docker ps | grep radiofisik| awk '{print $1}')
```

Удаление всех контейнеров и по фильтру

```bash
docker rm -v $(docker ps -aq -f status=exited)
docker rmi $(docker images| grep eshop) -f
```

Удаление всех образов

```bash
docker rmi $(docker images -q) --force
```

## Действия в контейнере

Войти в контейнер можно с помощью команды

```bash
docker exec -it container_name /bin/bash
```

### Список процессов

Бывает полезным посмотреть список процессов в виде

```bash
root@90e9c18af1db:/app# ps -aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4232   736 ?        Ss   10:37   0:00 tail -f /dev/null
root        13  0.8  4.9 3689992 102208 ?      SLsl 10:37   0:08 /remote_debugger/vsdbg --interpreter=vscode
root        25  0.7  5.1 7010728 104684 ?      SLl  10:37   0:07 /usr/bin/dotnet --additionalProbingPath /root/.nuget/pa
root       471  0.0  0.1  18136  3188 pts/0    Ss   10:50   0:00 /bin/bash
root       802  0.0  0.1  36636  2788 pts/0    R+   10:53   0:00 ps -aux
```

В случае когда образ минимальный и команде ps нет ее можно установить

```bash
apt-get update && apt-get install procps
```

### Захват трафика в сети контейнеров

Часто полезно посмотреть обмен трафиком между контейнерами для этого можно использовать привычные инструменты Charles и Wireshark. Оба они умеют работать с pcap файлами, которые можно захватить с помощью tcpdump. Чтобы найти в каком бридже необходимо запустить снифер выполним ряд команд

```bash
docker network ls
bridge link | grep 26e299704302
tcpdump -i br-26e299704302  -w capture.pcap

```

## Docker Machine && Docker Toolbox

Docker Toolbox по сути представляет собой виртуальную машину Virtual Box внутри которой запускается Linux а на нем работает Docker. Войти в саму машину можно по команде

```bash
docker-machine ssh default
```

Посмотреть адрес, который обычно 192.168.99.100

```bash
docker-machine ip
```

Изначально на диск виртуальной машины выделяется мало пространства на виртуальном жестком диске. Оно быстро кончается. Самое простое решение удалить виртуальную машину и пересоздать с большим диском.

```bash
docker-machine rm default
docker-machine create -d virtualbox --virtualbox-disk-size "100000" default
```

## Мониторинг ресурсов

```bash
 docker ps -q | xargs  docker stats --no-stream
 docker system df -v
```

## Backup

Резервное копирование вольюмов

```bash
docker run --rm `docker volume list -q | egrep -v '^.{64}$'| awk '{print "-v " $1 ":/mnt/" $1}'` alpine tar -C /mnt -cj . > data-volumes.tar.bz2
```