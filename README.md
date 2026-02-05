# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$docker-compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$docker-compose exec web python manage.py migrate  # создаём/обновляем таблицы в БД
$docker-compose exec web python manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

```bash
$git pull
$docker-compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker-compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$docker-compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

```html
<a name="env-variables"></a>
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Развертывание Django приложения в Kubernetes

### Предварительные требования

- Установленный и запущенный Minikube
- Настроенный `kubectl` для работы с Minikube

### Быстрый старт

1. **Создайте все необходимые ресурсы Kubernetes:**

```bash
# Примените все конфигурационные файлы
kubectl apply -f kubernetes/secret.yaml
kubectl apply -f kubernetes/django-deployment.yaml
kubectl apply -f kubernetes/django-service.yaml
kubectl apply -f kubernetes/ingress.yaml
```

2. **Включите Ingress контроллер в Minikube:**

```bash
minikube addons enable ingress
```

3. **Проверьте, что все компоненты запустились:**

```bash
# Проверьте поды
kubectl get pods

# Проверьте сервисы
kubectl get services

# Проверьте Ingress
kubectl get ingress

# Проверьте поды Ingress контроллера
kubectl get pods -n ingress-nginx
```

### Настройка локального домена

Для доступа к сайту по домену star-burger.test необходимо добавить запись в локальный файл hosts:

**Windows**

1. Откройте файл `C:\Windows\System32\drivers\etc\hosts` с правами администратора

2. Добавьте строку:

```text
127.0.0.1 star-burger.test
```

**macOS / Linux**

1. Откройте терминал и отредактируйте файл:

```bash
sudo nano /etc/hosts
```

2. Добавьте строку:

```text
127.0.0.1 star-burger.test
```

3. Сохраните изменения (Ctrl+O, Enter, Ctrl+X)

### Важное примечание для Windows и macOS

При использовании Minikube с Docker драйвером необходимо запустить туннель для корректной работы Ingress:

```bash
# Запустите в отдельном окне терминала и оставьте его открытым:
minikube tunnel
```

Не закрывайте окно с minikube tunnel! Оно должно оставаться активным для доступа к сайту.

### Проверка работоспособности

После выполнения всех шагов сайт будет доступен по адресу:

```text
http://star-burger.test
```

## Очистка сессии

Автоматическая очистка устаревших сессий настраивается через CronJob, который запускается каждый месяц.

### Конфигурация

Файл: `django-clearsessions.yaml`
Расписание: первый день месяца в 00:00 (по cron: `0 0 1 * *`)

### Управление

- Проверить CronJob: `kubectl get cronjobs`
- Ручной запуск: `kubectl create job --from=cronjob/django-clearsessions django-clearsessions-once`
- Проверить выполнение: `kubectl get jobs`

## Миграция при обновлении приложения

Запустите Job, для выполнения миграций

```bash
kubectl apply -f django-migrate.yaml
```
