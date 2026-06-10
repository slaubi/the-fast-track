Использование Redis для хранения сессий
=====================================================================

.. index::
    single: Redis

В зависимости от нагрузки на сайт и/или его инфраструктуру нам возможно понадобится использовать Redis вместо PostgreSQL для управления пользовательскими сессиями.

Когда мы переносили хранение сессий из файловой системы в базу данных, мы перечислили все необходимые шаги для добавления нового сервиса.

Следуя тем же инструкциям, ниже приведён код патча, который активирует Redis для хранения сессий вместо базы данных, как это было раньше:

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Platform.sh;Redis

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -37,6 +37,7 @@ applications:
                     - iconv
                     - mbstring
                     - pdo_pgsql
    +                - redis
                     - sodium
                     - xsl

    @@ -62,6 +63,7 @@ applications:

             relationships:
                 database: "database:postgresql"
    +            redis: "rediscache:redis"

             hooks:
                 build: |
    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -21,3 +21,6 @@ services:
                 type: network-storage:2.0

    +    rediscache:
    +        type: redis:8.0
    +
     applications:
    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -14,6 +14,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:8.0-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -4,3 +4,3 @@ framework:
         # Note that the session will be started ONLY if you read or write from it.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'

Разве это не *прекрасно*?

"Перезагрузите" Docker, чтобы запустить сервис Redis:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

Откройте приложение, чтобы убедиться, что ничего не сломалось.

Как и всегда, зафиксируйте новые изменения в репозитории и опубликуйте новую версию:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Двигаемся дальше

    * `Документация Redis`_.

.. _`Документация Redis`: https://redis.io/documentation
