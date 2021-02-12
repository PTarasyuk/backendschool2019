# Сборка

Конечный артефакт, который мы собираемся разворачивать и который хотим получить в результате сборки, — Docker-образ. Для сборки необходимо **выбрать базовый образ** c Python. Официальный образ python:latest весит ~1 ГБ и, если его использовать в качестве базового, образ с приложением будет огромным. Существуют [образы на основе ОС Alpine](https://github.com/jfloff/alpine-python#why), размер которых намного меньше. Но с растущим количеством устанавливаемых пакетов размер конечного образа вырастет, и в итоге даже образ, собранный на основе Alpine, будет не таким уж и маленьким. Я выбрал в качестве базового образа [snakepacker/python](https://github.com/snakepacker/python) — он весит немного больше Alpine-образов, но основан на Ubuntu, которая предлагает огромный выбор пакетов и библиотек.

Еще один способ **уменьшить размер образа с приложением** — не включать в итоговый образ компилятор, библиотеки и файлы с заголовками для сборки, которые не потребуются для работы приложения.

Для этого можно воспользоваться [многоступенчатой сборкой](https://docs.docker.com/develop/develop-images/multistage-build/#use-multi-stage-builds) Docker:

1. С помощью «тяжелого» образа `snakepacker/python:all` (~1 ГБ, в сжатом виде ~500 МБ) создаем виртуальное окружение, устанавливаем в него все зависимости и пакет с приложением. Этот образ нужен исключительно для сборки, он может содержать компилятор, все необходимые библиотеки и файлы с заголовками.

    ```dockerfile
    FROM snakepacker/python:all as builder
    
    # Создаем виртуальное окружение
    RUN python3.8 -m venv /usr/share/python3/app
    
    # Копируем source distribution в контейнер и устанавливаем его
    COPY dist/ /mnt/dist/
    RUN /usr/share/python3/app/bin/pip install /mnt/dist/*
    ```

2. Готовое виртуальное окружение копируем в «легкий» образ `snakepacker/python:3.8` (~100 МБ, в сжатом виде ~50 МБ), который содержит только интерпретатор требуемой версии Python.

    **Важно:** в виртуальном окружении используются абсолютные пути, поэтому его необходимо скопировать по тому же адресу, по которому оно было собрано в контейнере-сборщике.

```dockerfile
FROM snakepacker/python:3.8 as api

# Копируем готовое виртуальное окружение из контейнера builder
COPY --from=builder /usr/share/python3/app /usr/share/python3/app

# Устанавливаем ссылки, чтобы можно было воспользоваться командами приложения
RUN ln -snf /usr/share/python3/app/bin/analyzer-* /usr/local/bin/

# Устанавливаем выполняемую при запуске контейнера команду по умолчанию
CMD ["analyzer-api"]
```

Чтобы **сократить время на сборку образа**, зависимые модули приложения можно установить до его установки в виртуальное окружение. Тогда Docker закеширует их и не будет устанавливать заново, если они не менялись.

!!! example "`Dockerfile` целиком"

    ```dockerfile
    ############### Образ для сборки виртуального окружения ################
    # Основа — «тяжелый» (~1 ГБ, в сжатом виде ~500 ГБ) образ со всеми необходимыми
    # библиотеками для сборки модулей
    FROM snakepacker/python:all as builder
    
    # Создаем виртуальное окружение и обновляем pip
    RUN python3.8 -m venv /usr/share/python3/app
    RUN /usr/share/python3/app/bin/pip install -U pip
    
    # Устанавливаем зависимости отдельно, чтобы закешировать. При последующей сборке
    # Docker пропустит этот шаг, если requirements.txt не изменится
    COPY requirements.txt /mnt/
    RUN /usr/share/python3/app/bin/pip install -Ur /mnt/requirements.txt
    
    # Копируем source distribution в контейнер и устанавливаем его
    COPY dist/ /mnt/dist/
    RUN /usr/share/python3/app/bin/pip install /mnt/dist/* \
        && /usr/share/python3/app/bin/pip check
    
    ########################### Финальный образ ############################
    # За основу берем «легкий» (~100 МБ, в сжатом виде ~50 МБ) образ с Python
    FROM snakepacker/python:3.8 as api
    
    # Копируем в него готовое виртуальное окружение из контейнера builder
    COPY --from=builder /usr/share/python3/app /usr/share/python3/app
    
    # Устанавливаем ссылки, чтобы можно было воспользоваться командами приложения
    RUN ln -snf /usr/share/python3/app/bin/analyzer-* /usr/local/bin/
    
    # Устанавливаем выполняемую при запуске контейнера команду по умолчанию
    CMD ["analyzer-api"]
    ```

Для удобства сборки я добавил команду `make upload`, которая собирает Docker-образ и загружает его на hub.docker.com.