# Нагрузочное тестирование

Итак, приложение покрыто тестами, развернуто и готово к эксплуатации. Для полноты картины на минутку вспомним, что поводом для построения сервиса когда-то было техническое задание. В нем были указаны ограничения: на выгрузке с десятью тысячами жителей, из которых тысяча — родственники первого порядка, каждый обработчик должен обрабатывать запрос менее чем за 10 секунд. Безусловно, такое тестирование целесообразно производить именно на конечном сервере (а, скажем, не на CI-сервере): результаты тестирования напрямую зависят от конфигурации сервера и количества доступных ресурсов.

Допустим, мы сгенерировали выгрузку с жителями, вызвали друг за другом все обработчики, каждый из них отработал менее чем за 10 секунд. Достаточно ли этого? Можно предположить, что скорость обработки данных будет деградировать при увеличении количества данных, загружаемых в сервис. Важно понимать, сколько выгрузок сможет обработать сервис, прежде чем обработчики перестанут укладываться в ограничения.

Хоть для тестирования данного сервиса и не требуется генерировать высокий RPS, его нагрузочное тестирование имеет свою особенность: использовать статический набор запросов не получится. Например, чтобы получить список жителей, необходимо иметь идентификатор выгрузки `import_id`, который возвращается обработчиком `POST /imports` и может оказаться любым целым числом. Этот подход называется тестированием по сценарию.

Учитывая, что генерация данных уже реализована на Python 3, я решил воспользоваться фреймворком [Locust](https://locust.io/).

Чтобы выполнить нагрузочное тестирование, необходимо описать сценарий в файле `locustfile.py` и запустить модуль командой `locust`. Затем результаты тестирования можно наблюдать на графиках в веб-интерфейсе или таблице результатов в консоли.

Графики Locust показывают общую информацию. Мне было интересно узнать, на каком раунде сервис не уложится в таймаут. Я добавил переменную с номером текущей
итерации `self.round` и логирование каждого запроса с указанием итерации тестирования и времени выполнения.

!!! example "Описываем сценарий в файле `locustfile.py`"

    ```python
    import logging
    from http import HTTPStatus
    
    from locust import HttpLocust, constant, task, TaskSet
    from locust.exception import RescheduleTask
    
    from analyzer.api.handlers import (
        CitizenBirthdaysView, CitizensView, CitizenView, TownAgeStatView
    )
    from analyzer.utils.testing import generate_citizen, generate_citizens, url_for
    
    
    class AnalyzerTaskSet(TaskSet):
        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)
            self.round = 0
    
        def make_dataset(self):
            citizens = [
                # Первого жителя создаем с родственником. В запросе к
                # PATCH-обработчику список relatives будет содержать только другого
                # жителя, что потребует выполнения максимального кол-ва запросов
                # (как на добавление новой родственной связи, так и на удаление
                # существующей).
                generate_citizen(citizen_id=1, relatives=[2]),
                generate_citizen(citizen_id=2, relatives=[1]),
                *generate_citizens(citizens_num=9998, relations_num=1000,
                                   start_citizen_id=3)
            ]
            return {citizen['citizen_id']: citizen for citizen in citizens}
    
        def request(self, method, path, expected_status, **kwargs):
            with self.client.request(
                    method, path, catch_response=True, **kwargs
            ) as resp:
                if resp.status_code != expected_status:
                    resp.failure(f'expected status {expected_status}, '
                                 f'got {resp.status_code}')
                logging.info(
                    'round %r: %s %s, http status %d (expected %d), took %rs',
                    self.round, method, path, resp.status_code, expected_status,
                    resp.elapsed.total_seconds()
                )
                return resp
    
        def create_import(self, dataset):
            resp = self.request('POST', '/imports', HTTPStatus.CREATED,
                                json={'citizens': list(dataset.values())})
            if resp.status_code != HTTPStatus.CREATED:
                raise RescheduleTask
            return resp.json()['data']['import_id']
    
        def get_citizens(self, import_id):
            url = url_for(CitizensView.URL_PATH, import_id=import_id)
            self.request('GET', url, HTTPStatus.OK,
                         name='/imports/{import_id}/citizens')
    
        def update_citizen(self, import_id):
            url = url_for(CitizenView.URL_PATH, import_id=import_id, citizen_id=1)
            self.request('PATCH', url, HTTPStatus.OK,
                         name='/imports/{import_id}/citizens/{citizen_id}',
                         json={'relatives': [i for i in range(3, 10)]})
    
        def get_birthdays(self, import_id):
            url = url_for(CitizenBirthdaysView.URL_PATH, import_id=import_id)
            self.request('GET', url, HTTPStatus.OK,
                         name='/imports/{import_id}/citizens/birthdays')
    
        def get_town_stats(self, import_id):
            url = url_for(TownAgeStatView.URL_PATH, import_id=import_id)
            self.request('GET', url, HTTPStatus.OK,
                         name='/imports/{import_id}/towns/stat/percentile/age')
    
        @task
        def workflow(self):
            self.round += 1
            dataset = self.make_dataset()
    
            import_id = self.create_import(dataset)
            self.get_citizens(import_id)
            self.update_citizen(import_id)
            self.get_birthdays(import_id)
            self.get_town_stats(import_id)
    
    
    class WebsiteUser(HttpLocust):
        task_set = AnalyzerTaskSet
        wait_time = constant(1)
    ```

Выполнив 100 итераций c максимальными выгрузками, я убедился, что время работы всех обработчиков укладывается в ограничения:

![Statistics Locust](/img/qxrib3jily9gzmexxhupu35tipa.png)

Как видно на графике распределения времени ответов обработчиков, скорость обработки запросов почти не деградирует с ростом количества данных (желтый — 95 перцентиль, зеленый — медиана). Даже со ста выгрузками сервис будет работать эффективно.

![Response Time Locust](/img/dnpxfn5ly7vttziqnnfpvvdsl9g.png)

На графиках потребления ресурсов виден всплеск — установка приложения с помощью Ansible и далее ровное потребление ресурсов с ~20.15 до ~20.30 под нагрузкой от Locust.

![Resources Usage Locust](/img/om1eoknqvzqmyez-plpgykhz8io.png)
