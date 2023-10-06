# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

Образ с БД считывает следующие переменные окружения:  
`POSTGRES_DB` -- имя БД

`POSTGRES_USER` -- пользователь БД

`POSTGRES_PASSWORD` -- пароль для пользователя БД

## Деплой с minikube

- Убедитесь, что запущен контейнер БД, в данном случае можно использовать команду:
```sh
docker-compose -f docker-compose.db.yml up -d
```
- Запустите **minikube**
```sh
minikube start
```
- Создайте configmap с переменными окружения ([полезная статья](https://humanitec.com/blog/handling-environment-variables-with-kubernetes))

- Разверните приложение, используя следующую команду:
```sh
kubectl apply -f django_app.yml 
```
- Добавьте hosts для ingress (предварительно необходимо его настроить, см.следующую секцию):
```sh
kubectl apply -f ingress-hosts.yml 
```

### Настройка Ingress

Для доступа к сайту через URL необходимо использовать ingress controller  
В данном случае используется **Ingress-Nginx Controller**

- Включите Ingress-Nginx Controller в кластере minikube командой:
```sh
minikube addons enable ingress
```

- Создайте configmap для TCP services:
```sh
kubectl apply -f tcp-services.yml
```

- Добавьте TCP services к nginx ingress controller:
```sh
kubectl patch configmap tcp-services -n ingress-nginx --patch '{"data":{"port":"default/web-service:port"}}'
```
где:  
&nbsp;&nbsp;&nbsp;&nbsp; **port** - порт, который прослушивает ваш service  
&nbsp;&nbsp;&nbsp;&nbsp; **default** -  namespace в котором располагается service  
&nbsp;&nbsp;&nbsp;&nbsp; **web-service** - имя service  

- Последний шаг, необходимо пропатчить ingress-nginx-controller:
```sh
kubectl patch deployment ingress-nginx-controller --patch "$(cat ingress-nginx-controller-patch.yaml)" -n ingress-nginx
```

- Проверить соединение можно следующей командой:
```sh
telnet $(minikube ip) port
```
Вы должны увидеть что-то подобное:
```sh
Trying 192.168.49.2...
Connected to 192.168.49.2.
Escape character is '^]'.
```

## Удаление сессий

С помощью CronJob можно настроить автоматическое удаление сессий по расписанию (в нашем примере - раз в месяц).  
Для этого применим следующий конфигурационный файл:
```sh
kubectl apply -f django-clearsessions.yaml
```