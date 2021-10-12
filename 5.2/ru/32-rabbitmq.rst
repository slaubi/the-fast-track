Использование RabbitMQ в качестве брокера сообщений
=========================================================================================

.. index::
    single: RabbitMQ

RabbitMQ — очень популярный брокер сообщений, который можно использовать в качестве альтернативы PostgreSQL.

Переход с PostgreSQL на RabbitMQ
------------------------------------------

Сделайте следующие изменение, чтобы использовать RabbitMQ вместо PostgreSQL в качестве брокера сообщений:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -6,7 +6,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                dsn: '%env(RABBITMQ_URL)%'
                     options:
                         auto_setup: false
                         use_notify: true

Добавление RabbitMQ в Docker
---------------------------------------

.. index::
    single: Docker;RabbitMQ

Как вы уже догадались, нам также нужно добавить RabbitMQ в файл Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -21,3 +21,7 @@ services:
         redis:
             image: redis:5-alpine
             ports: [6379]
    +
    +    rabbitmq:
    +        image: rabbitmq:3.7-management
    +        ports: [5672, 15672]

Перезапуск сервисов Docker
--------------------------------------------

Остановите и перезапустите контейнеры, чтобы Docker Compose смог добавить новый контейнер RabbitMQ:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Знакомство с панелью управления RabbitMQ
--------------------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Если вы хотите посмотреть очереди и сообщения, проходящие через RabbitMQ, воспользуйтесь специальной панелью управления:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Это также можно сделать через панель отладки:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Для входа в интерфейс управления RabbitMQ в качестве логина/пароля используйте ``guest``/``guest``:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Развёртывание новой версии с RabbitMQ
--------------------------------------------------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Чтобы активировать RabbitMQ на продакшен-серверах добавьте его в список сервисов:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -18,3 +18,8 @@ files:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

Также укажите его в конфигурации веб-контейнера и включите PHP-модуль ``amqp``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - amqp
             - redis
             - blackfire
             - xsl
    @@ -28,6 +29,7 @@ disk: 512
     relationships:
         database: "db:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"

     web:
         locations:

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

Чтобы перейти в панель управления RabbitMQ на продакшен-сервер прежде всего откройте туннель:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Двигаемся дальше

    * `Документация RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
