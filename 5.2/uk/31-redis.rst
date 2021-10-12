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
    single: SymfonyCloud;Redis

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - redis
             - blackfire
             - xsl
             - pdo_pgsql
    @@ -26,6 +27,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -15,3 +15,6 @@ varnish:
     files:
         type: network-storage:1.0
         disk: 256
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: '%env(DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -17,3 +17,7 @@ services:
             image: blackfire/blackfire
             env_file: .env.local
             ports: [8707]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Хіба це не *прекрасно*?

"Перезавантажте" Docker, щоб запустити службу Redis:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Протестуйте локально, переглядаючи веб-сайт; все має працювати як і раніше.

Зафіксуйте й розгорніть, як зазвичай:

.. code-block:: bash
    :class: ignore

    $ symfony deploy

.. sidebar:: Йдемо далі

    * `Документація по Redis <https://redis.io/documentation>`_.
