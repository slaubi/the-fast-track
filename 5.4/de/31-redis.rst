Nutze Redis um Sessions zu speichern
====================================

.. index::
    single: Redis

Abhängig vom Website-Verkehr und/oder der Infrastruktur möchtest Du vielleicht Redis nutzen anstelle von PostgreSQL um Benutzer-Sessions zu verwalten.

Als wir über das Branchen von unserem Projektcode gesprochen haben um Sessions vom Dateisystem zur Datenbank umzustellen, haben wir alle Schritte aufgeführt, die wir brauchen um einen neuen Dienst hinzu zufügen.

So kannst Du Redis auf einen Schlag zu deinem Projekt hinzufügen:

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

Schick, oder?

Starte Docker neu, um auch den Redis-Dienst zu starten:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

Teste lokal, indem Du Dir die Website anschaust. Alles sollte noch genauso funktionieren wie vorher.

Commit und deploye wie üblich:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: Weiterführendes

    * `Redis-Dokumentation`_.

.. _`Redis-Dokumentation`: https://redis.io/documentation
