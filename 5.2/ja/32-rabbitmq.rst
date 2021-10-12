RabbitMQをメッセージのブローカーとして使う
===========================================================

.. index::
    single: RabbitMQ

RabbitMQはとても有名なメッセージブローカーで、PostgreSQLの代わりに使うことができます。

PostgreSQLからRabbitMQに切り替える
------------------------------------------

メッセージブローカーとしてPostgreSQLの代わりにRabbitMQを利用します:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -6,7 +6,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                dsn: '%env(RABBITMQ_URL)%'
                     options:
                         auto_setup: false
                         use_notify: true

DockerにRabbitMQを追加する
--------------------------------

.. index::
    single: Docker;RabbitMQ

既に予想していたかもしれませんが、RabbitMQをDocker Composeにも追加する必要があります:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -21,3 +21,7 @@ services:
         redis:
             image: redis:5-alpine
             ports: [6379]
    +
    +    rabbitmq:
    +        image: rabbitmq:3.7-management
    +        ports: [5672, 15672]

Dockerサービスを再起動する
------------------------------------

Docker ComposeでRabbitMQ コンテナを扱わせるために、コンテナをすべて停止して再起動します:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

RabbitMQのWeb管理画面を使う
-----------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

RabbitMQを通っていくキューやメッセージを見たければ、RabbitMQのWeb管理画面を開きます:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

あるいはWebデバッグツールバーからも見ることができます:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

RabbitMQの管理画面には ``guest``/``guest`` アカウントを使ってログインします:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

RabbitMQをデプロイする
-----------------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

RabbitMQをプロダクションサーバーに追加するには、サービスのリストに追加します:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -18,3 +18,8 @@ files:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

Webコンテナの設定でも参照し、 ``amqp`` PHP拡張を有効にします:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - amqp
             - redis
             - blackfire
             - xsl
    @@ -28,6 +29,7 @@ disk: 512
     relationships:
         database: "db:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"

     web:
         locations:

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

RabbitMQサービスがプロジェクトにインストールされると、トンネルを使ってWeb管理画面にアクセスできるようになります:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: より深く学ぶために

    * `RabbitMQ ドキュメント <https://www.rabbitmq.com/documentation.html>`_.
