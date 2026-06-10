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
