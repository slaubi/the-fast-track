Używanie RabbitMQ jako pośrednika wiadomości
===============================================

.. index::
    single: RabbitMQ

RabbitMQ jest bardzo popularnym pośrednikiem wiadomości (ang. message broker), który możesz wykorzystać jako alternatywę dla PostgreSQL.

Zmiana PostgreSQL na RabbitMQ
-----------------------------

Wprowadź następujące zmiany, aby użyć RabbitMQ zamiast PostgreSQL jako pośrednika wiadomości:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    -                options:
    -                    use_notify: true
    -                    check_delayed_interval: 60000
    +                dsn: '%env(RABBITMQ_URL)%'
                     retry_strategy:
                         max_retries: 3
                         multiplier: 2

Musimy również dodać obsługę RabbitMQ dla Messengera:

.. code-block:: terminal

    $ symfony composer req amqp-messenger

Dodawanie RabbitMQ do stosu Dockera
-----------------------------------

.. index::
    single: Docker;RabbitMQ

Jak pewnie się domyślasz, musimy dodać RabbitMQ do stosu Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yml
    +++ b/docker-compose.yml
    @@ -19,6 +19,10 @@ services:
         image: redis:5-alpine
         ports: [6379]

    +  rabbitmq:
    +    image: rabbitmq:3.7-management
    +    ports: [5672, 15672]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       db-data:

Restartowanie usług Dockera
----------------------------

Aby Docker Compose zauważył RabbitMQ, musisz zatrzymać kontenery i je zrestartować:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: terminal
    :class: hide

    $ sleep 10

Odkrywanie webowego interfejsu do zarządzania RabbitMQ
-------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Jeżeli chcesz zobaczyć kolejki i wiadomości przepływające przez RabbitMQ, otwórz webowy interfejs zarządzania:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:rabbitmq

lub wykorzystaj pasek narzędzi do debugowania:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Użyj kombinacji ``guest``/``guest`` aby zalogować się do webowego interfejsu zarządzania RabbitMQ.

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Wdrażanie RabbitMQ
-------------------

.. index::
    single: Platform.sh;RabbitMQ
    single: RabbitMQ

Aby dodać RabbitMQ do serwerów produkcyjnych, dodaj go do listy usług:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -18,3 +18,8 @@ files:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

Dodaj odniesienie do RabbitMQ w konfiguracji kontenera oraz włącz rozszerzenie PHP o nazwie ``amqp``:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -8,6 +8,7 @@ dependencies:

     runtime:
         extensions:
    +        - amqp
             - apcu
             - blackfire
             - ctype
    @@ -41,6 +42,7 @@ mounts:
     relationships:
         database: "database:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"
         
     hooks:
         build: |

.. index::
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

Aby dostać się do webowego interfejsu zarządzania RabbitMQ, po tym kiedy zostanie on zainstalowany w Twoim projekcie, musisz najpierw otworzyć tunel:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony cloud:tunnel:close

.. sidebar:: Idąc dalej

    * `Dokumentacja RabbitMQ`_.

.. _`Dokumentacja RabbitMQ`: https://www.rabbitmq.com/documentation.html
