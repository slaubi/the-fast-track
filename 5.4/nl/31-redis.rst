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

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -14,6 +14,7 @@ runtime:
             - iconv
             - mbstring
             - pdo_pgsql
    +        - redis
             - sodium
             - xsl
             
    @@ -39,6 +40,7 @@ mounts:

     relationships:
         database: "database:postgresql"
    +    redis: "rediscache:redis"
         
     hooks:
         build: |
    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
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
             storage_factory_id: session.storage.factory.native
    --- a/docker-compose.yml
    +++ b/docker-compose.yml
    @@ -15,6 +15,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:5-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       db-data:

Is het niet *mooi*?

"Herstart" Docker om de Redis service te starten:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

Test lokaal door op de website te bladeren; alles zou nog moeten werken zoals voorheen.

Commit en deploy zoals gewoonlijk:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: Verder gaan

    * `Redis documentatie`_.

.. _`Redis documentatie`: https://redis.io/documentation
