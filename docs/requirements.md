# Зависимости

## Основные

### [`aiohttp`](https://docs.aiohttp.org/en/stable/)

HTTP Web сервер и клиент для asyncio ([PEP-3156](https://www.python.org/dev/peps/pep-3156/)). Лекция даёт общее представление о библиотеке, показывает как выполнять клиентские запросы и строить Web сервер с поточной выдачей и Web-сокетами.

### [`aiohttp-apispec`](https://aiohttp-apispec.readthedocs.io/en/latest/)

Создание и документирование REST API на основе `aiohttp`.

Ключевые особенности aiohttp-apispec:

* `docs`, `request_schema`, `match_info_schema`, `querystring_schema`, `form_schema`, `json_schema`, `headers_schema`, `cookies_schema`, декораторы для добавленияподдержки спецификации swagger прямо из коробки;
* промежуточное программное обеспечение (middleware) `validation_middleware`, чтобы разрешить проверку с использованием marshmallow схем из этих декораторов;
* Поддержка **SwaggerUI**.

### [`aiomisc`](https://aiomisc.readthedocs.io/ru/latest/)

Различные утилиты для `asyncio`

### [`alembic`](https://alembic.sqlalchemy.org/en/latest/)

Легкий инструмент миграции базы данных для использования с SQLAlchemy Database Toolkit для Python.

### [`asyncpgsa`](https://asyncpgsa.readthedocs.io/en/latest/)

Оболочка библиотеки python вокруг asyncpg для использования с sqlalchemy.

### [`ConfigArgParse`](https://github.com/bw2/ConfigArgParse)

Расширяет модуль синтаксического анализа командной строки `argparse`, улучшая поддержку файлов конфигурации и переменных среды.

### [`psycopg2-binary`](https://pypi.org/project/psycopg2-binary/)

Скомпилированный wheel-пакет с бинарниками `psycopg` - самого популярного адаптера PostgreSQL для языка программирования Python.

### [`pytz`](https://pythonhosted.org/pytz/)

Переносит базу данных [Olson tz](https://ru.wikipedia.org/wiki/Tz_database) в Python и позволяет точные и кросс-платформенные вычисления часовых поясов.

### [`setproctitle`](https://github.com/dvarrazzo/py-setproctitle)

Позволяет процессу изменять свой заголовок (отображаемый системными инструментами, такими как ps и top).

### [`SQLAlchemy`](https://www.sqlalchemy.org/)

Реализует технологию программирования [ORM](https://ru.wikipedia.org/wiki/ORM) и позволяет описывать структуры баз данных и способы взаимодействия с ними прямо на языке Python.

## Для разработки и тестирования

### [`coverage`](https://coverage.readthedocs.io/en/coverage-5.4/)

Инструмент для измерения покрытия кода программ Python тестами.

### [`Faker`](https://faker.readthedocs.io/en/master/)

Генератор поддельных данных.

### [`locust`](https://locust.io/)

Позволяет задать сценарии нагрузки Python кодом, поддерживающий распределенную нагрузку

### [`pylama`](https://github.com/klen/pylama)

Утилита для проверки python-кода схожая с Flake8, но обладающая рядом улучшений.

### [`pytest`](https://docs.pytest.org/en/stable/)

Основанная на Python среда тестирования, которая используется для написания и выполнения тестовых кодов.

### [`pytest-aiohttp`](https://github.com/aio-libs/pytest-aiohttp/)

Плагин `pytest` для поддержки `aiohttp`

### [`pytest-cov`](https://github.com/pytest-dev/pytest-cov)

Этот плагин создает отчеты о покрытии тестами.

### [`SQLAlchemy-Utils`](https://sqlalchemy-utils.readthedocs.io/en/latest/)

Предоставляет настраиваемые типы данных и различные служебные функции для SQLAlchemy.
