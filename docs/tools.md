# Tools

## venv (Virtual Environment)

Виртуальная среда обеспечивает изолированное пространство для проектов Python на сервере, то есть, все необходимые зависимости - исполняемые файлы, библиотеки и прочие файлы копируются в некоторый выбранный каталог, а приложение использует их, а не установленные в системе. Это позволяет обеспечить стабильность среды разработки и чистоту основной системы.

### Установка `venv`

```text
sudo apt install -y python3-venv
```

### Создание виртуальной среды для приложения

```text
mkdir myapp && cd myapp
python3 -m venv env
```

### Активация окружения виртуальной среды

```text
source env/bin/activate
```

, где env — это имя вашего окружения разработки

После активации строка приглашения интерпретатора команд будет иметь префикс с именем среды:

```text
(env) netpoint@ubuntu:~/myapp$
```

### Деактивация виртуальной среды

```text
(env) netpoint@ubuntu:~/myapp$ deactivate
```

## MkDocs

[MkDocs](https://www.mkdocs.org/) - это быстрый, простой и просто великолепный генератор статических сайтов, предназначенный для создания проектной документации. Исходные файлы документации написаны на [Markdown](https://ru.wikipedia.org/wiki/Markdown) и настроены с помощью одного файла конфигурации [YAML](https://ru.wikipedia.org/wiki/YAML) `mkdocs.yaml`.

### Установка `mkdocs`

```text
pip install mkdocs
```

Теперь в вашей системе должна быть установлена ​​команда `mkdocs`. Запустите `mkdocs --version`, чтобы убедиться, что все работает нормально.

### Создание сайта с документацией

```text
mkdocs new .
```

Это создаст следующую структуру:

```text
.
├─ docs/
│  └─ index.md
└─ mkdocs.yml
```

### Предварительный просмотр во время написания

MkDocs включает в себя сервер предварительного просмотра в реальном времени, поэтому вы можете предварительно просмотреть свои изменения при написании документации. Сервер автоматически перестроит сайт после сохранения.

```text
mkdocs serve
```

### Визуальная тема MkDocs-Material

[MkDocs-Material](https://squidfunk.github.io/mkdocs-material/) - это тема для MkDocs, генератора статических сайтов, ориентированного на (техническую) проектную документацию.

```text
pip install mkdocs-material
```
