Utilizzare Redis per memorizzare le sessioni
============================================

.. index::
    single: Redis

A seconda del traffico sul sito web e/o la sua infrastruttura, potreste volere utilizzare Redis per gestire le sessioni utente al posto di PostgreSQL.

Quando abbiamo parlato di utilizzare branch per il codice di progetto, al fine di spostare le sessioni dal filesystem al database, abbiamo elencato tutti gli step necessari per aggiungere un nuovo servizio.

Qui è dove possiamo aggiungere Redis al progetto in un sol colpo:

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Upsun;Redis

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

Non è *bellissimo*?

Riavviare Docker per far partire il servizio Redis:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

Testare localmente visitando il sito web; tutto dovrebbe funzionare come prima.

Facciamo un commit e rilasciamo il codice come sempre:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Andare oltre

    * `Documentazione di Redis`_.

.. _`Documentazione di Redis`: https://redis.io/documentation
