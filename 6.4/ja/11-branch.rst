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

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

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

#. Update the PHP configuration if needed (like adding the PostgreSQL PHP
   extension);

#. 必要であればDocker と Platform.sh 上のインフラストラクチャを変更します（PostgreSQL サービスを追加します）。

#. ローカルでテストします。

#. リモートでテストします。

#. master ブランチにマージします。

#. 本番環境にデプロイします。

#. ブランチを削除します。

セッションをデータベースに保存するには、 ``session.handler_id`` をデータベースDSNを指すように変更します。

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -8,7 +8,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(resolve:DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

セッションをデータベースに保存するには、 ``sessions`` テーブルの作成が必要です。 Doctrineマイグレーションを使って作成します:

.. code-block:: terminal

    $ symfony console make:migration

データベースのマイグレーションを行います:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Web サイトを閲覧してローカルでテストしてください。セッションをまだ利用しておらず見た目の変更がないので、全てこれまでと同じように機能するはずです。

.. note::

    We don't need steps 3 to 5 here as we are re-using the database as the session storage, but the chapter about using Redis shows how straightforward it is to add, test, and deploy a new service in both Docker and Platform.sh.

新しいブランチに変更をコミットします:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

ブランチをデプロイする
---------------------------------

.. index::
    single: Platform.sh;Environment

本番環境にデプロイする前に、本番環境と同一のインフラストラクチャでこのブランチをテストすべきでしょう。また、 Symfony の ``prod`` 環境で全てが正常に動作することを検証すべきです（ローカルの Web サイトでは Symfony の ``dev`` 環境を利用していました）。

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

では、 *Git ブランチ* に基づいた *Platform.sh 環境* を作成しましょう:

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:deploy

このコマンドで以下の新しい環境を作成します:

* このブランチは現在の Git ブランチ (``sessions-in-db``) のコードとインフラストラクチャを引き継ぎます。

* データベースやファイル（例えばユーザーがアップロードしたファイルです）を含む全サービスのデータについて一貫したスナップショットを取得することにより、マスタ環境（本番環境）からデータが取得されます。

* コード、データ、インフラストラクチャをデプロイするために専用のクラスターが新たに作成されます。

今回のデプロイは本番へのデプロイと同じ手順に従うため、データベースのマイグレーションも実行されます。これは本番のデータでマイグレーションが機能することを検証するための良い方法です。

非 ``master`` 環境は ``master`` 環境と非常に良く似ていますが、多少の差分が存在します。例えば、メールはデフォルトでは送信されません。

.. index::
    single: Symfony CLI;cloud:url

デプロイが完了したら、新しいブランチをブラウザで開きます:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

すべての Platform.sh コマンドは現在の Git ブランチ上で動作することに気付きましたか？このコマンドを実行すると ``sessions-in-db`` ブランチのデプロイ URL が開きます。URL は ``https://sessions-in-db-xxx.eu-5.platformsh.site/`` となります。

新しい環境の Web サイトをテストしてみてください。マスタ環境で作成した全てのデータを確認できるはずです。

``master`` 環境でさらにカンファレンスを追加した場合、``sessions-in-db`` 環境には表示されません。その逆の場合も同様です。各環境は独立しており、隔離されています。

もし master 上のコードが更新されていたら、コードとインフラストラクチャのコンフリクトを解消して、Git ブランチをリベースし、最新のバージョンをデプロイすることができます。

.. index::
    single: Symfony CLI;cloud:env:sync

マスタ環境から ``sessions-in-db`` 環境にデータを同期することも可能です:

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

デプロイ前に本番のデプロイをデバッグする
------------------------------------------------------------

.. index::
    single: Platform.sh;Debugging

すべての Platform.sh 環境はデフォルトで ``master``/``prod`` 環境（Symfonyの ``prod`` 環境）と同じ設定を利用しています。これによって、実際の条件でアプリケーションをテストすることが出来ます。本番サーバー上で直に開発とテストを行っている感覚になりますが、リスクはありません。FTP経由でデプロイしていた古き良き時代を思い出させますね。

.. index::
    single: Symfony CLI;cloud:env:debug

何か問題が起きた時に、Symfony の ``dev`` 環境に切り替えたくなるかもしれません:

.. code-block:: terminal

    $ symfony cloud:env:debug

問題が解決した後は、本番設定に戻しておきましょう:

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    ``master`` ブランチ上では、**決して** ``dev`` 環境を有効化したり Symfony プロファイラを有効化したりしないでください。アプリケーションの動作がとても重たくなり、セキィリティ上の深刻な脆弱性が多く発生してしまいます。

デプロイ前に本番のデプロイをテストする
---------------------------------------------------------

次期バージョンの Web サイトを本番データでアクセスすることで、見た目の回帰テストから性能テストに至るまで、多くの事を確認できます。`Blackfire`_ はこの仕事をやる上で完璧なツールです。

Blackfireを使ってデプロイ前にコードをテストする方法をもっと学ぶためには、:doc:`Performance <29-performance>`  に関する手順を参照してください。

本番へマージする
------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

ブランチの変更内容に問題が無ければ、コードとインフラストラクチャを Git の master ブランチにマージしてください:

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

そしてデプロイを実行します:

.. code-block:: terminal

    $ symfony cloud:deploy

デプロイされると、コードとインフラストラクチャの変更のみ Platform.sh にプッシュされます。データは一切影響を受けません。

クリーンアップする
---------------------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

最後に、Git ブランチ と Platform.sh 環境を削除してクリーンアップします:

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: より深く学ぶために

    * `Git ブランチ`_;

.. _`ステークホルダー`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Git ブランチ`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
