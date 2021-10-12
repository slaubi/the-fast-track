Utilizează RabbitMQ ca broker de mesaje
========================================

.. index::
    single: RabbitMQ

RabbitMQ este un broker de mesaje foarte popular pe care îl poți utiliza ca alternativă la PostgreSQL.

Trecerea de la PostgreSQL la RabbitMQ
-------------------------------------

Pentru a utiliza RabbitMQ în locul la PostgreSQL ca broker de mesaje:

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

Adaugă RabbitMQ la stack-ul Docker
-----------------------------------

.. index::
    single: Docker;RabbitMQ

După cum probabil ai ghicit, trebuie să adăugăm și RabbitMQ în stack-ul Docker Compose:

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

Repornirea serviciilor Docker
-----------------------------

Pentru a forța Docker Compose să ia în considerare containerul RabbitMQ, oprește containerele și pornește-le din nou:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Explorarea interfeței de gestionare web RabbitMQ
-------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Dacă vrei să vezi cozile și mesajele care trec prin RabbitMQ, deschide interfața de administrare web:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Sau din bara de depanare:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Folosește ``guest``/``guest`` pentru a te conecta la interfața de gestionare RabbitMQ:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Publicarea pe server la RabbitMQ
--------------------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Adăugarea RabbitMQ la serverele de producție se poate face prin adăugarea acestuia la lista de servicii:

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

Adaugă referința în configurația containerului web și activează extensia PHP ``amqp``:

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

Când serviciul RabbitMQ este instalat într-un proiect, poți accesa interfața sa de administrare web deschizând mai întâi un tunel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Mergând mai departe

    * `Documentația RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
