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
    single: Platform.sh;Redis

.. code-block:: diff
    :caption: patch_file

    --- i/.platform.app.yaml
    +++ w/.platform.app.yaml
    @@ -10,6 +10,7 @@ runtime:
             - iconv
             - mbstring
             - pdo_pgsql
    +        - redis
             - sodium
             - xsl
             
    @@ -36,6 +37,7 @@ mounts:

     relationships:
         database: "database:postgresql"
    +    redis: "rediscache:redis"
         
     hooks:
         build: |
    --- i/.platform/services.yaml
    +++ w/.platform/services.yaml
    @@ -16,3 +16,6 @@ varnish:
     files:
         type: network-storage:2.0
         disk: 256
    +
    +rediscache:
    +    type: redis:5.0
    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -14,6 +14,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:5-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -8,7 +8,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

N'est-ce pas *joli* ?

"Rebooter" Docker pour démarrer le service Redis :

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

Tester localement en navigant sur le site. Tout devrait continuer à fonctionner comme avant.

Commitez et déployez comme d'habitude :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: Aller plus loin

    * `Documentation de Redis`_.

.. _`Documentation de Redis`: https://redis.io/documentation
