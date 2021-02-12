# Непрерывная интеграция (CI - Continuous Integration)

Теперь, когда код покрыт тестами и мы умеем собирать Docker-образ, самое время автоматизировать эти процессы. Первое, что приходит в голову: запускать тесты на создание пул-реквестов, а при добавлении изменений в master-ветку собирать новый Docker-образ и загружать его на [Docker Hub](https://hub.docker.com/) (или [GitHub Packages](https://github.com/features/packages), если вы не собираетесь распространять образ публично).

Я решил эту задачу с помощью [GitHub Actions](https://github.com/features/actions). Для этого потребовалось создать YAML-файл в папке `.github/workflows` и описать в нем workflow (c двумя задачами: `test` и `publish`), которое я назвал `CI`.

Задача `test` выполняется при каждом запуске workflow CI, с помощью [services](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idservices) поднимает контейнер с PostgreSQL, ожидает, когда он станет доступен, и запускает `pytest` в контейнере `snakepacker/python:all`.

Задача `publish` выполняется, только если изменения были добавлены в ветку `master` и если задача `test` была выполнена успешно. Она собирает `source distribution` контейнером `snakepacker/python:all`, затем собирает и загружает Docker-образ с помощью [`docker/build-push-action@v1`](https://github.com/docker/build-push-action).

!!! example "Полное описание workflow"

    ```yml
    name: CI
    
    # Workflow должен выполняться при добавлении изменений
    # или новом пул-реквесте в master
    on:
      push:
        branches: [ master ]
      pull_request:
        branches: [ master ]
    jobs:
      # Тесты должны выполняться при каждом запуске workflow
      test:
        runs-on: ubuntu-latest
    
        services:
          postgres:
            image: docker://postgres
            ports:
              - 5432:5432
            env:
              POSTGRES_USER: user
              POSTGRES_PASSWORD: hackme
              POSTGRES_DB: analyzer
    
        steps:
          - uses: actions/checkout@v2
          - name: test
            uses: docker://snakepacker/python:all
            env:
              CI_ANALYZER_PG_URL: postgresql://user:hackme@postgres/analyzer
            with:
              args: /bin/bash -c "pip install -U '.[dev]' && pylama && wait-for-port postgres:5432 && pytest -vv --cov=analyzer --cov-report=term-missing tests"
    
      # Сборка и загрузка Docker-образа с приложением
      publish:
        # Выполняется только если изменения попали в ветку master
        if: github.event_name == 'push' && github.ref == 'refs/heads/master'
        # Требует, чтобы задача test была выполнена успешно
        needs: test
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v2
          - name: sdist
            uses: docker://snakepacker/python:all
            with:
              args: make sdist
    
          - name: build-push
            uses: docker/build-push-action@v1
              with:
                username: ${{ secrets.REGISTRY_LOGIN }}
                password: ${{ secrets.REGISTRY_TOKEN }}
                repository: alvassin/backendschool2019
                target: api
                tags: 0.0.1, latest
    ```

Теперь при добавлении изменений в `master` во вкладке Actions на GitHub можно увидеть запуск тестов, сборку и загрузку Docker-образа:

![Actions GitHub](/img/r5qxpmy7mlttqniibfvxadvzitw.png)

А при создании пул-реквеста в master-ветку в нем также будут отображаться результаты выполнения задачи `test`:

![Pull-request GitHub](/img/pzd9nkmxq7a75fnlcw2tx37i308.png)
