RabbitMQ als Message-Broker nutzen
==================================

.. index::
    single: RabbitMQ

RabbitMQ ist ein sehr beliebter Message-Broker den Du als Alternative zu PostgreSQL nutzen kannst.

Von PostgreSQL zu RabbitMQ wechseln
-----------------------------------

Um RabbitMQ anstelle von PostgreSQL als Message-Broker zu nutzen:

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

Wir müssen auch RabbitMQ-Unterstützung für Messenger hinzufügen:

.. code-block:: terminal

    $ symfony composer req amqp-messenger

Füge RabbitMQ zum Docker-Stack hinzu
-------------------------------------

.. index::
    single: Docker;RabbitMQ

Wie Du sicherlich schon erraten hast, müssen wir RabbitMQ auch in den Docker Compose-Stack aufnehmen:

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

Neustarten der Docker Dienste
-----------------------------

Um Docker Compose zu zwingen den RabbitMQ-Container zu berücksichtigen, stoppe den Container und starte ihn neu:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

.. code-block:: terminal
    :class: hide

    $ sleep 10

Die RabbitMQ Web-Verwaltungs-Oberfläche erkunden
-------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Wenn Du die Queues und Nachrichten sehen willst, die durch RabbitMQ verwaltet werden, öffne seine Web-Verwaltungs-Oberfläche:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:rabbitmq

Oder mit der Web-Debug-Toolbar:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Nutze ``guest``/``guest`` als Login für die RabbitMQ Verwaltungs-Oberfläche:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

RabbitMQ deployen
-----------------

.. index::
    single: Platform.sh;RabbitMQ
    single: RabbitMQ

Man kann RabbitMQ auf dem Produktivsystem aktivieren, indem man es zu der Liste der Dienste hinzufügt:

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

Verweise auch darauf in der Web-Container-Konfiguration und aktiviere die ``amqp``-PHP-Erweiterung:

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
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

Wenn der RabbitMQ-Dienst in einem Projekt installiert ist, kannst Du auf die Web-Verwaltungs-Oberfläche über einen Tunnel zugreifen:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony cloud:tunnel:close

.. sidebar:: Weiterführendes

    * `RabbitMQ-Dokumentation`_.

.. _`RabbitMQ-Dokumentation`: https://www.rabbitmq.com/documentation.html
