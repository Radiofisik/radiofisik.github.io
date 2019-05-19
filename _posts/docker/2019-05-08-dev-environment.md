---
title: Настройка dev среды
description: разработка в среде Visual Studio с Docker Toolbox и Docker Compose
---

Использование docker в разработке имеет ряд преимуществ, таких как удобство развертывание, работа в одной среде в проде и деве... 

## Workflow

Типичный процесс работы с использованием docker:
- Разработка в Visual studio
- После отправки через git push в Gitlab, TeamCity забирает изменения и собирает docker образы которые отравляет в реджистри и nuget пакеты, которые отправляет в репозиторий.
- Далее в простейшем случае для запуска проекта используется docker-compose, который на основании файла конфигурации вытягивает образы, создает контейнеры и запускает их с необходимыми связами для работы всего проекта.

## Отладка
### Типовой подход
В плане отладки с контейнерами не все так просто. Visual Studio 2017 по умолчанию поддерживает отладку в контейнере, который создан через механизм интеграции с docker-compose. Работает это так. При добавлении в солюшен интеграции с  docker compose создается проект docker-compose.dcproj который включает файлы docker-compose.yml и docker-compose.override.yml. При запуске проекта Visual Studio 2017 запускает 

```bash
docker-compose  -f "docker-compose.yml" -f "docker-compose.override.yml" -f "docker-compose.vs.debug.g.yml" -p dockercompose1508596578922884945 --no-ansi up -d
```
То есть добавляет свой сгенерированный файл `docker-compose.vs.debug.g.yml` следующего содержания:

```yml
version: '3.4'

services:
  dockertest2019:
    image: dockertest2019:dev
    build:
      target: base
    environment:
      - NUGET_FALLBACK_PACKAGES=/root/.nuget/fallbackpackages;/root/.nuget/fallbackpackages2
    volumes:
      - C:\Users\radiofisik\source\repos\dockertest2019\dockertest2019:/app
      - C:\Users\radiofisik\vsdbg\vs2017u5:/remote_debugger:ro
      - C:\Users\radiofisik\.nuget\packages\:/root/.nuget/packages:ro
      - C:\Program Files\dotnet\sdk\NuGetFallbackFolder:/root/.nuget/fallbackpackages:ro
      - C:\ProgramData\Xamarin\NuGet\:/root/.nuget/fallbackpackages2:ro
    entrypoint: tail -f /dev/null
    labels:
      com.microsoft.visualstudio.debuggee.program: "dotnet"
      com.microsoft.visualstudio.debuggee.arguments: " --additionalProbingPath /root/.nuget/packages --additionalProbingPath /root/.nuget/fallbackpackages --additionalProbingPath /root/.nuget/fallbackpackages2  \"bin/Debug/netcoreapp2.2/dockertest2019.dll\""
      com.microsoft.visualstudio.debuggee.workingdirectory: "/app"
      com.microsoft.visualstudio.debuggee.killprogram: "/bin/sh -c \"if PID=$$(pidof dotnet); then kill $$PID; fi\""
```

По сути переопределяется все содержимое контейнера и запускается удаленная отладка в контейнере. Такой подход имеет ряд недостатков:
- Хорошо работает только если есть только один общий для всех микросервисов docker-compose файл - в противном случае у контейнеров будет проблема с сетевым взаимодействием  - так как команда использует опцию `-p dockercompose1508596578922884945` которая переопределяет `COMPOSE_PROJECT_NAME` на основании которого генерируется имя контейнера, и самое главное имя сети. То есть для нормальной связи контейнеров, в случае если разные микросервисы находятся в разных солюшенах и разных docker-compose файлах придется придумывать ухищрения.
- Отладка в контейнере работает сравнительно медленно.
- При использовании в солюшене проекта *.dcproj появляется проблема при попытке выполнить команду для упаковки nuget пакетов
  ``` dotnet pack Solution.sln ```
При этом попытка исключить проект из упаковки данный проект стандартным для *.csproj образом  через прописывание ``` <IsPackable>false</IsPackable>``` не работает. Для решения приходится упаковывать пакеты по-проектно, что не совсем удобно.
- Иногда возникают проблемы с тем что в результате переопределения в контейнере оказываются   dll без зависимостей. Данная проблема решается добавлением в файл проекта ```<CopyLocalLockFileAssemblies>true</CopyLocalLockFileAssemblies>```
- Все еще более плохо если в команде используются разные инструменты VS, VS Code, Rider

### Альтернативный подход
Типовая архитектура микросервисного приложения предполагает наличие некоторого Reverse Proxy между внешним миром и микросервисами. Соответственно запрос попадает сначала на прокси, затем прокси посылает его на микросервис. В качестве прокси часто используют nginx который также можно разместить в docker контейнере.  В такой среде отладку можно вести локально переопределяя не содержимое контейнера, а сам сервис контейнера - с помощью reverse proxy указывая на локально запущенный под отладкой сервис вместо того что крутится в контейнере. Такой подход позволяет создать скрипты для включения 

  ```bash
  copy ".\Root\debug\01.someService.conf"  ".\Root\services\"
  docker restart reverse_proxy_1
  ```
  и отключения

  ```bash
  del ".\Root\services\01.someService.conf" /q
  docker restart reverse_proxy_1
  ```
  где `01.someService.conf` перенаправляет запросы на `http://192.168.0.52:5007`

  ```nginx
  location ~ ^/api/someService/(.*)$ {
      set $upstream_endpoint http://192.168.0.52:5007;
      rewrite ^/api/someService/(.*) /api/$1 break;
      proxy_pass $upstream_endpoint;
  }
  ```
Остается решить проблему доступа самого сервиса под отладкой ко всей инфраструктуре в контейнера, и главная проблема тут - внутри и наружи разные адреса - это так по крайней мере если используется Docker Toolbox. Проблема решается довольно легко, так как код в контейнере использует переменные конфигурации полученные по следующему пути файл .env + локальные переменные среды - docker.compose.yml (environment секция) - переменные среды контейнера, которые имеют высокий приоритет. В случае же локального запуска переменные берутся из файла appsettings.json и локальных переменных среды. Таким образом осталось обеспечить чтобы код для запуска контейнера и код запуска кода под отладкой использовали разные переменные среды. Об этом далее.

## Кастомизация запуска Docker Toolbox

При запуске Docker Toolbox происходит запуск bash скрипта 

```
C:\Program Files\Docker Toolbox\start.sh
```

В который можно дописать вызов своего скрипта  `./initenv.sh` Для кастомизации запускаемой docker-machine. Пример моего скрипта, который запускает скрипт `inside.sh` внутри docker-machine для ее настройки. Затем устанавливает переменные среды для запуска контейнера (для решения описанной ранее проблемы) и запускает bash в среде где установлены эти переменные среды чтобы можно было запустить тут docker-compose up

```bash
#!/bin/bash

echo "Init my project"
docker-machine ssh default "bash -s"<./inside.sh

cd /d/myProject
unset database_connectionstring
unset redis_connection
export ELASTIC_CONNECTION=http://192.168.99.100:19200

exec bash 

```

Скрипт inside.sh в моем случае устанавливает vm.max_map_count которая необходима для нормальной работы elastic search, настраивает доступ по протоколу http к registry для хранения докер образов.

```bash
echo "inside docker"
sudo -s
sudo sysctl -w vm.max_map_count=262144;
echo "192.168.0.2 registry" >> /etc/hosts
echo '{ "insecure-registries":["registry:5000"] }' > /etc/docker/daemon.json
cat /etc/hosts
```

Таким образом после запуска Docker Toolbox мы оказываемся в нужной папке с нужными переменными среды, но для запуска проекта в реальной жизни приходится использовать команды вида

```bash
docker-compose -f docker-compose.yml -f docker-compose.override-local.yml -f docker-compose.database-local.yml -f docker-compose.mailcatcher.yml up -d
```

и аналогичные с небольшими отличиями в конце. Набирать или копировать постоянно их не удобно и чтобы избежать этого можно использовать alias которые можно настроить и в Windows и это будет работать в docker tools и в git bash. При запуске они ищут файл 

```
"C:\Users\username\.bashrc"
```

туда можно дописать что-код для загрузки alias из другого файла

```bash
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi
```

в самом же `.bash_aliases `

```bash
alias prj='docker-compose -f docker-compose.yml -f docker-compose.database-local.yml -f docker-compose.mailcatcher.yml -f docker-compose.override-local.yml'
```

после этого команды становятся короткими, например:

```bash
prj up -d
prj down
prj pull
...
```

## Полезные алиасы

Для создания небольших проектов удобно использовать 

```bash
githubc=mkdir $1 && cd $1 && touch readme.md && git init && git add . && git commit -m "initial" &&curl -H "Authorization: token YOURTOKEN" -H "Content-Type: application/json" https://api.github.com/user/repos -d "{\"name\": \"$1\"}" && git remote add origin https://github.com/Radiofisik/$1.git && git push --set-upstream origin master

githubu=curl -H "Authorization: token YOURTOKEN" -H "Content-Type: application/json" https://api.github.com/user/repos -d "{\"name\": \"$1\"}" && git remote add origin https://github.com/Radiofisik/$1.git && git push --set-upstream origin master
```

так можно создавать репозитории не заходя на github.

