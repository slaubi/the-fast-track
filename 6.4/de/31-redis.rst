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

Schick, oder?

Starte Docker neu, um auch den Redis-Dienst zu starten:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

Teste lokal, indem Du Dir die Website anschaust. Alles sollte noch genauso funktionieren wie vorher.

Commit und deploye wie üblich:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Weiterführendes

    * `Redis-Dokumentation`_.

.. _`Redis-Dokumentation`: https://redis.io/documentation
