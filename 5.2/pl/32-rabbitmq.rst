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
    @@ -6,7 +6,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                dsn: '%env(RABBITMQ_URL)%'
                     options:
                         auto_setup: false
                         use_notify: true

Dodawanie RabbitMQ do stosu Dockera
-----------------------------------

.. index::
    single: Docker;RabbitMQ

Jak pewnie się domyślasz, musimy dodać RabbitMQ do stosu Docker Compose:

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

Restartowanie usług Dockera
----------------------------

Aby Docker Compose zauważył RabbitMQ, musisz zatrzymać kontenery i je zrestartować:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Odkrywanie webowego interfejsu do zarządzania RabbitMQ
-------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Jeżeli chcesz zobaczyć kolejki i wiadomości przepływające przez RabbitMQ, otwórz webowy interfejs zarządzania:

.. code-block:: bash
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
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Aby dodać RabbitMQ do serwerów produkcyjnych, dodaj go do listy usług:

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

Dodaj odniesienie do RabbitMQ w konfiguracji kontenera oraz włącz rozszerzenie PHP o nazwie ``amqp``:

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

Aby dostać się do webowego interfejsu zarządzania RabbitMQ, po tym kiedy zostanie on zainstalowany w Twoim projekcie, musisz najpierw otworzyć tunel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Idąc dalej

    * `Dokumentacja RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
