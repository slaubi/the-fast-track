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

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -5,7 +5,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
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

    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -18,6 +18,10 @@ services:
         image: redis:8.0-alpine
         ports: [6379]

    +  rabbitmq:
    +    image: rabbitmq:4.2-management
    +    ports: [5672, 15672]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:

Restartowanie usług Dockera
----------------------------

Aby Docker Compose zauważył RabbitMQ, musisz zatrzymać kontenery i je zrestartować:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

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
    single: Upsun;RabbitMQ
    single: RabbitMQ

Aby dodać RabbitMQ do serwerów produkcyjnych, dodaj go do listy usług:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -25,4 +25,8 @@ services:
             rediscache:
                 type: redis:8.0

    +    queue:
    +        type: rabbitmq:4.2
    +        size: S
    +
     applications:

Dodaj odniesienie do RabbitMQ w konfiguracji kontenera oraz włącz rozszerzenie PHP o nazwie ``amqp``:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -39,6 +39,7 @@ applications:

             runtime:
                 extensions:
    +                - amqp
                     - apcu
                     - blackfire
                     - ctype
    @@ -72,5 +73,6 @@ applications:
             relationships:
                 database: "database:postgresql"
                 redis: "rediscache:redis"
    +            rabbitmq: "queue:rabbitmq"

             hooks:
                 build: |

.. index::
    single: Upsun;Tunnel
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
