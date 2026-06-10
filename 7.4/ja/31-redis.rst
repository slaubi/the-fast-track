Redisにセッションを保存する
======================================

.. index::
    single: Redis

Webサイトのトラフィックやインフラストラクチャによって、PostgreSQLではなくRedisを使ってセッションを管理したいかもしれません。

プロジェクトのコードを分岐させて、セッションをファイルシステムからデータベースにセッションを移行したとき、新しいサービスを追加するすべての必要なステップを並べました。

こちらが、1つのパッチでプロジェクトにRedisを追加する方法です:

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

これは *美しく* ないでしょうか？

Redisサービスを追加するために、Dockerを"再起動"します:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

ブラウザでWebサイトを表示して、ローカルでテストします。以前と同じくすべての機能が使えます。

通常通りコミットしてデプロイします:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: より深く学ぶために

    * `Redis ドキュメント`_.

.. _`Redis ドキュメント`: https://redis.io/documentation
