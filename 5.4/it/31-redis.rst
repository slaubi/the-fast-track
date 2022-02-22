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

Non è *bellissimo*?

Riavviare Docker per far partire il servizio Redis:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

Testare localmente visitando il sito web; tutto dovrebbe funzionare come prima.

Facciamo un commit e rilasciamo il codice come sempre:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: Andare oltre

    * `Documentazione di Redis`_.

.. _`Documentazione di Redis`: https://redis.io/documentation
