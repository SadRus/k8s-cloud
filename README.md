# Django site

Готовые манифесты Kubernetes для деплоя докеризированного сайта на Django в облаке.

Сайт доступен по ссылке https://edu-berserk-jones.sirius-k8s.dvmn.org/  

[Ссылка на веб-консоль яндекс облако](https://console.cloud.yandex.ru/folders/b1gtcctl0mkamhmvoq79/managed-kubernetes/cluster/cat528346gdueh53ts39/overview).  Namespace **edu-berserk-jones**

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` - обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` - настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` - настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` - адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Деплой

- Должен быть установлен kubectl
```sh
sudo apt-get update
sudo apt-get install -y kubectl
```

- У вас должен быть доступ к кластеру, например развернутому в яндекс облаке  
Для подключения к кластеру обновите конфиг ~/.kube/config, зачастую инструкция для подключения указана на хостинге  

- Установите OpenLens, k9s и т.п. для удобного мониторинга вашего кластера через графический интерфейс  

- У вас должен быть доступ к БД  
В данном случае используется **Yandex Managed Service for PostgreSQL**, данные для подключения лежат в secrets

- Также необходимо создать ConfigMap (переменные окружение см. выше), пример команды:
```sh
kubectl apply -f django-config.yaml
```

- Т.к. в манифесте для образа приложения выбран тег - latest, то будет использована свежая версия сайта с DockerHub, либо можно скачать свежий образ самостоятельно, используя команду:
```sh
docker pull sadrus/k8s_djangoapp:latest
```

- Перед запуском или при обновлении сайта необходимо применить миграции БД:
```sh
kubectl apply -f django-migrate.yaml
```

- Логи можно смотреть в kubectl logs, например:
```sh
kubectl logs svc/web-service 
```

- Разверните приложение, используя следующую команду:
```sh
kubectl apply -f django_app.yaml 
```

## Удаление сессий

С помощью CronJob можно настроить автоматическое удаление сессий по расписанию (в нашем примере - раз в месяц).  
Для этого применим следующий конфигурационный файл:
```sh
kubectl apply -f django-clearsessions.yaml
```
