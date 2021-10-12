Usando RabbitMQ como agente de mensajes
=======================================

.. index::
    single: RabbitMQ

RabbitMQ es un agente de mensajes muy popular que puedes utilizar como alternativa a PostgreSQL.

Cambiando de PostgreSQL a RabbitMQ
----------------------------------

Para utilizar RabbitMQ en lugar de PostgreSQL como agente de mensajes:

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

Agregando RabbitMQ a la pila de Docker
--------------------------------------

.. index::
    single: Docker;RabbitMQ

Como habrás adivinado, también necesitamos agregar RabbitMQ a la pila de Docker Compose:

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

Reiniciando los servicios de Docker
-----------------------------------

Para forzar a Docker Compose a que tenga en cuenta el contenedor RabbitMQ, para los contenedores y reinícialos:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Explorando la interfaz web de administración de RabbitMQ
---------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Si quieres ver las colas y los mensajes que fluyen por RabbitMQ, abre su interfaz web de administración:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

O desde la barra de herramientas de depuración web:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Usa ``guest``/``guest`` para acceder a la interfaz de administración de RabbitMQ:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Desplegando RabbitMQ
--------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Se puede añadir RabbitMQ a los servidores de producción agregándolos a la lista de servicios:

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

Añade la referencia también en la configuración del contenedor web y activa la extensión de PHP ``amqp``:

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

Cuando el servicio RabbitMQ está instalado en un proyecto, se puede acceder a su interfaz de administración web abriendo primeramente el túnel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Yendo más allá

    * `Documentación de RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
