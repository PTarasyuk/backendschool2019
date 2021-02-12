# Вопросы-ответы

## [jnikish](https://habr.com/ru/users/jnikish/) - 6 мая 2020 в 06:14

Большое спасибо за статью!

Принимал участие в конкурсе (не прошел, не успел задеплоиться).

Подскажите, пожалуйста, по следующим вопросам:

1. Почему не прикрутили NGINX? Не будет ли standalone сервер aiohttp медленнее, чем с NGINX?

2. Почему `citizens` не разбиты на отдельные таблицы для каждого импорта? Намек на это также был в вашем FAQ видео. Еще это позволило бы лочиться только по `citizens.id`, а не по всей выгрузке (`import_id`).

3. Не рассматривался ли вариант получения (от клиента) выгрузки по частям? Возможно ли это сделать средствами aiohttp/python? Например, через [`request.content`](https://docs.aiohttp.org/en/stable/streams.html) или это не поможет читать данные кусками?

4. Не очень понял используется ли connection pool. В коде, при старте, приложение коннектится к postgres, но не очень понятен размер пула (или коннекшн один?).

5. Я c экосистемой python не очень хорошо знаком (в последние пару лет сижу на java), немного смутило, что нет разделения на слои («луковой» архитектуры).

    Транспортный слой (контроллеры, у вас это `handlers`) перемешан со слоем бизнес логики и со слоем БД. Можно было бы как-нибудь разделить, часть вынести в сервисы. Кмк, это уменьшило бы связанность (если говорить о проекте, который будет со временем расти).

6. Так же не используется DI. Для python есть библиотека `injector`, но создается впечатление что в экосистеме python так не принято :)

    Кстати, в Яндексе где-нибудь используется `injector` или своя DI-библиотека для python?

В целом, статья очень поучительная, многие вещи не знал :)

Спасибо!

### [alvassin](https://habr.com/ru/users/alvassin/) - 7 мая 2020 в 09:49

`1. Почему не прикрутили NGINX? Не будет ли standalone сервер aiohttp медленнее, чем с NGINX?`

:   Андрей Светлов рекомендует использовать nginx c aiohttp по следующим причинам:

    * *Nginx может предотвратить множество атак на основе некорректных запросов HTTP, отклонять запросы, методы которых не поддерживаются приложением, запросы со слишком большим body.*

        Это может снять какую-то нагрузку с приложения в production, но выходит за рамки данного задания

    * *Nginx здорово раздает статику.*

        В нашем приложении нет статических файлов.

    * *Nginx позволяет распределить нагрузку по нескольким экземплярам приложения слушающих разные сокеты (upstream module), чтобы загрузить все ядра одного сервера.*

        На мой взгляд проще и эффективнее обслуживать запросы в одном приложении на 1 сокете несколькими форками, чем устанавливать и настраивать для этого отдельное приложение. Я рассказывал как это сделать в разделе «Приложение».

    * *Добавлю от себя: nginx может играть роль балансера, распределяя нагрузку по разным физическим серверам (виртуалкам).*

        По условиям задания, у нас есть только один сервер, где мы можем развернуть приложение, поэтому это тоже выходит за рамки нашей задачи.

    Как вы видите, именно для этого приложения nginx не очень полезен. Но в других случаях (описанных выше), он может очень здорово увеличить эффективность вашего сервиса.

`2. Почему citizens не разбиты на отдельные таблицы для каждого импорта? Намек на это также был в вашем FAQ видео. Еще это позволило бы лочиться только по citizens.id, а не по всей выгрузке (import_id).`

:   Для этой задачи большой разницы нет, можно создавать и отдельные таблицы. Минусы этого подхода: если потребуется реализовать обработчик, возвращающий всех жителей всех выгрузок придется делать очень много UNION-ов, а также при создании таблиц с динамическими названиями их придется экранировать вручную, т.к. PostgreSQL не позволяет использовать аргументы в DDL.

`3. Не рассматривался ли вариант получения (от клиента) выгрузки по частям? Возможно ли это сделать средствами aiohttp/python? Например, через request.content: docs.aiohttp.org/en/stable/streams.html или это не поможет читать данные кусками?`

:   ??? tldr "Получать и обрабатывать json запрос по частям силами aiohttp можно"

        ```python
        from http import HTTPStatus
        
        import ijson
        from aiohttp.web_response import Response
        
        from .base import BaseView
        
        
        class ImportsView(BaseView):
            URL_PATH = '/imports'
        
            async def post(self):
                async for citizen in ijson.items_async(
                    self.request.content,
                    'citizens.item',
                    buf_size=4096
                ):
                    # Обработка жителя
                    print(citizen)
                return Response(status=HTTPStatus.CREATED)
        ```

    Сейчас валидация входных данных в обработчике `POST /imports` происходит двумя Marshmallow схемами: `CitizenSchema` (проверяет конкретного жителя) `ImportSchema` (проверяет связи между жителями и уникальность `citizen_id`) и только затем пишется в базу.

    В случае обработки жителей по одному, вы можете проверить только текущего жителя. Чтобы проверить уникальность его `citizen_id` и его родственные связи в рамках выгрузки придется так или иначе накапливать информацию о уже обработанных жителях выгрузки и откатывать транзакцию, если вы встретите некорректные данные. Можно хранить для каждого жителя только поля `citizen_id` и `relatives`, это сэкономит память, но код будет сложнее, поэтому в этом приложении от этого варианта я отказался.

`4. Не очень понял используется ли connection pool. В коде, при старте, приложение коннектится к postgres, но не очень понятен размер пула (или коннекшн один?).`

:   Используется пул с 10 подключениями ([размер по умолчанию](https://magicstack.github.io/asyncpg/current/api/index.html#asyncpg-api-pool)). При запуске `aiohttp` приложения [`cleanup_ctx`](https://docs.aiohttp.org/en/stable/web_reference.html#aiohttp.web.Application.cleanup_ctx) вызывает [генератор (setup_pg)](https://github.com/alvassin/backendschool2019/blob/master/analyzer/api/app.py#L38), который в свою очередь создает объект [`asyncpgsa.PG`](https://github.com/CanopyTax/asyncpgsa/blob/d228f8d09a8e1019523547f3215517dbe49c7c5f/asyncpgsa/pgsingleton.py#L12) (с пулом соединений) и сохраняет его в `app['pg']`. Базовый обработчик, для удобства предоставляет объект `app['pg']` в виде свойства `pg`. Кол-во подключений, кстати, можно вынести [аргументом `ConfigArgParse`](https://github.com/alvassin/backendschool2019/blob/master/analyzer/api/__main__.py#L38) — настраивать приложение будет очень удобно.

`5. Я c экосистемой python не очень хорошо знаком (в последние пару лет сижу на java), немного смутило, что нет разделения на слои («луковой» архитектуры). Транспортный слой (контроллеры, у вас это handlers) перемешан со слоем бизнес логики и со слоем БД. Можно было бы как-нибудь разделить, часть вынести в сервисы. Кмк, это уменьшило бы связанность (если говорить о проекте, который будет со временем расти).`

:   У нас микросервисы, поэтому на мой взгляд нужно руководствоваться правилом [KISS](https://ru.wikipedia.org/wiki/KISS_(принцип)). Если что — можно быстро написать новый и выкинуть старый.

`6. Так же не используется DI. Для python есть библиотека injector, но создается впечатление что в экосистеме python'a так не принято :) Кстати, в Яндексе где-нибудь используется injector или своя DI-библиотека для python'а?`

:   Не хотелось переусложнять это приложение. В Едадиле мы часто пользуемся модулем [`aiomisc-dependency`](https://pypi.org/project/aiomisc-dependency/). Он здорово выручает, когда в одной программе живет несколько сервисов (REST API, почтовый сервис, еще что-нибудь) и им требуется разделяемый пул ресурсов (например, один `connection pool asyncpg`), хотя этот модуль вполне можно использовать и для создания любых других объектов. Что касается Яндекса — не могу сказать, разработчиков очень много и у всех есть свои любимые инструменты.

## [EuLeEr](https://habr.com/ru/users/EuLeEr/) - 8 мая 2020 в 19:46

С удовольствием прослушал курс бэк-энд разработчика, попал на эту публикацию и решил собрать сервис.

1. make postgres

2. make docker

3. docker run -it \

    -e ANALYZER_PG_URL=postgresql://user:hackme@localhost/analyzer \

    alvassin/backendschool2019 analyzer-db upgrade head

— и это пока последняя команда, потому что в результате:

??? tldr "Ошибка присоединения к Postgres"

    ```text
    Traceback (most recent call last):
      File "/usr/share/python3/app/lib/python3.8/site-packages/sqlalchemy/engine/base.py", line 2285, in _wrap_pool_connect
        return fn()
    ...
      File "/usr/share/python3/app/lib/python3.8/site-packages/psycopg2/__init__.py", line 126, in connect
        conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
    psycopg2.OperationalError: could not connect to server: Connection refused
            Is the server running on host "localhost" (127.0.0.1) and accepting
            TCP/IP connections on port 5432?
    could not connect to server: Cannot assign requested address
            Is the server running on host "localhost" (::1) and accepting
            TCP/IP connections on port 5432?
    
    
    The above exception was the direct cause of the following exception:
    
    Traceback (most recent call last):
      File "/usr/local/bin/analyzer-db", line 11, in <module>
        load_entry_point('analyzer==0.0.1', 'console_scripts', 'analyzer-db')()
    ...
      File "/usr/share/python3/app/lib/python3.8/site-packages/psycopg2/__init__.py", line 126, in connect
        conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
    sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not connect to server: Connection refused
            Is the server running on host "localhost" (127.0.0.1) and accepting
            TCP/IP connections on port 5432?
    could not connect to server: Cannot assign requested address
            Is the server running on host "localhost" (::1) and accepting
            TCP/IP connections on port 5432?
    
    (Background on this error at: http://sqlalche.me/e/e3q8)
    ```

??? tldr "Посмотрел sudo `netstat -tupln`"

    ```text
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 127.0.0.1:46624         0.0.0.0:*               LISTEN      6645/kited      
    tcp        0      0 127.0.0.1:46601         0.0.0.0:*               LISTEN      6929/code       
    tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN      5020/dnsmasq    
    tcp        0      0 10.75.69.1:53           0.0.0.0:*               LISTEN      3251/dnsmasq    
    tcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN      2476/dnsmasq    
    tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      16677/cupsd     
    tcp6       0      0 :::8081                 :::*                    LISTEN      4938/docker-proxy
    tcp6       0      0 fd42:c2ba:2d3b:518e::53 :::*                    LISTEN      3251/dnsmasq    
    tcp6       0      0 fe80::b8bb:caff:fe06:53 :::*                    LISTEN      3251/dnsmasq    
    tcp6       0      0 ::1:631                 :::*                    LISTEN      16677/cupsd     
    tcp6       0      0 :::5432                 :::*                    LISTEN      25651/docker-proxy
    ```
Не видно tcp4 — но ведь все сделано, как Васин рассказал ;-)

Оба контейнера живы, `docker run alvassin/backendschool2019 analyzer-db --help` — работает

Postgres-контейнер тоже

Хоть это вне архитектуры приложения — но как-то потерял несколько часов — не могу понять, почему не устанавливается соединение.

Спасибо заранее

### [EuLeEr](https://habr.com/ru/users/EuLeEr/) - 9 мая 2020 в 09:48

Пообщался в переписке с автором [@alvassin](https://habr.com/ru/users/alvassin/) — если производить запуск 3 команды (на одном хосте) и дополнением -`-network host` — все заработает:

```text
3. docker run -it  --network host \
    -e ANALYZER_PG_URL=postgresql://user:hackme@localhost/analyzer \
    alvassin/backendschool2019 analyzer-db upgrade head
```

Аналогично для запуска приложения:

```text
4. docker run -it -p 8081:8081 --network host \
    -e ANALYZER_PG_URL=postgresql://user:hackme@localhost/analyzer \
    alvassin/backendschool2019
```

```text
5. http://0.0.0.0:8081 в браузере
```

и видна swagger документация сервиса

### [alvassin](https://habr.com/ru/users/alvassin/) - 9 мая 2020 в 15:37

!!! example "Что делает команда `make postgres`"

    ```text
    # Останавливает контейнер analyzer-postgres, если он запущен
    $ docker stop analyzer-postgres || true
    
    # Запускает контейнер с именем analyzer-postgres
    $ docker run --rm --detach --name=analyzer-postgres \
        --env POSTGRES_USER=user \
        --env POSTGRES_PASSWORD=hackme \
        --env POSTGRES_DB=analyzer \
        --publish 5432:5432 postgres
    ```

У вас не получалось подключиться, потому что если при запуске контейнера сеть не указана явно, он создается в **дефолтной bridge-сети**. В такой сети контейнеры могут обращаться друг к другу (и к хосту) [только по ip адресу](https://docs.docker.com/network/bridge/#differences-between-user-defined-bridges-and-the-default-bridge). Кстати, Docker for Mac предоставляет специальное DNS имя [host.docker.internal](https://docs.docker.com/docker-for-mac/networking/#use-cases-and-workarounds) для обращений к хост-машине из контейнеров, что довольно удобно для локальной разработки.

Аргумент `--publish 5432:5432` указывает Docker добавить в `iptables` правило, по которому запросы к хост-машине на порт `5432` отправляются в контейнер с Postgres на порт `5432`.

??? tldr "Поэкспериментируем с подключением к контейнеру с Postgres"

    Подключение с хост-машины на published порт:

    ```text
    $ telnet localhost 5432
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    ```

    Подключение из контейнера с приложением (по IP хост-машины на published-порт):

    ```text
    # Получаем IP хост-машины
    $ docker run -it alvassin/backendschool2019 /bin/bash -c \
        'apt update && apt install -y iproute2 && ip route show | grep default'
    default via 172.17.0.1 dev eth0
    
    # Применяем миграции
    $ docker run -it \
        -e ANALYZER_PG_URL=postgresql://user:hackme@172.17.0.1/analyzer \
        alvassin/backendschool2019 \
        analyzer-db upgrade head
    ```

    Подключение из контейнера с приложением (по `host.docker.internal`, published порт нужен, потому что подключаемся через хост-машину):

    ```text
    $ export HOST=host.docker.internal
    $ docker run -it \
        -e ANALYZER_PG_URL=postgresql://user:hackme@${HOST}/analyzer \
        alvassin/backendschool2019 \
        analyzer-db upgrade head
    ```

    Подключение из контейнера с приложением (по IP контейнера Postgres, published порт в этом случае не нужен):

    ```text
    # Получаем IP
    $ docker inspect \
        -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
        analyzer-postgres
    172.17.0.2
    
    $ docker run -it \
        -e ANALYZER_PG_URL=postgresql://user:hackme@172.17.0.2/analyzer \
        alvassin/backendschool2019 \
        analyzer-db upgrade head
    ```

Если контейнер с приложением запускается в **сети хост-машины** (параметр `--network host`), он [не получает отдельного ip адреса и его сетевой стек не изолируется](https://docs.docker.com/network/host/).

??? tldr "Поэтому такой контейнер имеет доступ к портам хост-машины через localhost."

    ```text
    $ docker run -it \
        --network host \
        -e ANALYZER_PG_URL=postgresql://user:hackme@localhost/analyzer \
        alvassin/backendschool2019 \
        analyzer-db upgrade head
    ```

Третий вариант подружить контейнеры — **создать bridge сеть вручную** и запустить в ней оба контейнера — и Postgres и контейнер с приложением. В отличие от дефолтной bridge-сети, в созданных пользователем bridge-сетях можно [обращаться по имени контейнера](https://docs.docker.com/network/bridge/#differences-between-user-defined-bridges-and-the-default-bridge).

??? tldr "Это бы выглядело следующим образом"

    ```text
    # Создаем сеть
    $ docker network create analyzer-net
    
    # Запускаем контейнер с PostgreSQL в сети analyzer-net
    $ docker run --rm -d \
        --name=analyzer-postgres \
        --network analyzer-net \
        -e POSTGRES_USER=user \
        -e POSTGRES_PASSWORD=hackme \
        -e POSTGRES_DB=analyzer \
        -p 5432:5432 \
        postgres
    
    # Запускаем контейнер с приложением в сети analyzer-net,
    # обращаемся по имени к контейнеру с Postgres
    $ docker run -it --network analyzer-net \
        -e ANALYZER_PG_URL=postgresql://user:hackme@analyzer-postgres/analyzer \
        alvassin/backendschool2019 \
        analyzer-db upgrade head
    ```
