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

Czyż to nie jest *piękne*?

Zrestartuj Dockera i uruchom usługę Redis:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Przetestuj lokalnie przeglądając stronę - wszystko powinno działać tak samo jak wcześniej.

Jak zwykle, zatwierdź (ang. commit) i wdróż (ang. deploy) zmiany:

.. code-block:: bash
    :class: ignore

    $ symfony deploy

.. sidebar:: Idąc dalej

    * `Dokumentacja Redis <https://redis.io/documentation>`_.
