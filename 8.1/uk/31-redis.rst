Використання Redis для зберігання сесій
=====================================================================

.. index::
    single: Redis

Залежно від відвідуваності веб-сайту та/або його інфраструктури, ви можете використовувати Redis замість PostgreSQL, щоб керувати сесіями користувачів.

Коли ми говорили про розгалуження коду проекту для переміщення сесії з файлової системи до бази даних, ми перерахували всі необхідні кроки для додавання нового сервісу.

Ось як ви можете додати Redis у свій проект одним патчем:

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Upsun;Redis

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

Хіба це не *прекрасно*?

"Перезавантажте" Docker, щоб запустити службу Redis:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

Протестуйте локально, переглядаючи веб-сайт; все має працювати як і раніше.

Зафіксуйте й розгорніть, як зазвичай:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Йдемо далі

    * `Документація по Redis`_.

.. _`Документація по Redis`: https://redis.io/documentation
