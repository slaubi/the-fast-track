RabbitMQ als Message-Händler nutzen
====================================

.. index::
    single: RabbitMQ

RabbitMQ ist ein sehr beliebter Message-Händler den Du als Alternative zu PostgreSQL nutzen kannst.

Von PostgreSQL zu RabbitMQ wechseln
-----------------------------------

Um RabbitMQ anstelle von PostgreSQL als Message-Händler zu nutzen:

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

Füge RabbitMQ zum Docker-Stack hinzu
-------------------------------------

.. index::
    single: Docker;RabbitMQ

Wie Du sicherlich schon erraten hast, müssen wir RabbitMQ auch in den Docker Compose-Stack aufnehmen:

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

Neustarten der Docker Dienste
-----------------------------

Um Docker Compose zu zwingen den RabbitMQ-Container zu berücksichtigen, stoppe den Container und starte ihn neu:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Die RabbitMQ Web-Verwaltungs-Oberfläche erkunden
-------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Wenn Du die Queues und Nachrichten sehen willst, die durch RabbitMQ verwaltet werden, öffne seine Web-Verwaltungs-Oberfläche:

.. code-block:: bash
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
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Man kann RabbitMQ auf dem Produktivsystem aktivieren, indem man es zu der Liste der Dienste hinzufügt:

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

Verweise auch darauf in der Web-Container-Konfiguration und aktiviere die ``amqp``-PHP-Erweiterung:

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

Wenn der RabbitMQ-Dienst in Deinem Projekt installiert ist, kannst Du auf die Web-Verwaltungs-Oberfläche über einen Tunnel zugreifen:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Weiterführendes

    * `RabbitMQ-Dokumentation <https://www.rabbitmq.com/documentation.html>`_.
