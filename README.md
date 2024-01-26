# Django site

- [Описание](#description)
- [Как подготовить окружение к локальной разработке](#prepare)
  - [Как запустить сайт для локальной разработки](#start-local)
  - [Как вести разработку](#howtodev)
  - [Как обновить приложение из основного репозитория](#update)
  - [Как добавить библиотеку в зависимости](#addlib)
- [Развертывание приложения в Minikube](#dep-minicube)
  - [Регулярное удаление сессий Django](#clearsessions)
  - [Миграция базы данных](#migrate)
- [Развертывание приложения в Yandex Cloud](#yc)
  - [Как задеплоить код](#yc-deploy)
  - [Как подготовить dev окружение](#prepare-dev)
- [Переменные окружения](#environs)

## Описание <a name="description"></a>

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке <a name="prepare"></a>

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

### Как запустить сайт для локальной разработки <a name="start-local"></a>

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

### Как вести разработку <a name="howtodev"></a>

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория <a name="update"></a>

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости <a name="addlib"></a>

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

## Развертывание приложения в Minikube <a name="dep-minicube"></a>

Запустите Minikube со своими параметрами. Пример команды:

```
minikube start --cpus=2 --memory=2gb --disk-size=10gb --driver=virtualbox
```

В файле ConfigMap.yaml находятся переменные окружения (см. "Переменные окружения").
Подставьте им свои значения.
Примените манифест ConfigMap к Minikube кластеру:

```
kubectl apply -f ConfigMap.yaml
```

Примените манифест развертывания к Minikube кластеру:

```
kubectl apply -f deployment.yaml
```

Примените манифест Ingress к Minikube кластеру:

```
kubectl apply -f ingress.yaml
```

Узнайте IP адрес виртуальной машины:

```
minikube ip
```

Добавьте в файл /etc/hosts строку:

```
<minikube-ip>  my-host.test
```

Теперь приложение будет доступно по адресу http://my-host.test в вашем веб-браузере.

### Регулярное удаление сессий Django <a name="clearsessions"></a>

Чтобы включить автоматическое удаление устаревших сессий посетителей сайта из базы данных
примените манифест CronJob к Minikube кластеру:

```
kubectl apply -f django-clearsessions.yaml
```

### Миграция базы данных <a name="migrate"></a>

Для выполнения миграции примените манифест командой:

```
kubectl  apply -f django-migrate.yaml
```

## Развертывание приложения в Yandex Cloud <a name="yc"></a>

Установите интерфейс командной строки Yandex Cloud (CLI)
следуя [инструкциям](https://cloud.yandex.com/en/docs/cli/quickstart).
Подключитесь к кластеру. Все необходимые манифесты для деплоя находятся
в директории `yc-sirius/edu-naughty-pike`. Перейдите в нее и следуйте дальнейшим инструкциям.

### Как задеплоить код <a name="yc-deploy"></a>

В файле `nginx-deploy.yaml` змените значение `nodePort: 30321` на свое
согласно настройкам ALB-роутера. Разверните в кластере веб-сервер:

```
kubectl -n <namespace> apply -f nginx-deploy.yaml
```

### Как подготовить dev окружение <a name="prepare-dev"></a>

Для подключения к базе данных [скачайте SSL-сертификат](https://cloud.yandex.ru/ru/docs/managed-postgresql/operations/connect#get-ssl-cert)
для postgresql командой:

```
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```

Сертификат будет сохранен в файле `~/.postgresql/root.crt`

Создайте Secret:

```
kubectl create secret generic postgresql-ssl -n <namespace> --from-file=/path to/root.crt
```

Посмотреть пример подключения секрета к контейнеру можно в файле
`yc-sirius/edu-naughty-pike/ubuntu-example.yaml`

## Переменные окружения <a name="environs"></a>

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
