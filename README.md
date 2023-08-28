# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию (docker-compose)

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

## Запуск в кластере Kubernetes (minikube)
 - Устанавливаем и запускаем [minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/) 
 - Запускаем в кластере БД PostgreSQL с помощью  [helm](https://helm.sh/ru/docs/intro/install/)
```shell
# Запуск PostgreSQL
helm install pg-db --set auth.postgresPassword=<db_password> oci://registry-1.docker.io/bitnamicharts/postgresql

# Делаем экспорт пароля от админ-пользователя БД в переменную окружения:
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default pg-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

# Создаем `pod` с утилитой `psql` и выполнить в ней команду подключения к БД (после запуска # команды дождитесь загрузки и появления `postgres=#` ):
kubectl run pg-db-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r7 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host pg-db-postgresql -U postgres -d postgres -p 5432

# Создаем пользователя БД (и пароль) для подключения приложения:
CREATE ROLE <username> WITH LOGIN ENCRYPTED PASSWORD '<put-your-password-here>';

# Создаем базу для работы приложения, владельцем которой будет выбранный пользователь:
CREATE DATABASE <database-name> OWNER <username>;
```

- В директории kubernetes создаем файл конфигурации `django-app-config-v1.yaml` с настройками для приложения:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app-config
  labels:
    env: dev
    author: <you_name>
data:
  SECRET_KEY: "django_secret_key"
  DEBUG: "False"
  ALLOWED_HOSTS: "127.0.0.1,star-burger.test"
  DATABASE_URL: "postgres://<db_user>:<db_password>@pg-db-postgresql:5432/<db_name>"

```
 - Применяем файл настроек:
```shell
kubectl apply -f django-app-config-v1.yaml
```
 - Стартуем остальные процессы:
 ```shell
 kubectl apply -f django-app-deploy.yaml,django-app-service.yaml,ingress.yaml,django-app-clearsessions-cronjob.yaml
```
 - Если надо запустить миграции
 ```shell
 kubectl apply -f django-app-migrate-job.yaml
```
 - Создаем суперпользователя приложения
 ```shell
 kubectl exec -it <django-deploy-pod-name> -- python manage.py createsuperuser
```
 - Для того чтобы к сайту можно было получить доступ по DNS имени выполняем
 ```shell
 echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

Проверить работу всех приложений:
```shell
kubectl get deployments.apps,pods,service,configmaps,ingress,cronjobs.batch
```
Для изменения настроек приложения (после внесения изменений в `ConfigMap`)
```shell
kubectl apply -f django-app-config-v1.yaml && kubectl rollout restart deployment django-deploy
```

