Використання RabbitMQ у якості брокера повідомлень
=======================================================================================

.. index::
    single: RabbitMQ

RabbitMQ — це дуже популярний брокер повідомлень, який ви можете використовувати як альтернативу PostgreSQL.

Перехід від PostgreSQL до RabbitMQ
----------------------------------------------

Щоб використовувати RabbitMQ замість PostgreSQL у якості брокера повідомлень:

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

Додавання RabbitMQ у стек Docker
----------------------------------------------

.. index::
    single: Docker;RabbitMQ

Як ви могли здогадатися, нам також потрібно додати RabbitMQ у стек Docker Compose:

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

Перезавантаження сервісів Docker
--------------------------------------------------------

Щоб змусити Docker Compose взяти до уваги контейнер RabbitMQ, зупиніть контейнери й перезавантажте їх:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Ознайомлення з веб-інтерфейсом управління RabbitMQ
---------------------------------------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Якщо ви хочете побачити черги й повідомлення, які проходять через RabbitMQ, відкрийте його веб-інтерфейс управління:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Або з панелі інструментів веб-налагодження:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Використовуйте ``guest``/``guest``, щоб увійти до інтерфейсу управління RabbitMQ:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Розгортання RabbitMQ
-------------------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Додавання RabbitMQ до продакшн серверів можна здійснити додавши його до списку сервісів:

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

Також вкажіть його в конфігурації веб-контейнера й увімкніть розширення PHP ``amqp``:

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

Коли сервіс RabbitMQ встановлено в проекті, ви можете отримати доступ до його веб-інтерфейсу управління, спочатку відкривши тунель:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Йдемо далі

    * `Документація по RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
