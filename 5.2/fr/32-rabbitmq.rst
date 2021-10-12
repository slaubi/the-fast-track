Utiliser RabbitMQ comme gestionnaire de messages
================================================

.. index::
    single: RabbitMQ

RabbitMQ est un gestionnaire de messages très répandu que vous pouvez utiliser comme alternative à PostgreSQL

Basculer de PostgreSQL à RabbitMQ
---------------------------------

Pour utiliser RabbitMQ à la place de PostgreSQL comme gestionnaire de messages :

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

Ajouter RabbitMQ aux services Docker
------------------------------------

.. index::
    single: Docker;RabbitMQ

Comme vous l'avez sûrement deviné, nous avons aussi besoin d'ajouter RabbitMQ aux services Docker Compose :

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

Redémarrer les services Docker
------------------------------

Pour forcer Docker Compose à prendre en compte le conteneur RabbitMQ, arrêter les conteneurs et relancer les :

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Explorer l'interface web de gestion de RabbitMQ
-----------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

Si vous voulez voir les files et les messages défilant dans RabbitMQ, ouvrez son interface web de gestion :

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Ou depuis la barre de débogage web :

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Utilisez ``guest``/``guest`` pour vous connecter sur l'interface de gestion RabbitMQ :

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Déployer RabbitMQ
-----------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Ajouter RabbitMQ aux serveurs de production peut être fait en l'ajoutant à la liste des services :

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

Référencez-le également dans la configuration du conteneur web et activez l'extension PHP ``amqp`` :

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

Quand le service RabbitMQ est installé sur un projet, vous pouvez accéder à l'interface web de gestion en ouvrant tout d'abord un tunnel :

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: Aller plus loin

    * `Documentation de RabbitMQ <https://www.rabbitmq.com/documentation.html>`_.
