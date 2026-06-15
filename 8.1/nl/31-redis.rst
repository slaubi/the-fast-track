Redis gebruiken om sessies op te slaan
======================================

.. index::
    single: Redis

Afhankelijk van het websiteverkeer en/of de infrastructuur, wil je misschien Redis gebruiken om sessies te beheren in plaats van PostgreSQL.

Toen we het hadden over het vertakken van de projectcode om de sessie van het bestandssysteem naar de database te verplaatsen, hebben we alle benodigde stappen opgesomd om een nieuwe service toe te voegen.

Hier zie je hoe je Redis in één patch aan jouw project kunt toevoegen:

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Platform.sh;Redis

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -37,6 +37,7 @@ applications:
                     - iconv
                     - mbstring
                     - pdo_pgsql
    +                - redis
                     - sodium
                     - xsl

    @@ -62,6 +63,7 @@ applications:

             relationships:
                 database: "database:postgresql"
    +            redis: "rediscache:redis"

             hooks:
                 build: |
    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -21,3 +21,6 @@ services:
                 type: network-storage:2.0

    +    rediscache:
    +        type: redis:8.0
    +
     applications:
    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -14,6 +14,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:8.0-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -4,3 +4,3 @@ framework:
         # Note that the session will be started ONLY if you read or write from it.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'

Is het niet *mooi*?

"Herstart" Docker om de Redis service te starten:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

Test lokaal door op de website te bladeren; alles zou nog moeten werken zoals voorheen.

Commit en deploy zoals gewoonlijk:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Verder gaan

    * `Redis documentatie`_.

.. _`Redis documentatie`: https://redis.io/documentation
