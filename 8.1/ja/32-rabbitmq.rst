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

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -5,7 +5,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                dsn: '%env(RABBITMQ_URL)%'
                     retry_strategy:
                         max_retries: 3
                         multiplier: 2

メッセンジャー用にRabbitMQサポートを追加する必要があります:

.. code-block:: terminal

    $ symfony composer req amqp-messenger

DockerにRabbitMQを追加する
--------------------------------

.. index::
    single: Docker;RabbitMQ

既に予想していたかもしれませんが、RabbitMQをDocker Composeにも追加する必要があります:

.. code-block:: diff
    :caption: patch_file

    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -18,6 +18,10 @@ services:
         image: redis:8.0-alpine
         ports: [6379]

    +  rabbitmq:
    +    image: rabbitmq:4.2-management
    +    ports: [5672, 15672]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:

Dockerサービスを再起動する
------------------------------------

Docker ComposeでRabbitMQ コンテナを扱わせるために、コンテナをすべて停止して再起動します:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

.. code-block:: terminal
    :class: hide

    $ sleep 10

RabbitMQのWeb管理画面を使う
-----------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

RabbitMQを通っていくキューやメッセージを見たければ、RabbitMQのWeb管理画面を開きます:

.. code-block:: terminal
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
    single: Upsun;RabbitMQ
    single: RabbitMQ

RabbitMQをプロダクションサーバーに追加するには、サービスのリストに追加します:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -25,4 +25,8 @@ services:
             rediscache:
                 type: redis:8.0

    +    queue:
    +        type: rabbitmq:4.2
    +        size: S
    +
     applications:

Webコンテナの設定でも参照し、 ``amqp`` PHP拡張を有効にします:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -39,6 +39,7 @@ applications:

             runtime:
                 extensions:
    +                - amqp
                     - apcu
                     - blackfire
                     - ctype
    @@ -72,5 +73,6 @@ applications:
             relationships:
                 database: "database:postgresql"
                 redis: "rediscache:redis"
    +            rabbitmq: "queue:rabbitmq"

             hooks:
                 build: |

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

RabbitMQサービスがプロジェクトにインストールされると、トンネルを使ってWeb管理画面にアクセスできるようになります:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony cloud:tunnel:close

.. sidebar:: より深く学ぶために

    * `RabbitMQ ドキュメント`_.

.. _`RabbitMQ ドキュメント`: https://www.rabbitmq.com/documentation.html
