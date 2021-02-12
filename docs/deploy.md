# Деплой

Чтобы развернуть приложение на предоставленном сервере, нужно установить Docker, Docker Compose, запустить контейнеры с приложением и PostgreSQL и применить миграции.

Эти шаги можно автоматизировать с помощью системы управления конфигурациями Ansible. Она написана на Python, не требует специальных агентов (подключается прямо по ssh), использует jinja-шаблоны и позволяет декларативно описывать желаемое состояние в YAML-файлах. Декларативный подход позволяет не задумываться о текущем состоянии системы и действиях, необходимых, чтобы привести систему к желаемому состоянию. Вся эта работа ложится на плечи модулей Ansible.

Ansible позволяет сгруппировать логически связанные задачи в роли и затем переиспользовать. Нам потребуются две роли: `docker` (устанавливает и настраивает Docker) и `analyzer` (устанавливает и настраивает приложение).

**Роль `docker`** добавляет в систему репозиторий с Docker, устанавливает и настраивает пакеты `docker-ce` и `docker-compose`.

Опционально можно наладить автоматическое возобновление работы REST API после перезагрузки сервера. Ubuntu позволяет решить эту задачу силами системы инициализации [`systemd`](https://ru.wikipedia.org/wiki/Systemd). Она управляет юнитами, представляющими собой различные ресурсы (демоны, сокеты, точки монтирования и другие). Чтобы добавить новый юнит в `systemd`, необходимо описать его конфигурацию в отдельном файле `.service` и разместить этот файл в одной из специальных папок, например в `/etc/systemd/system`. Затем юнит можно запустить, а также включить для него автозагрузку.

Пакет `docker-ce` при установке автоматически создаст файл с конфигурацией юнита — необходимо только убедиться, что он запущен и включается при запуске системы. Для Docker Compose файл конфигурации `docker-compose@.service` будет создан силами Ansible. Символ `@` в названии указывает `systemd`, что юнит является шаблоном. Это позволяет запускать сервис `docker-compose` с параметром — например, с названием нашего сервиса, который будет подставлен вместо `%i` в файле конфигурации юнита:

```ini
[Unit]
Description=%i service with docker compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/etc/docker/compose/%i
ExecStart=/usr/local/bin/docker-compose up -d --remove-orphans
ExecStop=/usr/local/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

Роль analyzer сгенерирует из шаблона файл `docker-compose.yml` по адресу `/etc/docker/compose/analyzer`, зарегистрирует приложение как автоматически запускаемый сервис в `systemd` и применит миграции. Когда роли готовы, необходимо описать playbook.

```yml
---

- name: Gathering facts
  hosts: all
  become: yes
  gather_facts: yes

- name: Install docker
  hosts: docker
  become: yes
  gather_facts: no
  roles:
    - docker

- name: Install analyzer
  hosts: api
  become: yes
  gather_facts: no
  roles:
    - analyzer
```

Список хостов, а также переменные, использованные в ролях, можно указать в inventory-файле `hosts.ini`.

```ini
[api]
130.193.51.154

[docker:children]
api

[api:vars]
analyzer_image = alvassin/backendschool2019
analyzer_pg_user = user
analyzer_pg_password = hackme
analyzer_pg_dbname = analyzer
```

После того, как все файлы Ansible будут готовы, запустим его:

```text
$ ansible-playbook -i hosts.ini deploy.yml
```
