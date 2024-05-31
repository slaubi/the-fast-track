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

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -14,6 +14,7 @@ runtime:
             - iconv
             - mbstring
             - pdo_pgsql
    +        - redis
             - sodium
             - xsl
             
    @@ -40,6 +41,7 @@ mounts:

     relationships:
         database: "database:postgresql"
    +    redis: "rediscache:redis"
         
     hooks:
         build: |
    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -16,3 +16,6 @@ varnish:
     files:
         type: network-storage:2.0
         disk: 256
    +
    +rediscache:
    +    type: redis:5.0
    --- a/compose.yaml
    +++ b/compose.yaml
    @@ -20,6 +20,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:5-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -9,7 +9,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

これは *美しく* ないでしょうか？

Redisサービスを追加するために、Dockerを"再起動"します:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d

ブラウザでWebサイトを表示して、ローカルでテストします。以前と同じくすべての機能が使えます。

通常通りコミットしてデプロイします:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: より深く学ぶために

    * `Redis ドキュメント`_.

.. _`Redis ドキュメント`: https://redis.io/documentation
