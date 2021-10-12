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

Is het niet *mooi*?

"Herstart" Docker om de Redis service te starten:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Test lokaal door op de website te bladeren; alles zou nog moeten werken zoals voorheen.

Commit en deploy zoals gewoonlijk:

.. code-block:: bash
    :class: ignore

    $ symfony deploy

.. sidebar:: Verder gaan

    * `Redis documentatie <https://redis.io/documentation>`_.
