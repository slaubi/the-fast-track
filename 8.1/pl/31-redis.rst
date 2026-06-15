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

Czyż to nie jest *piękne*?

Zrestartuj Dockera i uruchom usługę Redis:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

Przetestuj lokalnie przeglądając stronę - wszystko powinno działać tak samo jak wcześniej.

Jak zwykle, zatwierdź (ang. commit) i wdróż (ang. deploy) zmiany:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Idąc dalej

    * `Dokumentacja Redis`_.

.. _`Dokumentacja Redis`: https://redis.io/documentation
