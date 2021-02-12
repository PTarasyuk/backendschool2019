# Analyzer

Приложение для [практического руководства](https://habr.com/ru/company/yandex/blog/499534/) по разработке бэкенд-сервисов на Python (на основе [вступительного испытания](https://disk.yandex.ru/i/dA9umaGbQdMNLw) в [Школу бэкенд-разработки Яндекса](https://yandex.ru/promo/academy/backend-school/) в 2019 году).

## Что внутри?

Приложение упаковано в Docker-контейнер и разворачивается с помощью Ansible.

Внутри Docker-контейнера доступны две команды:

* `analyzer-db` - утилита для управления состоянием БД
* `analyzer-api` - утилита для запуска REST API сервиса

## Как использовать?

Как применять миграции:

```text
docker run -it \
    -e ANALYZER_PG_URL=postgresql://user:hackme@localhost/analyzer \
    alvassin/backendschool2019 analyzer-db upgrade head
```

Как запустить REST API сервис локально на порту 8081:

```text
docker run -it -p 8081:8081 \
    -e ANALYZER_PG_URL=postgresql://user:hackme@localhost/analyzer \
    alvassin/backendschool2019
```

Все доступные опции запуска любой команды можно получить с помощью аргумента `--help`:

```text
docker run alvassin/backendschool2019 analyzer-db --help
docker run alvassin/backendschool2019 analyzer-api --help
```

Опции для запуска можно указывать как аргументами командной строки, так и переменными окружения с префиксом `ANALYZER` (например: вместо аргумента `--pg-url` можно воспользоваться `ANALYZER_PG_URL`).

### Как развернуть?

Чтобы развернуть и запустить сервис на серверах, добавьте список серверов в файл deploy/hosts.ini (с установленной Ubuntu) и выполните команды:

```text
cd deploy
ansible-playbook -i hosts.ini --user=root deploy.yml
```

## Разработка

### Быстрые команды

* `make` Отобразить список доступных команд
* `make devenv` Создать и настроить виртуальное окружение для разработки
* `make postgres` Поднять Docker-контейнер с PostgreSQL
* `make lint` Проверить синтаксис и стиль кода с помощью [pylama](https://github.com/klen/pylama)
* `make clean` Удалить файлы, созданные модулем [distutils](https://docs.python.org/3/library/distutils.html)
* `make test` Запустить тесты
* `make sdist` Создать [source distribution](https://packaging.python.org/glossary/)
* `make docker` Собрать Docker-образ
* `make upload` Загрузить Docker-образ на hub.docker.com

### Как подготовить окружение для разработки?

```text
make devenv
make postgres
source env/bin/activate
analyzer-db upgrade head
analyzer-api
```

После запуска команд приложение начнет слушать запросы на `0.0.0.0:8081`. Для отладки в PyCharm необходимо запустить `env/bin/analyzer-api`.

### Как запустить тесты локально?

```text
make devenv
make postgres
source env/bin/activate
pytest
```

Для отладки в PyCharm необходимо запустить `env/bin/pytest`.

### Как запустить нагрузочное тестирование?

Для запуска [locust](https://locust.io/) необходимо выполнить следующие команды:

```text
make devenv
source env/bin/activate
locust
```

После этого станет доступен веб-интерфейс по адресу <http://localhost:8089>.

## Ссылки

* [Трансляция с ответами](https://www.youtube.com/watch?v=Bf0liGAahao) на наиболее частые вопросы по тестовым заданиям и Школе.
