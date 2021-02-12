# Разработка

Я решил дать Python-пакету название `analyzer` и использовать следующую структуру:

```text
analyzer/ .............. Python-пакет с проектом
├─ api/
|  ├─ handlers/ ........ Обработчики
|  ├─ __init__.py
|  ├─ __main.py ........ Утилита для запуска REST API
|  ├─ app.py ........... Описание aiohttp-приложения
|  ├─ middleware.py
|  ├─ payloads.py ...... Инструкции по упаковке body aiohttp-ответов
|  └─ schema.py ........ Схемы валидации запросов и ответов REST API
├─ db/
|  ├─ alembic/ ......... Миграции и всё, что с ними связано
|  ├─ __init__.py 
|  ├─ __main__.py ...... Утилита для управления состоянием БД
|  └─ schema.py ........ Описание схемы БД
├─ utils/ .............. Утилиты
└─ __init__.py
```

В файле `analyzer/__init__.py` я разместил общую информацию о пакете: описание ([docstring](https://www.python.org/dev/peps/pep-0257/#what-is-a-docstring)), версию, лицензию, контакты разработчиков.

??? tldr "Ее можно посмотреть встроенной командой `help`"

    ```bash
    $ python
    >>> import analyzer
    >>> help(analyzer)
    
    Help on package analyzer:
    
    NAME
        analyzer
    
    DESCRIPTION
        Сервис с REST API, анализирующий рынок для промоакций.
    
    PACKAGE CONTENTS
        api (package)
        db (package)
        utils (package)
    
    DATA
        __all__ = ('__author__', '__email__', '__license__', '__maintainer__',...
        __email__ = 'alvassin@yandex.ru'
        __license__ = 'MIT'
        __maintainer__ = 'Alexander Vasin'
    
    VERSION
        0.0.1
    
    AUTHOR
        Alexander Vasin
    
    FILE
        /Users/alvassin/Work/backendschool2019/analyzer/__init__.py
    ```

Пакет имеет две входных точки — REST API-сервис (`analyzer/api/__main__.py`) и утилита управления состоянием БД (`analyzer/db/__main__.py`). Файлы называются `__main__.py` неспроста:

1. Во-первых, такое название привлекает внимание, по нему понятно, что файл является входной точкой.
2. Во-вторых, благодаря этому подходу к входным точкам [можно обращаться с помощью команды `python -m`](https://docs.python.org/3/library/__main__.html#module-__main__).

```bash
# REST API
$ python -m analyzer.api --help

# Утилита управления состоянием БД
$ python -m analyzer.db --help
```

## Почему нужно начать с `setup.py`?

Забегая вперед, подумаем, как можно распространять приложение: оно может быть упаковано в zip- (а также wheel/egg-) архив, rpm-пакет, pkg-файл для macOS и установлено на удаленный компьютер, в виртуальную машину, MacBook или Docker-контейнер.

Главная цель файла [`setup.py`](https://docs.python.org/3/distutils/setupscript.html) — описать пакет с приложением для [`distutils`](https://docs.python.org/3/library/distutils.html)/[`setuptools`](https://setuptools.readthedocs.io/en/latest/).

В файле необходимо указать общую информацию о пакете (название, версию, автора и т. д.), но также в нем можно указать требуемые для работы модули, «экстра»-зависимости (например для тестирования), точки входа (например, исполняемые команды) и требования к интерпретатору.

Плагины `setuptools` позволяют собирать из описанного пакета артефакт. Есть встроенные плагины: zip, egg, rpm, macOS pkg. Остальные плагины распространяются через PyPI: [`wheel`](https://pypi.org/project/wheel), [`xar`](https://pypi.org/project/xar), [`pex`](https://pypi.org/project/pex).

В сухом остатке, описав один файл, мы получаем огромные возможности. Именно поэтому разработку нового проекта нужно начинать с `setup.py`.

В функции `setup()` зависимые модули указываются списком:

```python
setup(..., install_requires=["aiohttp", "SQLAlchemy"])
```

Но я описал зависимости в отдельных файлах `requirements.txt` и `requirements.dev.txt`, содержимое которых используется в `setup.py`. Мне это кажется более гибким, плюс тут есть секрет: впоследствии это позволит собирать Docker-образ быстрее. Зависимости будут ставиться отдельным шагом до установки самого приложения, а при пересборке Docker-контейнера попадать в кеш.

Чтобы `setup.py` смог прочитать зависимости из файлов `requirements.txt` и `requirements.dev.txt`, написана функция:

```python
def load_requirements(fname: str) -> list:
    requirements = []
    with open(fname, 'r') as fp:
        for req in parse_requirements(fp.read()):
            extras = '[{}]'.format(','.join(req.extras)) if req.extras else ''
            requirements.append(
                '{}{}{}'.format(req.name, extras, req.specifier)
            )
    return requirements
```

Стоит отметить, что `setuptools` при сборке source distribution по умолчанию включает в сборку только файлы `.py`, `.c`, `.cpp` и `.h`. Чтобы файлы с зависимостями `requirements.txt` и `requirements.dev.txt` попали в пакет, их необходимо явно указать в файле [`MANIFEST.in`](https://packaging.python.org/guides/using-manifest-in/).

??? tldr "`setup.py` целиком"

    ```python
    import os
    from importlib.machinery import SourceFileLoader
    
    from pkg_resources import parse_requirements
    from setuptools import find_packages, setup
    
    module_name = 'analyzer'
    
    # Возможно, модуль еще не установлен (или установлена другая версия), поэтому
    # необходимо загружать __init__.py с помощью machinery.
    module = SourceFileLoader(
        module_name, os.path.join(module_name, '__init__.py')
    ).load_module()
    
    
    def load_requirements(fname: str) -> list:
        requirements = []
        with open(fname, 'r') as fp:
            for req in parse_requirements(fp.read()):
                extras = '[{}]'.format(','.join(req.extras)) if req.extras else ''
                requirements.append(
                    '{}{}{}'.format(req.name, extras, req.specifier)
                )
        return requirements
    
    
    setup(
        name=module_name,
        version=module.__version__,
        author=module.__author__,
        author_email=module.__email__,
        license=module.__license__,
        description=module.__doc__,
        long_description=open('README.rst').read(),
        url='https://github.com/alvassin/backendschool2019',
        platforms='all',
        classifiers=[
            'Intended Audience :: Developers',
            'Natural Language :: Russian',
            'Operating System :: MacOS',
            'Operating System :: POSIX',
            'Programming Language :: Python',
            'Programming Language :: Python :: 3',
            'Programming Language :: Python :: 3.8',
            'Programming Language :: Python :: Implementation :: CPython'
        ],
        python_requires='>=3.8',
        packages=find_packages(exclude=['tests']),
        install_requires=load_requirements('requirements.txt'),
        extras_require={'dev': load_requirements('requirements.dev.txt')},
        entry_points={
            'console_scripts': [
                # f-strings в setup.py не используются из-за соображений
                # совместимости.
                # Несмотря на то, что этот пакет требует Python 3.8, технически
                # source distribution для него может собираться с помощью более
                # ранних версий Python. Не стоит лишать пользователей этой
                # возможности.
                '{0}-api = {0}.api.__main__:main'.format(module_name),
                '{0}-db = {0}.db.__main__:main'.format(module_name)
            ]
        },
        include_package_data=True
    )
    ```

Установить проект в режиме разработки можно следующей командой (в editable-режиме Python не установит пакет целиком в папку `site-packages`, а только создаст ссылки, поэтому любые изменения, вносимые в файлы пакета, будут видны сразу):

```bash
# Установить пакет с обычными и extra-зависимостями "dev"
pip install -e '.[dev]'

# Установить пакет только с обычными зависимостями
pip install -e .
```

## Как указать версии зависимостей?

Здорово, когда разработчики активно занимаются своими пакетами — в них активнее исправляются ошибки, появляется новая функциональность и можно быстрее получить обратную связь. Но иногда изменения в зависимых библиотеках не имеют обратной совместимости и могут привести к ошибкам в вашем приложении, если не подумать об этом заранее.

Для каждого зависимого пакета можно указать определенную версию, например `aiohttp==3.6.2`. Тогда приложение будет гарантированно собираться именно с теми версиями зависимых библиотек, с которыми оно было протестировано. Но у этого подхода есть и недостаток — если разработчики исправят критичный баг в зависимом пакете, не влияющий на обратную совместимость, в приложение это исправление не попадет.

Существует подход к версионированию [Semantic Versioning](https://semver.org/), который предлагает представлять версию в формате `MAJOR.MINOR.PATCH`:

* `MAJOR` — увеличивается при добавлении обратно несовместимых изменений;
* `MINOR` — увеличивается при добавлении новой функциональности с поддержкой обратной совместимости;
* `PATCH` — увеличивается при добавлении исправлений багов с поддержкой обратной совместимости.

Если зависимый пакет следует этому подходу (о чем авторы обычно сообщают в файлах `README` или `CHANGELOG`), то достаточно зафиксировать значения `MAJOR`, `MINOR` и ограничить минимальное значение для PATCH-версии: `>= MAJOR.MINOR.PATCH`, `== MAJOR.MINOR.*`.

Такое требование можно реализовать с помощью оператора [`~=`](https://www.python.org/dev/peps/pep-0440/#compatible-release). Например, `aiohttp~=3.6.2` позволит PIP установить для `aiohttp` версию 3.6.3, но не 3.7.

Если указать интервал версий зависимостей, это даст еще одно преимущество — не будет конфликтов версий между зависимыми библиотеками.

Если вы разрабатываете библиотеку, которая требует другой пакет-зависимость, то разрешите для него не одну определенную версию, а интервал. Тогда потребителям вашей библиотеки будет намного легче ее использовать (вдруг их приложение требует этот же пакет-зависимость, но уже другой версии).

Semantic Versioning — лишь соглашение между авторами и потребителями пакетов. Оно не гарантирует, что авторы пишут код без багов и не могут допустить ошибку в новой версии своего пакета.

## База данных

### Проектируем схему

В описании обработчика `POST /imports` приведен пример выгрузки с информацией о жителях:

??? example "Пример выгрузки"

    ```json
    {
        "citizens": [
            {
                "citizen_id": 1,
                "town": "Москва",
                "street": "Льва Толстого",
                "building": "16к7стр5",
                "apartment": 7,
                "name": "Иванов Иван Иванович",
                "birth_date": "26.12.1986",
                "gender": "male",
                "relatives": [2]
            },
            {
                "citizen_id": 2,
                "town": "Москва",
                "street": "Льва Толстого",
                "building": "16к7стр5",
                "apartment": 7,
                "name": "Иванов Сергей Иванович",
                "birth_date": "01.04.1997",
                "gender": "male",
                "relatives": [1]
            },
            {
                "citizen_id": 3,
                "town": "Керчь",
                "street": "Иосифа Бродского",
                "building": "2",
                "apartment": 11,
                "name": "Романова Мария Леонидовна",
                "birth_date": "23.11.1986",
                "gender": "female",
                "relatives": []
            },
            ...
        ]
    }
    ```

Первой мыслью было хранить всю информацию о жителе в одной таблице `citizens`, где родственные связи были бы представлены полем `relatives` в **виде списка целых чисел**.

??? tldr "Но у этого способа есть ряд недостатков"

    1. В обработчике `GET /imports/$import_id/citizens/birthdays` для получения месяцев, на которые приходятся дни рождения родственников, потребуется выполнить слияние таблицы `citizens` с самой собой. Для этого будет необходимо развернуть список с идентификаторами родственников `relatives` с помощью функции `UNNEST`.
        
        Такой запрос будет выполняться сравнительно медленно, и **обработчик не уложится в 10-секундный таймаут**:

        ```sql
        SELECT 
            relations.citizen_id, 
            relations.relative_id, 
            date_part('month', relatives.birth_date) as relative_birth_month
        FROM (
            SELECT
                citizens.import_id, 
                citizens.citizen_id,
                UNNEST(citizens.relatives) as relative_id
            FROM citizens
            WHERE import_id = 1
        ) as relations
        INNER JOIN citizens as relatives ON
            relations.import_id = relatives.import_id AND
            relations.relative_id = relatives.citizen_id
        ```

    2. В таком подходе **целостность данных в поле `relatives` не обеспечивается PostgreSQL**, а контролируется приложением: технически в список `relatives` можно добавить любое целое число, в том числе идентификатор несуществующего жителя. Ошибка в коде или человеческий фактор (редактирование записей напрямую в БД администратором) обязательно рано или поздно приведут к несогласованному состоянию данных.

Далее, я решил привести все требуемые для работы данные к [третьей нормальной форме](https://ru.wikipedia.org/wiki/Третья_нормальная_форма), и получилась следующая структура:

![Третья нормальная форма](/img/dipfpxxw142kafiect0yhrtt4uo.png)

1. Таблица **`imports`** состоит из автоматически инкрементируемого столбца `import_id`. Он нужен для создания проверки по внешнему ключу в таблице `citizens`.

2. В таблице **`citizens`** хранятся **скалярные данные** о жителе (все поля за исключением информации о родственных связях).

    В качестве первичного ключа используется пара (`import_id`, `citizen_id`), гарантирующая уникальность жителей `citizen_id` в рамках `import_id`.

    Внешний ключ `citizens.import_id -> imports.import_id` гарантирует, что поле `citizens.import_id` будет содержать только существующие выгрузки.

3. Таблица **`relations`** содержит информацию о **родственных связях**.

    **Одна родственная связь представлена двумя записями** (от жителя к родственнику и обратно): эта избыточность позволяет использовать более простое условие при слиянии таблиц `citizens` и `relations` и получать информацию более эффективно.

    Первичный ключ состоит из столбцов (`import_id`, `citizen_id`, `relative_id`) и гарантирует, что в рамках одной выгрузки `import_id` у жителя `citizen_id` будут родственники c уникальными `relative_id`.

    Также в таблице используются два составных внешних ключа: `(relations.import_id, relations.citizen_id) -> (citizens.import_id, citizens.citizen_id)` и `(relations.import_id, relations.relative_id) -> (citizens.import_id, citizens.citizen_id)`, гарантирующие, что в таблице будут указаны существующие житель `citizen_id` и родственник `relative_id` из одной выгрузки.

Такая структура обеспечивает **целостность данных средствами PostgreSQL**, позволяет **эффективно получать жителей с родственниками** из базы данных, но **подвержена состоянию гонки** во время обновления информации о жителях конкурентными запросами (подробнее рассмотрим при реализации обработчика `PATCH`).

### Описываем схему в SQLAlchemy

В [лекции 5](https://habr.com/ru/company/yandex/blog/498856/#5) я рассказывал, что для создания запросов с помощью SQLAlchemy необходимо описать схему базы данных с помощью специальных объектов: таблицы описываются с помощью `sqlalchemy.Table` и привязываются к реестру `sqlalchemy.MetaData`, который хранит всю метаинформацию о базе данных. К слову, реестр `MetaData` способен не только хранить описанную в Python метаинформацию, но и представлять реальное состояние базы данных в виде объектов SQLAlchemy.

Эта возможность в том числе позволяет Alembic сравнивать состояния и генерировать код миграций автоматически.

Кстати, у каждой базы данных своя схема именования constraints по умолчанию. Чтобы вы не тратили время на именование новых constraints или на воспоминания/поиски того, как назван constraint, который вы собираетесь удалить, SQLAlchemy предлагает использовать шаблоны именования [naming conventions](https://docs.sqlalchemy.org/en/13/core/constraints.html#configuring-constraint-naming-conventions). Их можно определить в реестре `MetaData`.

??? tldr "Создаем реестр `MetaData` и передаем в него шаблоны именования"

    ```python
    # analyzer/db/schema.py
    from sqlalchemy import MetaData
    
    convention = {
        'all_column_names': lambda constraint, table: '_'.join([
            column.name for column in constraint.columns.values()
        ]),
    
        # Именование индексов
        'ix': 'ix__%(table_name)s__%(all_column_names)s',
    
        # Именование уникальных индексов
        'uq': 'uq__%(table_name)s__%(all_column_names)s',
    
        # Именование CHECK-constraint-ов
        'ck': 'ck__%(table_name)s__%(constraint_name)s',
    
        # Именование внешних ключей
        'fk': 'fk__%(table_name)s__%(all_column_names)s__%(referred_table_name)s',
    
        # Именование первичных ключей
        'pk': 'pk__%(table_name)s'
    }
    metadata = MetaData(naming_convention=convention)
    ```

Если указать шаблоны именования, Alembic воспользуется ими во время автоматической генерации миграций и будет называть все constraints в соответствии с ними. В дальнейшем созданный реестр `MetaData` потребуется для описания таблиц:

??? tldr "Описываем схему базы данных объектами SQLAlchemy"

    ```python
    # analyzer/db/schema.py
    from enum import Enum, unique
    
    from sqlalchemy import (
        Column, Date, Enum as PgEnum, ForeignKey, ForeignKeyConstraint, Integer,
        String, Table
    )
    
    
    @unique
    class Gender(Enum):
        female = 'female'
        male = 'male'
    
    
    imports_table = Table(
        'imports',
        metadata,
        Column('import_id', Integer, primary_key=True)
    )
    
    citizens_table = Table(
        'citizens',
        metadata,
        Column('import_id', Integer, ForeignKey('imports.import_id'),
               primary_key=True),
        Column('citizen_id', Integer, primary_key=True),
        Column('town', String, nullable=False, index=True),
        Column('street', String, nullable=False),
        Column('building', String, nullable=False),
        Column('apartment', Integer, nullable=False),
        Column('name', String, nullable=False),
        Column('birth_date', Date, nullable=False),
        Column('gender', PgEnum(Gender, name='gender'), nullable=False),
    )
    
    relations_table = Table(
        'relations',
        metadata,
        Column('import_id', Integer, primary_key=True),
        Column('citizen_id', Integer, primary_key=True),
        Column('relative_id', Integer, primary_key=True),
        ForeignKeyConstraint(
            ('import_id', 'citizen_id'),
            ('citizens.import_id', 'citizens.citizen_id')
        ),
        ForeignKeyConstraint(
            ('import_id', 'relative_id'),
            ('citizens.import_id', 'citizens.citizen_id')
        ),
    )
    ```

### Настраиваем Alembic

Когда схема базы данных описана, необходимо сгенерировать миграции, но для этого сначала нужно настроить Alembic, об этом тоже рассказывается в [лекции 5](https://habr.com/ru/company/yandex/blog/498856/#5).

Чтобы воспользоваться командой `alembic`, необходимо выполнить следующие шаги:

1. Установить пакет: `pip install alembic`

2. Инициализировать Alembic: `cd analyzer && alembic init db/alembic`.

    Эта команда создаст файл конфигурации `analyzer/alembic.ini` и папку `analyzer/db/alembic` со следующим содержимым:

    * `env.py` — вызывается каждый раз при запуске Alembic. Подключает в Alembic реестр `sqlalchemy.MetaData` с описанием желаемого состояния БД и содержит инструкции по запуску миграций.
    * `script.py.mako` — шаблон, на основе которого генерируются миграции.
    * `versions` — папка, в которой Alembic будет искать (и генерировать) миграции.

3. Указать адрес базы данных в файле `alembic.ini`:

    ```ini
    ; analyzer/alembic.ini
    [alembic]
    sqlalchemy.url = postgresql://user:hackme@localhost/analyzer
    ```

4. Указать описание желаемого состояния базы данных (реестр `sqlalchemy.MetaData`), чтобы Alembic мог генерировать миграции автоматически:

    ```python
    # analyzer/db/alembic/env.py
    from analyzer.db import schema
    target_metadata = schema.metadata
    ```

Alembic настроен и им уже можно пользоваться, но в нашем случае такая конфигурация имеет ряд недостатков:

1. Утилита `alembic` ищет `alembic.ini` в текущей рабочей директории. Путь к `alembic.ini` можно указать аргументом командной строки, но это неудобно: хочется иметь возможность вызывать команду из любой папки без дополнительных параметров.

2. Чтобы настроить Alembic на работу с определенной базой данных, требуется менять файл `alembic.ini`. Гораздо удобнее было бы указать настройки БД переменной окружения и/или аргументом командной строки, например `--pg-url`.

3. Название утилиты `alembic` не очень хорошо коррелирует с названием нашего сервиса (а пользователь фактически может вообще не владеть Python и ничего не знать об Alembic). Конечному пользователю было бы намного удобнее, если бы все исполняемые команды сервиса имели общий префикс, например `analyzer-*`.

Эти проблемы решаются с помощью небольшой обертки `analyzer/db/__main__.py`:

* Для обработки аргументов командной строки Alembic использует стандартный модуль [`argparse`](https://docs.python.org/3/library/argparse.html). Он позволяет добавить необязательный аргумент `--pg-url` со значением по умолчанию из переменной окружения `ANALYZER_PG_URL`.

    ??? example "Код"

        ```python
        import os
        from alembic.config import CommandLine, Config
        from analyzer.utils.pg import DEFAULT_PG_URL
        
        
        def main():
            alembic = CommandLine()
            alembic.parser.add_argument(
                '--pg-url', default=os.getenv('ANALYZER_PG_URL', DEFAULT_PG_URL),
                help='Database URL [env var: ANALYZER_PG_URL]'
            )
            options = alembic.parser.parse_args()
        
            # Создаем объект конфигурации Alembic
            config = Config(file_=options.config, ini_section=options.name,
                            cmd_opts=options)
        
            # Меняем значение sqlalchemy.url из конфига Alembic
            config.set_main_option('sqlalchemy.url', options.pg_url)
        
            # Запускаем команду alembic
            exit(alembic.run_cmd(config, options))
        
        
        if __name__ == '__main__':
            main()
        ```

* Путь до файла `alembic.ini` можно рассчитывать относительно расположения исполняемого файла, а не текущей рабочей директории пользователя.

    ??? example "Код"

        ```python
        import os
        from alembic.config import CommandLine, Config
        from pathlib import Path
        
        
        PROJECT_PATH = Path(__file__).parent.parent.resolve()
        
        
        def main():
            alembic = CommandLine()
            options = alembic.parser.parse_args()
        
            # Если указан относительный путь (alembic.ini), добавляем в начало
            # абсолютный путь до приложения
            if not os.path.isabs(options.config):
                options.config = os.path.join(PROJECT_PATH, options.config)
        
            # Создаем объект конфигурации Alembic
            config = Config(file_=options.config, ini_section=options.name,
                            cmd_opts=options)
        
            # Подменяем путь до папки с alembic на абсолютный (требуется, чтобы alembic
            # мог найти env.py, шаблон для генерации миграций и сами миграции)
            alembic_location = config.get_main_option('script_location')
            if not os.path.isabs(alembic_location):
                config.set_main_option('script_location',
                                       os.path.join(PROJECT_PATH, alembic_location))
        
            # Запускаем команду alembic
            exit(alembic.run_cmd(config, options))
        
        
        if __name__ == '__main__':
            main()
        ```

Когда утилита `/analyzer/db/__main__.py` для управления состоянием БД готова, ее можно зарегистрировать в `setup.py` как исполняемую команду с понятным конечному пользователю названием, например `analyzer-db`:

??? example "Регистрация исполняемой команды в `setup.py`"

    ```python
    from setuptools import setup
    
    setup(..., entry_points{
        'console_scripts': [
            'analyzer-db = analyzer.db.__main__:main'
        ]
    })
    ```

После переустановки модуля будет сгенерирован файл `env/bin/analyzer-db` и команда `analyzer-db` станет доступной:

```bash
$ pip install -e '.[dev]'
```

### Генерируем миграции

Чтобы сгенерировать миграции, требуется два состояния: желаемое (которое мы описали объектами SQLAlchemy) и реальное (база данных, в нашем случае пустая).

Я решил, что проще всего поднять Postgres с помощью Docker и для удобства добавил команду `make postgres`, запускающую в фоновом режиме контейнер с PostgreSQL на 5432 порту:

??? tldr "Поднимаем PostgreSQL и генерируем миграцию"

    ```bash
    $ make postgres
    ...
    $ analyzer-db revision --message="Initial" --autogenerate
    INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
    INFO  [alembic.runtime.migration] Will assume transactional DDL.
    INFO  [alembic.autogenerate.compare] Detected added table 'imports'
    INFO  [alembic.autogenerate.compare] Detected added table 'citizens'
    INFO  [alembic.autogenerate.compare] Detected added index 'ix__citizens__town' on '['town']'
    INFO  [alembic.autogenerate.compare] Detected added table 'relations'
      Generating /Users/alvassin/Work/backendschool2019/analyzer/db/alembic/versions/d5f704ed4610_initial.py ...  done
    ```

Alembic в целом хорошо справляется с рутинной работой генерации миграций, но я хотел бы обратить внимание на следующее:

* Пользовательские типы данных, указанные в создаваемых таблицах, создаются автоматически (в нашем случае — `gender`), но код для их удаления в `downgrade` не генерируется. Если применить, откатить и потом еще раз применить миграцию, это вызовет ошибку, так как указанный тип данных уже существует.

    ??? tldr "Удаляем тип данных `gender` в методе `downgrade`"

        ```python
        from alembic import op
        from sqlalchemy import Column, Enum
        
        
        GenderType = Enum('female', 'male', name='gender')
        
        
        def upgrade():
            ...
            # При создании таблицы тип данных GenderType будет создан автоматически
            op.create_table('citizens', ...,
                            Column('gender', GenderType, nullable=False))
            ...
        
        
        def downgrade():
            op.drop_table('citizens')
        
            # После удаления таблицы тип данных необходимо удалить
            GenderType.drop(op.get_bind())
        ```

* В методе `downgrade` некоторые действия иногда можно убрать (если мы удаляем таблицу целиком, можно не удалять ее индексы отдельно):

    ??? example "Например"

        ```python
        def downgrade():
            op.drop_table('relations')
        
            # Следующим шагом мы удаляем таблицу citizens, индекс будет удален автоматически
            # эту строчку можно удалить
            op.drop_index(op.f('ix__citizens__town'), table_name='citizens')
            op.drop_table('citizens')
            op.drop_table('imports')
        ```

Когда миграция исправлена и готова, применим ее:

```bash
$ analyzer-db upgrade head
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> d5f704ed4610, Initial
```

## Приложение

Прежде чем приступить к созданию обработчиков, необходимо сконфигурировать приложение aiohttp.

??? example "Если посмотреть aiohttp quickstart, можно написать приблизительно такой код"

    ```python
    import logging
    
    from aiohttp import web
    
    
    def main():
        # Настраиваем логирование
        logging.basicConfig(level=logging.DEBUG)
    
        # Создаем приложение
        app = web.Application()
    
        # Регистрируем обработчики
        app.router.add_route(...)
    
        # Запускаем приложение
        web.run_app(app)
    ```

Этот код вызывает ряд вопросов и имеет ряд недостатков:

* **Как конфигурировать приложение?** Как минимум, необходимо указать хост и порт для подключения клиентов, а также информацию для подключения к базе данных.

    Мне очень нравится решать эту задачу с помощью модуля [`ConfigArgParse`](https://pypi.org/project/ConfigArgParse/): он расширяет стандартный [`argparse`](https://docs.python.org/3/library/argparse.html) и позволяет использовать для конфигурации аргументы командной строки, переменные окружения (незаменимые для конфигурации Docker-контейнеров) и даже файлы конфигурации (а также совмещать эти способы). C помощью `ConfigArgParse` также можно валидировать значения параметров конфигурации приложения.

    ??? example "Пример обработки параметров с помощью `ConfigArgParse`"

        ```python
        from aiohttp import web
        from configargparse import ArgumentParser, ArgumentDefaultsHelpFormatter
        
        from analyzer.utils.argparse import positive_int
        
        
        parser = ArgumentParser(
            # Парсер будет искать переменные окружения с префиксом ANALYZER_,
            # например ANALYZER_API_ADDRESS и ANALYZER_API_PORT
            auto_env_var_prefix='ANALYZER_',
        
            # Покажет значения параметров по умолчанию
            formatter_class=ArgumentDefaultsHelpFormatter
        )
        
        parser.add_argument('--api-address', default='0.0.0.0',
                            help='IPv4/IPv6 address API server would listen on')
        
        # Разрешает только целые числа больше нуля
        parser.add_argument('--api-port', type=positive_int, default=8081,
                            help='TCP port API server would listen on')
        
        
        def main():
            # Получаем параметры конфигурации, которые можно передать как аргументами
            # командной строки, так и переменными окружения
            args = parser.parse_args()
        
            # Запускаем приложение на указанном порту и адресе
            app = web.Application()
            web.run_app(app, host=args.api_address, port=args.api_port)
        
        
        if __name__ == '__main__':
            main()
        ```

    Кстати, `ConfigArgParse`, как и `argparse`, умеет генерировать подсказку по запуску команды с описанием всех аргументов (необходимо позвать команду с аргументом `-h` или `--help`). Это невероятно облегчает жизнь пользователям вашего ПО:

    ??? example "Например"

        ```bash
        $ python __main__.py --help
        usage: __main__.py [-h] [--api-address API_ADDRESS] [--api-port API_PORT]
        
        If an arg is specified in more than one place, then commandline values override environment variables which override defaults.
        
        optional arguments:
          -h, --help            show this help message and exit
          --api-address API_ADDRESS
                                IPv4/IPv6 address API server would listen on [env var: ANALYZER_API_ADDRESS] (default: 0.0.0.0)
          --api-port API_PORT   TCP port API server would listen on [env var: ANALYZER_API_PORT] (default: 8081)
        ```

* После получения переменные окружения больше не нужны и даже могут представлять опасность — например, они могут случайно «утечь» с отображением информации об ошибке. Злоумышленники в первую очередь будут пытаться получить информацию об окружении, поэтому **очистка переменных окружения считается хорошим тоном**.

    Можно было бы воспользоваться `os.environ.clear()`, но Python позволяет управлять поведением модулей стандартной библиотеки с помощью многочисленных переменных окружения (например, вдруг потребуется включить режим отладки `asyncio`?), поэтому разумнее очищать переменные окружения по префиксу приложения, указанного в `ConfigArgParser`.

    ??? example "Пример"

        ```python
        import os
        from typing import Callable
        from configargparse import ArgumentParser
        from yarl import URL
        
        from analyzer.api.app import create_app
        from analyzer.utils.pg import DEFAULT_PG_URL
        
        
        ENV_VAR_PREFIX = 'ANALYZER_'
        
        parser = ArgumentParser(auto_env_var_prefix=ENV_VAR_PREFIX)
        parser.add_argument('--pg-url', type=URL, default=URL(DEFAULT_PG_URL),
                            help='URL to use to connect to the database')
        
        
        def clear_environ(rule: Callable):
            """
            Очищает переменные окружения, переменные для очистки определяет переданная
            функция rule
            """
            # Ключи из os.environ копируются в новый tuple, чтобы не менять объект
            # os.environ во время итерации
            for name in filter(rule, tuple(os.environ)):
                os.environ.pop(name)
        
        
        def main():
            # Получаем аргументы
            args = parser.parse_args()
        
            # Очищаем переменные окружения по префиксу ANALYZER_
            clear_environ(lambda i: i.startswith(ENV_VAR_PREFIX))
        
            # Запускаем приложение
            app = create_app(args)
            ...
        
        
        if __name__ == '__main__':
            main()
        ```

* **Запись логов в `stderr`/файл в основном потоке блокирует цикл событий.**

    В [лекции 9](https://habr.com/ru/company/yandex/blog/498856/#9) рассказывается, что по умолчанию `logging.basicConfig()` настраивает [запись логов в `stderr`](https://github.com/python/cpython/blob/950c6795aa0ffa85e103a13e7a04e08cb34c66ad/Lib/logging/__init__.py#L1907).

    Чтобы логирование не мешало эффективной работе асинхронного приложения, необходимо выполнять запись логов в отдельном потоке. Для этого можно воспользоваться готовым методом из модуля `aiomisc`.

    ??? tldr "Настраиваем логирование с помощью `aiomisc`"

        ```python
        import logging
        
        from aiomisc.log import basic_config
        
        
        basic_config(logging.DEBUG, buffered=True)
        ```

* **Как масштабировать приложение, если одного процесса станет недостаточно** для обслуживания входящего трафика? Можно сначала аллоцировать сокет, затем с помощью [`fork`](https://ru.wikipedia.org/wiki/Fork) создать несколько новых отдельных процессов, и соединения на сокете будут распределяться между ними механизмами ядра (конечно, под Windows это не работает).

    ??? example "Пример"

        ```python
        import os
        from sys import argv
        
        import forklib
        from aiohttp.web import Application, run_app
        from aiomisc import bind_socket
        from setproctitle import setproctitle
        
        
        def main():
            sock = bind_socket(address='0.0.0.0', port=8081, proto_name='http')
            setproctitle(f'[Master] {os.path.basename(argv[0])}')
        
            def worker():
                setproctitle(f'[Worker] {os.path.basename(argv[0])}')
                app = Application()
                run_app(app, sock=sock)
        
            forklib.fork(os.cpu_count(), worker, auto_restart=True)
        
        
        if __name__ == '__main__':
            main()
        ```

* Требуется ли приложению обращаться или аллоцировать какие-либо ресурсы во время работы? Если нет, по соображениям безопасности все ресурсы (в нашем случае — сокет для подключения клиентов) можно аллоцировать на старте, а затем сменить пользователя на `nobody`. Он обладает ограниченным набором привилегий — это здорово усложнит жизнь злоумышленникам.

    ??? example "Пример"

        ```python
        import os
        import pwd
        
        from aiohttp.web import run_app
        from aiomisc import bind_socket
        
        from analyzer.api.app import create_app
        
        
        def main():
            # Аллоцируем сокет
            sock = bind_socket(address='0.0.0.0', port=8085, proto_name='http')
        
            user = pwd.getpwnam('nobody')
            os.setgid(user.pw_gid)
            os.setuid(user.pw_uid)
        
            app = create_app(...)
            run_app(app, sock=sock)
        
        
        if __name__ == '__main__':
            main()
        ```

* В конце концов я решил вынести создание приложения в отдельную параметризуемую функцию `create_app`, чтобы можно было легко создавать идентичные приложения для тестирования.

### Сериализация данных

Все успешные ответы обработчиков будем возвращать в формате JSON. Информацию об ошибках клиентам тоже было бы удобно получать в сериализованном виде (например, чтобы увидеть, какие поля не прошли валидацию).

Документация `aiohttp` предлагает метод [`json_response`](https://docs.aiohttp.org/en/stable/web_quickstart.html#json-response), который принимает объект, сериализует его в JSON и возвращает новый объект `aiohttp.web.Response` с заголовком `Content-Type: application/json` и сериализованными данными внутри.

??? tldr "Как сериализовать данные с помощью `json_response`"

    ```python
    from aiohttp.web import Application, View, run_app
    from aiohttp.web_response import json_response
    
    
    class SomeView(View):
        async def get(self):
            return json_response({'hello': 'world'})
    
    
    app = Application()
    app.router.add_route('*', '/hello', SomeView)
    run_app(app)
    ```

Но существует и другой способ: `aiohttp` позволяет зарегистрировать произвольный сериализатор для определенного типа данных ответа в реестре `aiohttp.PAYLOAD_REGISTRY`. Например, можно указать сериализатор `aiohttp.JsonPayload` для объектов типа [`Mapping`](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping).

Важно понимать, что метод `json_response`, как и `aiohttp.JsonPayload`, используют стандартный `json.dumps`, который не умеет сериализовать сложные типы данных, например `datetime.date` или `asyncpg.Record` (`asyncpg` возвращает записи из БД в виде экземпляров этого класса). Более того, одни сложные объекты могут содержать другие: в одной записи из БД может быть поле типа `datetime.date`.

Разработчики Python предусмотрели эту проблему: метод `json.dumps` позволяет с помощью аргумента `default` указать функцию, которая вызывается, когда необходимо сериализовать незнакомый объект. Ожидается, что функция приведет незнакомый объект к типу, который умеет сериализовать модуль `json`.

??? tldr "Как расширить `JsonPayload` для сериализации произвольных объектов"

```python
import json
from datetime import date
from functools import partial, singledispatch
from typing import Any

from aiohttp.payload import JsonPayload as BaseJsonPayload
from aiohttp.typedefs import JSONEncoder


@singledispatch
def convert(value):
    raise NotImplementedError(f'Unserializable value: {value!r}')


@convert.register(Record)
def convert_asyncpg_record(value: Record):
    """
    Позволяет автоматически сериализовать результаты запроса, возвращаемые
    asyncpg
    """
    return dict(value)


@convert.register(date)
def convert_date(value: date):
    """
    В проекте объект date возвращается только в одном случае — если необходимо
    отобразить дату рождения. Для отображения даты рождения должен
    использоваться формат ДД.ММ.ГГГГ
    """
    return value.strftime('%d.%m.%Y')


dumps = partial(json.dumps, default=convert)


class JsonPayload(BaseJsonPayload):
    def __init__(self,
                 value: Any,
                 encoding: str = 'utf-8',
                 content_type: str = 'application/json',
                 dumps: JSONEncoder = dumps,
                 *args: Any,
                 **kwargs: Any) -> None:
        super().__init__(value, encoding, content_type, dumps, *args, **kwargs)
```

### Обработчики

`aiohttp` позволяет реализовать обработчики асинхронными функциями и классами. Классы более расширяемы: во-первых, код, относящийся к одному обработчику, можно разместить в одном месте, а во вторых, классы позволяют использовать наследование для избавления от дублирования кода (например, каждому обработчику требуется соединение с базой данных).

??? "Базовый класс обработчика"

    ```python
    from aiohttp.web_urldispatcher import View
    from asyncpgsa import PG
    
    
    class BaseView(View):
        URL_PATH: str
    
        @property
        def pg(self) -> PG:
            return self.request.app['pg']
    ```

Так как один большой файл читать сложно, я решил разнести обработчики по файлам. Маленькие файлы поощряют слабую связность, а если, например, есть кольцевые импорты внутри хэндлеров — значит, возможно, что-то не так с композицией сущностей.

#### `POST /imports`

На вход обработчик получает `json` с данными о жителях. **Максимально допустимый размер запроса** в `aiohttp` регулируется опцией `client_max_size` и [по умолчанию равен 2 МБ](http://docs.aiohttp.org/en/stable/web_reference.html#aiohttp.web.Application). При превышении лимита `aiohttp` вернет HTTP-ответ со статусом `413: Request Entity Too Large Error`.

В то же время корректный `json` c максимально длинными строчками и цифрами будет весить ~63 мегабайта, поэтому ограничения на размер запроса необходимо расширить.

Далее, необходимо **проверить и десериализовать данные**. Если они некорректные, нужно вернуть HTTP-ответ `400: Bad Request`.

Мне потребовались две схемы `Marhsmallow`. Первая, `CitizenSchema`, проверяет данные каждого отдельного жителя, а также десериализует строку с днем рождения в объект `datetime.date`:

* Тип данных, формат и наличие всех обязательных полей;
* Отсутствие незнакомых полей;
* Дата рождения должна быть указана в формате `DD.MM.YYYY` и не может иметь значения из будущего;
* Список родственников каждого жителя должен содержать уникальные существующие в этой выгрузке идентификаторы жителей.

Вторая схема, `ImportSchema`, проверяет выгрузку в целом:

* `citizen_id` каждого жителя в рамках выгрузки должен быть уникален;
* Родственные связи должны быть двусторонними (если у жителя #1 в списке родственников указан житель #2, то и у жителя #2 должен быть родственник #1).

**Если данные корректные, их необходимо добавить в БД** с новым уникальным `import_id`.

Для добавления данных потребуется выполнить несколько запросов в разные таблицы. Чтобы в БД не осталось частично добавленных данных в случае возникновения ошибки или исключения (например, при отключении клиента, который не получил ответ полностью, `aiohttp` [бросит исключение `CancelledError`](http://docs.aiohttp.org/en/stable/web_advanced.html?highlight=shield#web-handler-cancellation)), **необходимо использовать транзакцию**.

**Добавлять данные в таблицы необходимо частями**, так как в одном запросе к PostgreSQL может быть не более 32 767 аргументов. В таблице `citizens` 9 полей. Соответственно, за 1 запрос в эту таблицу можно вставить только 32 767 / 9 = 3640 строк, а в одной выгрузке может быть до 10 000 жителей.

#### `GET /imports/$import_id/citizens`

Обработчик возвращает всех жителей для выгрузки с указанным `import_id`. Если указанная **выгрузка не существует**, необходимо вернуть HTTP-ответ `404: Not Found`. Это поведение выглядит общим для обработчиков, которым требуется существующая выгрузка, поэтому я вынес код проверки в отдельный класс.

??? tldr "Базовый класс для обработчиков с выгрузками"

    ```python
    from aiohttp.web_exceptions import HTTPNotFound
    from sqlalchemy import select, exists
    
    from analyzer.db.schema import imports_table
    
    
    class BaseImportView(BaseView):
        @property
        def import_id(self):
            return int(self.request.match_info.get('import_id'))
    
        async def check_import_exists(self):
            query = select([
                exists().where(imports_table.c.import_id == self.import_id)
            ])
            if not await self.pg.fetchval(query):
                raise HTTPNotFound()
    ```

Чтобы получить список родственников для каждого жителя, потребуется выполнить `LEFT JOIN` из таблицы `citizens` в таблицу `relations`, агрегируя поле `relations.relative_id` с группировкой по `import_id` и `citizen_id`.

Если у жителя нет родственников, то `LEFT JOIN` вернет для него в поле `relations.relative_id` значение `NULL` и в результате агрегации список родственников будет выглядеть как `[NULL]`.

Чтобы исправить это некорректное значение, я воспользовался функцией [`array_remove`](https://postgrespro.ru/docs/postgrespro/12/functions-array).

БД хранит дату в формате `YYYY-MM-DD`, а нам нужен формат `DD.MM.YYYY`.

Технически форматировать дату можно либо SQL-запросом, либо на стороне Python в момент сериализации ответа с `json.dumps` (asyncpg возвращает значение поля `birth_date` как экземпляр класса `datetime.date`).

Я выбрал сериализацию на стороне Python, учитывая, что `birth_date` — единственный объект `datetime.date` в проекте с единым форматом (см. раздел «Сериализация данных»).

Несмотря на то, что в обработчике выполняется два запроса (проверка на существование выгрузки и запрос на получение списка жителей), **использовать транзакцию необязательно**. По умолчанию PostgreSQL использует уровень изоляции `READ COMMITTED` и даже в рамках одной транзакции будут видны все изменения других, успешно завершенных транзакций (добавление новых строк, изменение существующих).

Самая большая выгрузка в текстовом представлении может занимать ~63 мегабайта — это достаточно много, особенно учитывая, что одновременно может прийти несколько запросов на получение данных. Есть достаточно интересный способ **получать данные из БД с помощью [курсора](https://postgrespro.ru/docs/postgrespro/10/plpgsql-cursors) и отправлять их клиенту по частям**.

Для этого нам потребуется реализовать два объекта:

1. Объект `SelectQuery` типа `AsyncIterable`, возвращающий записи из базы данных. При первом обращении подключается к базе, открывает транзакцию и создает курсор, при дальнейшей итерации возвращает записи из БД. Возвращается обработчиком.

    ??? tldr "Код `SelectQuery`"

        ```python
        from collections import AsyncIterable
        from asyncpgsa.transactionmanager import ConnectionTransactionContextManager
        from sqlalchemy.sql import Select
        
        
        class SelectQuery(AsyncIterable):
            """
            Используется, чтобы отправлять данные из PostgreSQL клиенту сразу после
            получения, по частям, без буферизации всех данных
            """
            PREFETCH = 500
        
            __slots__ = (
                'query', 'transaction_ctx', 'prefetch', 'timeout'
            )
        
            def __init__(self, query: Select,
                         transaction_ctx: ConnectionTransactionContextManager,
                         prefetch: int = None,
                         timeout: float = None):
                self.query = query
                self.transaction_ctx = transaction_ctx
                self.prefetch = prefetch or self.PREFETCH
                self.timeout = timeout
        
            async def __aiter__(self):
                async with self.transaction_ctx as conn:
                    cursor = conn.cursor(self.query, prefetch=self.prefetch,
                                         timeout=self.timeout)
                    async for row in cursor:
                        yield row
        ```

2. Сериализатор `AsyncGenJSONListPayload`, который умеет итерироваться по асинхронным генераторам, сериализовать данные из асинхронного генератора в JSON и отправлять данные клиентам по частям. Регистрируется в `aiohttp.PAYLOAD_REGISTRY` как сериализатор объектов `AsyncIterable`.

    ??? tldr "Код `AsyncGenJSONListPayload`"

        ```python
        import json
        from functools import partial
        
        from aiohttp import Payload
        
        
        # Функция, умеющая сериализовать в JSON объекты asyncpg.Record и datetime.date
        dumps = partial(json.dumps, default=convert, ensure_ascii=False)
        
        
        class AsyncGenJSONListPayload(Payload):
            """
            Итерируется по объектам AsyncIterable, частями сериализует данные из них
            в JSON и отправляет клиенту
            """
            def __init__(self, value, encoding: str = 'utf-8',
                         content_type: str = 'application/json',
                         root_object: str = 'data',
                         *args, **kwargs):
                self.root_object = root_object
                super().__init__(value, content_type=content_type, encoding=encoding,
                                 *args, **kwargs)
        
            async def write(self, writer):
                # Начало объекта
                await writer.write(
                    ('{"%s":[' % self.root_object).encode(self._encoding)
                )
        
                first = True
                async for row in self._value:
                    # Перед первой строчкой запятая не нужна
                    if not first:
                        await writer.write(b',')
                    else:
                        first = False
        
                    await writer.write(dumps(row).encode(self._encoding))
        
                # Конец объекта
                await writer.write(b']}')
        ```

Далее, в обработчике можно будет создать объект `SelectQuery`, передать ему SQL запрос и функцию для открытия транзакции и вернуть его в `Response body`:

??? tldr "Код обработчика"

    ```python
    # analyzer/api/handlers/citizens.py
    from aiohttp.web_response import Response
    from aiohttp_apispec import docs, response_schema
    
    from analyzer.api.schema import CitizensResponseSchema
    from analyzer.db.schema import citizens_table as citizens_t
    from analyzer.utils.pg import SelectQuery
    
    from .query import CITIZENS_QUERY
    from .base import BaseImportView
    
    
    class CitizensView(BaseImportView):
        URL_PATH = r'/imports/{import_id:\d+}/citizens'
    
        @docs(summary='Отобразить жителей для указанной выгрузки')
        @response_schema(CitizensResponseSchema())
        async def get(self):
            await self.check_import_exists()
    
            query = CITIZENS_QUERY.where(
                citizens_t.c.import_id == self.import_id
            )
            body = SelectQuery(query, self.pg.transaction())
            return Response(body=body)
    ```

`aiohttp` обнаружит в реестре `aiohttp.PAYLOAD_REGISTRY` зарегистрированный сериализатор `AsyncGenJSONListPayload` для объектов типа `AsyncIterable`. Затем сериализатор будет итерироваться по объекту `SelectQuery` и отправлять данные клиенту. При первом обращении объект `SelectQuery` получает соединение к БД, открывает транзакцию и создает курсор, при дальнейшей итерации будет получать данные из БД курсором и возвращать их построчно.

Этот подход позволяет не выделять память на весь объем данных при каждом запросе, но у него есть особенность: приложение не сможет вернуть клиенту соответствующий HTTP-статус, если возникнет ошибка (ведь клиенту уже был отправлен HTTP-статус, заголовки, и пишутся данные).

При возникновении исключения не остается ничего, кроме как разорвать соединение. Исключение, конечно, можно залогировать, но клиент не сможет понять, какая именно ошибка произошла.

С другой стороны, похожая ситуация может возникнуть, даже если обработчик получит все данные из БД, но при передаче данных клиенту моргнет сеть — от этого никто не застрахован.

#### `PATCH /imports/$import_id/citizens/$citizen_id`

Обработчик получает на вход идентификатор выгрузки `import_id`, жителя `citizen_id`, а также json с новыми данными о жителе. В случае обращения к **несуществующей выгрузке или жителю** необходимо вернуть HTTP-ответ `404: Not Found`.

Переданные клиентом данные требуется **проверить и десериализовать**. Если они некорректные — необходимо вернуть HTTP-ответ `400: Bad Request`. Я реализовал Marshmallow-схему `PatchCitizenSchema`, которая проверяет:

* Тип и формат данных для указанных полей.
* Дату рождения. Она должна быть указана в формате `DD.MM.YYYY` и не может иметь значения из будущего.
* Список родственников каждого жителя. Он должен иметь уникальные идентификаторы жителей

Существование родственников, указанных в поле `relatives`, можно отдельно не проверять: при добавлении в таблицу relations несуществующего жителя PostgreSQL вернет ошибку `ForeignKeyViolationError`, которую можно обработать и вернуть HTTP-статус `400: Bad Request`.

Какой статус возвращать, если клиент прислал некорректные данные для несуществующего жителя или выгрузки? Семантически правильнее проверять сначала существование выгрузки и жителя (если такого нет — возвращать `404: Not Found`) и только потом —корректные ли данные прислал клиент (если нет — возвращать `400: Bad Request`). На практике часто бывает дешевле сначала проверить данные, и только если они корректные, обращаться к базе.

Оба варианта приемлемы, но я решил выбрать более дешевый второй вариант, так как в любом случае результат операции — ошибка, которая ни на что не влияет (клиент исправит данные и потом так же узнает, что житель не существует).

Если данные корректные, необходимо **обновить информацию о жителе в БД**. В обработчике потребуется сделать несколько запросов к разным таблицам. Если возникнет ошибка или исключение, изменения в базе данных должны быть отменены, поэтому **запросы необходимо выполнять в транзакции**.

Метод `PATCH` **позволяет передавать лишь некоторые поля** для изменяемого жителя.

Обработчик необходимо написать таким образом, чтобы он не падал при обращении к данным, которые не указал клиент, а также не выполнял запросы к таблицам, данные в которых не изменились.

Если клиент указал поле `relatives`, необходимо получить список существующих родственников. Если он изменился — определить, какие записи из таблицы `relatives` необходимо удалить, а какие добавить, чтобы привести базу данных в соответствие с запросом клиента. По умолчанию в PostgreSQL для изоляции транзакций используется [уровень READ COMMITTED](https://postgrespro.ru/docs/postgresql/9.6/transaction-iso#xact-read-committed). Это означает, что в рамках текущей транзакции будут видны изменения существующих (а также добавления новых) записей других завершенных транзакций. Это может привести к **состоянию гонки между конкурентными запросами**.

Предположим, существует выгрузка с жителями `#1`, `#2`, `#3`, без родственных связей. Сервис получает два одновременных запроса на изменение жителя `#1`: `{"relatives": [2]}` и `{"relatives": [3]}`. aiohttp создаст два обработчика, которые одновременно получат текущее состояние жителя из PostgreSQL.

Каждый обработчик не обнаружит ни одной родственной связи и примет решение добавить новую связь с указанным родственником. В результате у жителя `#1` поле `relatives` равно `[2,3]`.

![Изображение гонки между конкурентными запросами](/img/gobkmzuhzr47rfzgvtygkxb1stm.png)

Такое поведение нельзя назвать очевидным. Есть два варианта ожидаемо решить исход гонки: выполнить только первый запрос, а для второго вернуть HTTP-ответ `409: Conflict` (чтобы клиент повторил запрос), либо выполнить запросы по очереди (второй запрос будет обработан только после завершения первого).

Первый вариант можно реализовать, включив режим изоляции [`SERIALIZABLE`](https://postgrespro.ru/docs/postgresql/9.6/transaction-iso#xact-serializable). Если во время обработки запроса кто-то уже успел изменить и закоммитить данные, будет брошено исключение, которое можно обработать и вернуть соответствующий HTTP-статус.

Минус такого решения — большое число блокировок в PostgreSQL, `SERIALIZABLE` будет вызывать исключение, даже если конкурентные запросы меняют записи жителей из разных выгрузок.

Также можно воспользоваться механизмом [рекомендательных блокировок](https://postgrespro.ru/docs/postgrespro/10/explicit-locking#ADVISORY-LOCKS). Если получить такую блокировку по `import_id`, конкурентные запросы для разных выгрузок смогут выполняться параллельно.

Для обработки конкурентных запросов в одной выгрузке можно реализовать поведение любого из вариантов: функция `pg_try_advisory_xact_lock` пытается получить блокировку и возвращает результат `boolean` немедленно (если блокировку получить не удалось — можно бросить исключение), а `pg_advisory_xact_lock` ожидает, пока ресурс не станет доступен для блокировки (в этом случае запросы выполнятся последовательно, я остановился на этом варианте).

В итоге обработчик должен вернуть актуальную информацию об обновленном жителе. Можно было ограничиться возвращением клиенту данных из его же запроса (раз мы возвращаем ответ клиенту, значит, исключений не было и все запросы успешно выполнены). Или — воспользоваться ключевым словом `RETURNING` в запросах, изменяющих БД, и сформировать ответ из полученных результатов. Но оба этих подхода не позволили бы увидеть и протестировать случай с гонкой состояний.

К сервису не предъявлялись требования по высокой нагрузке, поэтому я решил запрашивать все данные о жителе заново и возвращать клиенту честный результат из БД.

#### GET /imports/$import_id/citizens/birthdays

Обработчик вычисляет число подарков, которое приобретет каждый житель выгрузки своим родственникам (первого порядка). Число сгруппировано по месяцам для выгрузки с указанным `import_id`. В случае обращения к **несуществующей выгрузке** необходимо вернуть HTTP-ответ `404: Not Found`.

Есть два варианта реализации:

1. Получить данные для жителей с родственниками из базы, а на стороне Python агрегировать данные по месяцам и сгенерировать списки для тех месяцев, для которых нет данных в БД.
2. Составить json-запрос в базу и дописать для отсутствующих месяцев заглушки.

Я остановился на первом варианте — визуально он выглядит более понятным и поддерживаемым. Число дней рождений в определенном месяце можно получить, сделав `JOIN` из таблицы с родственными связями (`relations.citizen_id` — житель, для которого мы считаем дни рождения родственников) в таблицу `citizens` (содержит дату рождения, из которой требуется получить месяц).

Значения месяцев не должны содержать ведущих нулей. Месяц, получаемый из поля `birth_date` c помощью функции `date_part`, может содержать ведущий ноль. Чтобы убрать его, я выполнил `cast` к `integer` в SQL-запросе.

Несмотря на то, что в обработчике требуется выполнить два запроса (проверить существование выгрузки и получить информации о днях рождения и подарках), **транзакция не требуется**.

По умолчанию PostgreSQL использует режим READ COMMITTED, при котором в текущей транзакции видны все новые (добавляемые другими транзакциями) и существующие (изменяемые другими транзакциями) записи после их успешного завершения.

Например, если в момент получения данных будет добавлена новая выгрузка — она никак не повлияет на существующие. Если в момент получения данных будет выполнен запрос на изменение жителя — то либо данные еще не будут видны (если транзакция, меняющая данные, не завершилась), либо транзакция полностью завершится и станут видны сразу все изменения. Целостность получаемых из базы данных не нарушится.

#### `GET /imports/$import_id/towns/stat/percentile/age`

Обработчик вычисляет 50-й, 75-й и 99-й перцентили возрастов (полных лет) жителей по городам в выборке с указанным `import_id`. В случае обращения к несуществующей выгрузке необходимо вернуть HTTP-ответ `404: Not Found`.

Несмотря на то, что в обработчике выполняется два запроса (проверка на существование выгрузки и получение списка жителей), **использовать транзакцию необязательно**.

Есть два варианта реализации:

1. Получить из БД возраста жителей, сгруппированные по городам, а затем на стороне Python вычислить перцентили с помощью numpy (который в задании указан как эталонный) и округлить до двух знаков после запятой.
2. Сделать всю работу на стороне PostgreSQL: функция `percentile_cont` вычисляет перцентиль с линейной интерполяцией, затем округляем полученные значения до двух знаков после запятой в рамках одного SQL-запроса, а numpy используем для тестирования.

Второй вариант требует передавать меньше данных между приложением и PostgreSQL, но у него есть не очень очевидный подводный камень: в PostgreSQL округление математическое, (`SELECT ROUND(2.5)` вернет 3), а в Python — бухгалтерское, к ближайшему целому (`round(2.5)` вернет 2).

Чтобы тестировать обработчик, реализация должна быть одинаковой и в PostgreSQL, и в Python (реализовать функцию с математическим округлением в Python выглядит проще). Стоит отметить, что при вычислении перцентилей numpy и PostgreSQL могут возвращать немного отличающиеся числа, но с учетом округления эта разница будет незаметна.
