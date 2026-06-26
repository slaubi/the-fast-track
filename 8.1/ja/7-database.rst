データベースをセットアップする
=============================================

.. index::
    single: Database

カンファレンスゲストブックの Webサイトでは、カンファレンス中のフィードバックを集めるようにします。カンファレンスの参加者からのコメントを永続的に格納できるようにする必要があります。

コメントは、次のデータ構造を持っています: author(著者), email(メールアドレス), text(フィードバックのテキスト), photo(オプショナルの写真)。これらのデータは、リレーショナルデータベースに格納するのに向いています。

今回は、 PostgreSQLをデータベースエンジンとして使います。

Docker Compose へ PostgreSQL を追加
---------------------------------------

.. index::
    single: Docker;PostgreSQL

私たちのローカル開発環境には、Docker を用いてサービスを運用するようにしています。自動生成された ``compose.yml`` ファイルには既に PostgreSQL がサービスとして追加されています:

.. code-block:: yaml
    :caption: compose.yaml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-16}-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB:-app}
            # You should definitely change the password in production
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
            POSTGRES_USER: ${POSTGRES_USER:-app}
    volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

PostgreSQL サーバをインストールし、データベース名やクレデンシャルを制御するための環境変数を設定します。値にこだわりはありません。

また、ローカルホストへ PostgreSQL のポート(``5432``)を公開します。これで自分の開発環境からデータベースへ接続できるようになります:

.. code-block:: yaml
    :caption: compose.override.yaml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    ``pdo_pgsql`` 拡張は、PHP をセットアップした前のステップでインストールされているはずです。

Docker Compose を起動しましょう
---------------------------------------

Docker Compose をバックグラウンドで起動します (``-d``):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

データベースが起動するのを待って、すべて正しく動いているかチェックしましょう:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

もしコンテナが起動していなかったり、 ``State`` カラムが ``Up`` になっていなければ、 Docker Compose のログをチェックしましょう:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

ローカルのデータベースへのアクセス
---------------------------------------------------

Using the ``psql`` command-line utility might prove useful from time to time. But you need to remember the credentials and the database name. Less obvious, you also need to know the local port the database runs on the host. Docker chooses a random port so that you can work on more than one project using PostgreSQL at the same time (the local port is part of the output of ``docker compose ps``).

Symfony CLI で ``psql`` を実行する際は、何も覚えておく必要はありません。

Symfony CLI は自動的にプロジェクトで実行されている Docker サービスを検知し、 ``psql`` コマンドで必要になるデータベース接続に関する環境変数を公開します。

.. index::
    single: Symfony CLI;run psql

この規約があるので、 ``symfony run`` を使ってデータベースに接続することがより簡単になるのです:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    あなたのローカル開発環境に ``psql`` バイナリがないときは、 ``docker compose`` を介して実行することも可能です:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

データベースのダンプとリストア
---------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

``pg_dump`` コマンドを使ってデータベースのデータをダンプします:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

そして、データをリストアします:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

PostgreSQL を Upsun へ追加
------------------------------------

.. index::
    single: Upsun;PostgreSQL

Upsun の本番インフラでは、PostgreSQL のようなサービスを追加する際に、``.upsun/config.yaml`` ファイルを使用しますが、 ``webapp`` パッケージのレシピで既に追加されています:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

``database`` サービスは PostgreSQL データベース (Dockerと同じバージョン) です。Upsunは初回デプロイ時にディスクを自動的に割り当てます。必要に応じて、後から ``symfony cloud:resources:set`` で調整してください。

また、アプリケーションコンテナと DB を "リンク" する必要があります。これは、 ``.upsun/config.yaml`` に記述されます:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

``postgresql`` の ``database`` サービスは、アプリケーションコンテナでは、 ``database`` と参照されます。

``pdo_pgsql`` 拡張がPHP ランタイムに追加されていることを確認しましょう:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Upsun のデータベースへのアクセス
---------------------------------------------------

PostgreSQL が Docker 経由のローカルと、 Upsun の本番で動いています。

ここで見たように、 ``symfony run`` コマンドで環境変数が公開されているので、 ``symfony run psql`` を実行すると Docker によってホストされているデータベースに自動的に接続することができます。

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

本番のコンテナ上の PostgreSQL に接続したければ、ローカル環境と Upsun の間で SSH トンネルを開く事が可能です:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

デフォルトでは、 Upsun のサービスは、ローカル環境に環境変数を公開していません。 ``var:expose-from-tunnel`` コマンドを実行して明示する必要があります。本番のデータベースに接続するのは、危険な運用だからです。*本当の* データを壊してしまうかもしれません。フラグを必須とすることで、あなたがしようとしていることの確認をしているのです。

今度は、前のように ``symfony run psql`` 経由でリモートの PostgreSQL データベースへ接続しましょう:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

接続が終わったら、トンネルをクローズするのを忘れないでください:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Shell の代わりに本番のデータベース上で SQL を実行するのに ``symfony sql`` コマンドも使用することができます。

環境変数を公開する
---------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

環境変数を使うことで、Docker Compose と Upsun はシームレスに Symfony と連携することができます。

``symfony var:export`` を実行して ``symfony`` が公開している全ての環境変数をチェックしてみましょう:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

``PG*`` 環境変数は ``psql`` ユーティリティで使われます。他はどうでしょうか？

``var:expose-from-tunnel`` コマンドを実行して、 Upsun へのトンネルをオープンすると、 ``var:export`` コマンドは、リモートの環境変数を返します:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

インフラストラクチャ情報を記述する
---------------------------------------------------

まだ気づいていないかもしれませんが、インフラストラクチャ情報をコードと一緒にファイルに保存することは、非常に役に立ちます。DockerやUpsunは設定ファイルを使って、プロジェクトのインフラストラクチャ情報を記述します。新しい機能でサービスの追加が必要な場合、同じパッチで、コードとインフラストラクチャを変更することができます。

.. sidebar:: より深く学ぶために

    * `Upsun サービス`_;

    * `Upsun トンネル`_;

    * `PostgreSQL のドキュメント`_;

    * `Docker Compose コマンド`_.

.. _`Upsun サービス`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Upsun トンネル`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`PostgreSQL のドキュメント`: https://www.postgresql.org/docs/
.. _`Docker Compose コマンド`: https://docs.docker.com/reference/cli/docker/compose/
