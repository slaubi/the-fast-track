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

    --- i/.platform.app.yaml
    +++ w/.platform.app.yaml
    @@ -10,6 +10,7 @@ runtime:
             - iconv
             - mbstring
             - pdo_pgsql
    +        - redis
             - sodium
             - xsl
             
    @@ -36,6 +37,7 @@ mounts:

     relationships:
         database: "database:postgresql"
    +    redis: "rediscache:redis"
         
     hooks:
         build: |
    --- i/.platform/services.yaml
    +++ w/.platform/services.yaml
    @@ -16,3 +16,6 @@ varnish:
     files:
         type: network-storage:2.0
         disk: 256
    +
    +rediscache:
    +    type: redis:5.0
    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -14,6 +14,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:5-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -8,7 +8,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

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
