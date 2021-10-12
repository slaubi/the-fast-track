Utiliser Redis pour stocker les sessions
========================================

.. index::
    single: Redis

Selon le traffic de votre site web et/ou son infrastructure, vous pourriez vouloir utiliser Redis pour gérer les sessions utilisateur au lieu de PostgreSQL.

Quand nous avons parlé de créer une branche du code du projet pour déplacer les sessions du système de fichiers vers la base de données, nous avons listé toutes les étapes nécessaire à l'ajout d'un nouveau service.

Voici comment vous pouvez ajouter Redis à votre projet en un seul patch :

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - redis
             - blackfire
             - xsl
             - pdo_pgsql
    @@ -26,6 +27,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -15,3 +15,6 @@ varnish:
     files:
         type: network-storage:1.0
         disk: 256
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: '%env(DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -17,3 +17,7 @@ services:
             image: blackfire/blackfire
             env_file: .env.local
             ports: [8707]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

N'est-ce pas *joli* ?

"Rebooter" Docker pour démarrer le service Redis :

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Tester localement en navigant sur le site. Tout devrait continuer à fonctionner comme avant.

Commitez et déployez comme d'habitude :

.. code-block:: bash
    :class: ignore

    $ symfony deploy

.. sidebar:: Aller plus loin

    * `Documentation de Redis <https://redis.io/documentation>`_.
