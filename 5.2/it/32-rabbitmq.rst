Usare RabbitMQ come Message Broker
==================================

.. index::
    single: RabbitMQ

RabbitMQ è un message broker molto popolare che può essere utilizzato come alternativa a PostgreSQL.

Passare da PostgreSQL a RabbitMQ
--------------------------------

Per utilizzare RabbitMQ al posto di PostgreSQL come message broker:

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

Aggiungere RabbitMQ allo stack Docker
-------------------------------------

.. index::
    single: Docker;RabbitMQ

Come potreste aver intuito, abbiamo bisogno di aggiungere RabbitMQ allo stack di Docker Compose:

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

Riavviare i servizi Docker
--------------------------

Per forzare Docker Compose a prendere in considerazione il container RabbitMQ, fermare i container e riavviarli:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Esplorare l'interfaccia web di gestione di RabbitMQ
---------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Se volete vedere le code e i messaggi che transitano attraverso RabbitMQ, aprite l'interfaccia web di gestione:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

O dalla barra di debug:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Inserire ``guest``/``guest`` come username e password, per accedere alla UI di gestione di RabbitMQ:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Distribuire RabbitMQ
--------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Aggiungere RabbitMQ ai server di produzione può essere fatto aggiungendolo alla lista dei servizi:

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

Fare riferimento ad esso anche nella configurazione del container web, ed abilitare l'estensione PHP ``amqp``:

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

Quando il servizio RabbitMQ è installato su di un progetto, potete accedere alla sua interfaccia di gestione web aprendo un tunnel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Andare oltre

    * `Documentazione di RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
