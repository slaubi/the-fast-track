Używanie Redisa do przechowywania sesji
========================================

.. index::
    single: Redis

W zależności od ruchu na Twojej stronie i jej infrastruktury, możesz zechcieć używać Redisa do obsługi sesji użytkowników zamiast PostgreSQL.

Kiedy mówiliśmy o podziale kodu projektu w taki sposób, aby przenieść obsługę sesji z plików do bazy danych, wymieniliśmy wszystkie niezbędne kroki potrzebne do dodania nowej usługi.

Aby dodać Redisa do swojego projektu wystarczy wprowadzić następującą poprawkę:

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

Czyż to nie jest *piękne*?

Zrestartuj Dockera i uruchom usługę Redis:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

Przetestuj lokalnie przeglądając stronę - wszystko powinno działać tak samo jak wcześniej.

Jak zwykle, zatwierdź (ang. commit) i wdróż (ang. deploy) zmiany:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: Idąc dalej

    * `Dokumentacja Redis`_.

.. _`Dokumentacja Redis`: https://redis.io/documentation
