---
title: Настройка этого блога
description: Давно назревала идея обновить когда-то созданную домашнюю страницу. Посмотрев что сейчас имеется решил использовать Jekyll
---

Давно назревала идея обновить когда-то созданную домашнюю страницу. Посмотрев что сейчас имеется решил использовать Jekyll в основном из-за поддержки в GitHub Pages.

Подобрал тему <https://github.com/thinker3197/ink>

Для того чтоб запусить локально использовал Windows Subsystem for Linux (WSL)
```
jekyll serve

#update 2024 command looks like
bundle exec jekyll serve
```
Предварительно установив jekyll и обновив пакеты через `bundler update`

Подключил подсветку синтаксиса native из репозитория <https://github.com/richleland/pygments-css> по статье <https://mycyberuniverse.com/ru/syntax-highlighting-jekyll.html>

Уменьшил размер фоновой картинки сервисом <https://www.richardwestenra.com/repeater/>

Настроил возможность хранения картинок вместе с самим файлом с помощью плагина <https://nhoizey.github.io/jekyll-postfiles/> 

Добавил favicon с помощью сервиса <http://www.xiconeditor.com/>