コードをブランチ運用する
====================================

プロジェクトにおいてコード変更のワークフローを整理する方法は数多く存在します。しかし、Git の master ブランチで直接作業し、テストすること無しにそのまま本番環境にデプロイする方法はおそらく最善策ではありません。

テストするということは、ユニットテストやファンクショナルテストを実行することだけでなく、アプリケーションの振る舞いを本番データを使って確認することも意味します。アプリケーションがエンドユーザーにデプロイされた時に、あなた自身や `ステークホルダー`_ がブラウザで確認できるのであれば、そのことは大きなアドバンテージであり、自信を持ってデプロイすることが可能となります。非技術者が新機能を検証できることはとても心強いことです。

次のステップでは、単純化と冗長な記述を避けるために Git の master ブランチ上で全ての作業を続けますが、どうやってより良くするか見ていきましょう。

Git のワークフローを取り入れる
-------------------------------------------

ワークフローの一例として、新しいフィーチャーやバグ修正毎にブランチを作成する方法があります。これは単純で効果的なやり方です。

ブランチを作成する
---------------------------

.. index::
    single: Git;branch
    single: Git;checkout

このワークフローは、Git ブランチの作成から始まります:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: bash

    $ git checkout -b sessions-in-db

このコマンドで ``master`` ブランチから ``sessions-in-db`` ブランチを作成します。そうすることで、コードとインフラストラクチャの設定を "フォーク" します。

データベースにセッションを格納する
---------------------------------------------------

.. index::
    single: Session;Database

ブランチ名から推測できるかもしれませんが、セッションストレージをファイルシステムからデータベース (PostgreSQL) に変更したいと考えています。

それを実現するために必要な手順は典型的なものです:

#. Git ブランチを作成します。

#. 必要に応じて Symfony の設定を変更します。

#. 必要に応じてコードの追加や編集を行います。

#. 必要があればPHP の設定を変更します（PostgreSQL PHP extension を追加したりなど）。

#. 必要であればDocker と SymfonyCloud 上のインフラストラクチャを変更します（PostgreSQL サービスを追加します）。

#. ローカルでテストします。

#. リモートでテストします。

#. master ブランチにマージします。

#. 本番環境にデプロイします。

#. ブランチを削除します。

セッションをデータベースに保存するには、 ``session.handler_id`` をデータベースDSNを指すように変更します。

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

セッションをデータベースに保存するには、 ``sessions`` テーブルの作成が必要です。 Doctrineマイグレーションを使って作成します:

.. code-block:: bash

    $ symfony console make:migration

``up()`` メソッドにテーブル追加の記述をおこないます:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -21,6 +21,14 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs

    +        $this->addSql('
    +            CREATE TABLE sessions (
    +                sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    +                sess_data BYTEA NOT NULL,
    +                sess_lifetime INTEGER NOT NULL,
    +                sess_time INTEGER NOT NULL
    +            )
    +        ');
         }

         public function down(Schema $schema): void

データベースのマイグレーションを行います:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Web サイトを閲覧してローカルでテストしてください。セッションをまだ利用しておらず見た目の変更がないので、全てこれまでと同じように機能するはずです。

.. note::

    今回はセッションストレージにデータベースを再利用しているので、ステップ3から5は必要ありません。しかし、後述のRedisを使用する章では Dockerや Symfony Cloud 上でサービスの追加、テストやデプロイがいかに容易かを示します。

新しいテーブルは Doctrine で管理しないので、次のデータベースマイグレーションでテーブルを削除しないように Doctrine を設定しなければいけません。

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/doctrine.yaml
    +++ b/config/packages/doctrine.yaml
    @@ -5,6 +5,8 @@ doctrine:
             # IMPORTANT: You MUST configure your server version,
             # either here or in the DATABASE_URL env var (see .env file)
             #server_version: '13'
    +
    +        schema_filter: ~^(?!session)~
         orm:
             auto_generate_proxy_classes: true
             naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware

新しいブランチに変更をコミットします:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

ブランチをデプロイする
---------------------------------

.. index::
    single: SymfonyCloud;Environment

本番環境にデプロイする前に、本番環境と同一のインフラストラクチャでこのブランチをテストすべきでしょう。また、 Symfony の ``prod`` 環境で全てが正常に動作することを検証すべきです（ローカルの Web サイトでは Symfony の ``dev`` 環境を利用していました）。

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

では、 *Git ブランチ* に基づいた *SymfonyCloud 環境* を作成しましょう:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-db --no-interaction

.. code-block:: bash

    $ symfony env:create

このコマンドで以下の新しい環境を作成します:

* このブランチは現在の Git ブランチ (``sessions-in-db``) のコードとインフラストラクチャを引き継ぎます。

* データベースやファイル（例えばユーザーがアップロードしたファイルです）を含む全サービスのデータについて一貫したスナップショットを取得することにより、マスタ環境（本番環境）からデータが取得されます。

* コード、データ、インフラストラクチャをデプロイするために専用のクラスターが新たに作成されます。

今回のデプロイは本番へのデプロイと同じ手順に従うため、データベースのマイグレーションも実行されます。これは本番のデータでマイグレーションが機能することを検証するための良い方法です。

非 ``master`` 環境は ``master`` 環境と非常に良く似ていますが、多少の差分が存在します。例えば、メールはデフォルトでは送信されません。

.. index::
    single: Symfony CLI;open:remote

デプロイが完了したら、新しいブランチをブラウザで開きます:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

すべての SymfonyCloud コマンドは現在の Git ブランチ上で動作することに気付きましたか？このコマンドを実行すると ``sessions-in-db`` ブランチのデプロイ URL が開きます。URL は ``https://sessions-in-db-xxx.eu.s5y.io/`` となります。

新しい環境の Web サイトをテストしてみてください。マスタ環境で作成した全てのデータを確認できるはずです。

``master`` 環境でさらにカンファレンスを追加した場合、``sessions-in-db`` 環境には表示されません。その逆の場合も同様です。各環境は独立しており、隔離されています。

もし master 上のコードが更新されていたら、コードとインフラストラクチャのコンフリクトを解消して、Git ブランチをリベースし、最新のバージョンをデプロイすることができます。

.. index::
    single: Symfony CLI;env:sync

マスタ環境から ``sessions-in-db`` 環境にデータを同期することも可能です:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

デプロイ前に本番のデプロイをデバッグする
------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

すべての SymfonyCloud 環境はデフォルトで ``master``/``prod`` 環境（Symfonyの ``prod`` 環境）と同じ設定を利用しています。これによって、実際の条件でアプリケーションをテストすることが出来ます。本番サーバー上で直に開発とテストを行っている感覚になりますが、リスクはありません。FTP経由でデプロイしていた古き良き時代を思い出させますね。

.. index::
    single: Symfony CLI;env:debug

何か問題が起きた時に、Symfony の ``dev`` 環境に切り替えたくなるかもしれません:

.. code-block:: bash

    $ symfony env:debug

問題が解決した後は、本番設定に戻しておきましょう:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    ``master`` ブランチ上では、**決して** ``dev`` 環境を有効化したり Symfony プロファイラを有効化したりしないでください。アプリケーションの動作がとても重たくなり、セキィリティ上の深刻な脆弱性が多く発生してしまいます。

デプロイ前に本番のデプロイをテストする
---------------------------------------------------------

次期バージョンの Web サイトを本番データでアクセスすることで、見た目の回帰テストから性能テストに至るまで、多くの事を確認できます。`Blackfire <https://blackfire.io>`_ はこの仕事をやる上で完璧なツールです。

Blackfireを使ってデプロイ前にコードをテストする方法をもっと学ぶためには、"パフォーマンス" に関する手順を参照してください。

本番へマージする
------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

ブランチの変更内容に問題が無ければ、コードとインフラストラクチャを Git の master ブランチにマージしてください:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-db

そしてデプロイを実行します:

.. code-block:: bash

    $ symfony deploy

デプロイされると、コードとインフラストラクチャの変更のみ SymfonyCloud にプッシュされます。データは一切影響を受けません。

クリーンアップする
---------------------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

最後に、Git ブランチ と SymfonyCloud 環境を削除してクリーンアップします:

.. code-block:: bash

    $ git branch -d sessions-in-db
    $ symfony env:delete --env=sessions-in-db --no-interaction

.. sidebar:: より深く学ぶために

    * `Git ブランチ <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

.. _`ステークホルダー`: https://en.wikipedia.org/wiki/Project_stakeholder
