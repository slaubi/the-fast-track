RabbitMQ gebruiken als een message-broker
=========================================

.. index::
    single: RabbitMQ

RabbitMQ is een zeer populaire message broker die je kunt gebruiken als alternatief voor PostgreSQL.

Overschakelen van PostgreSQL naar RabbitMQ
------------------------------------------

Om RabbitMQ te gebruiken in plaats van PostgreSQL als message-broker:

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

We moeten ook RabbitMQ ondersteuning toevoegen voor Messenger:

.. code-block:: terminal

    $ symfony composer req amqp-messenger

RabbitMQ toevoegen aan de Docker Stack
--------------------------------------

.. index::
    single: Docker;RabbitMQ

Zoals je misschien al geraden hebt, moeten we ook RabbitMQ toevoegen aan de Docker Compose stack:

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

Docker services herstarten
--------------------------

Om Docker Compose te dwingen rekening te houden met de RabbitMQ-container, stop je de containers en start je ze opnieuw:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

.. code-block:: terminal
    :class: hide

    $ sleep 10

Het verkennen van de RabbitMQ web-beheerinterface
-------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Als je wachtrijen wil zien en berichten door RabbitMQ wil zien vloeien, open dan de web-beheerinterface:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:rabbitmq

Of via de online debug toolbar:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Gebruik ``guest``/``guest`` om in te loggen op de RabbitMQ-beheerinterface:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

RabbitMQ deployen
-----------------

.. index::
    single: Upsun;RabbitMQ
    single: RabbitMQ

RabbitMQ toevoegen aan de productieservers kan, door deze toe te voegen aan de lijst met services:

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

Verwijs ernaar in de web container configuratie en schakel de ``amqp`` PHP-extensie in:

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

Wanneer de RabbitMQ-service in een project is geïnstalleerd, kan je de web beheerinterface openen door eerst de tunnel te openen:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony cloud:tunnel:close

.. sidebar:: Verder gaan

    * `RabbitMQ documentatie`_.

.. _`RabbitMQ documentatie`: https://www.rabbitmq.com/documentation.html
