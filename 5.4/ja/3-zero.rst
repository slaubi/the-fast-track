ゼロの状態からプロダクションまでやってみよう
==================================================================

さぁ始めましょう。まず、可能な限り早くプロジェクトが動くようにしたいと思います。まだ、何も開発していないので、まず "Under construction" のページを表示するだけのページから始めましょう。

理想的で、時代遅れで、アニメーションがある "Under construction" の GIF がインターネットにあるか探していたのですが、`これ`_ を使おうと思います:

.. image:: images/under-construction.gif
    :align: center

楽しくなると言ったでしょう。

プロジェクトの初期化
------------------------------

新しい Symfony のプロジェクトを、前章で説明した ``symfony`` CLI ツールで作成しましょう。

.. code-block:: terminal

    $ symfony new guestbook --version=5.4 --php=8.1 --webapp --docker --cloud
    $ cd guestbook

このコマンドは ``Composer`` コマンドの薄いラッパーで、Symfony プロジェクトを作成することを簡単にしてくれます。このコマンドは、最小限の依存のみを含んでいます。それは、どんなプロジェクトでも必要になるコンソールツールや HTTP アブストラクションなどのWebアプリケーションを作成するのに必要なSymfony コンポーネントの依存を含んだ `プロジェクトのスケルトン`_ を使用します。

十分な機能のあるウェブアプリケーションを作ろうとしているので、いくつか便利なオプションを追加しました。

* ``--webapp``: デフォルトでは、必要最小限の依存のみでアプリケーションを作ります。ウェブアプリケーションのプロジェクトでは ``webapp`` パッケージを使うことをおすすめします。このパッケージには「モダンな」ウェブアプリケーションに必要なほとんどのパッケージが含まれています。 ``webapp`` パッケージはSymfony MessengerやDoctrineとPostgreSQLのようなたくさんのSymfonyパッケージを追加します。

* ``--docker``:  ローカル開発環境ではDockerを使ってPostgreSQLのようなサービスを管理します。このオプションを有効にすると、Dockerが有効になり、パッケージを追加したときにSymfonyが自動的にそのパッケージに必要なDockerサービスを追加します。（たとえば、ORMを追加したときにPostgreSQLサービスを追加したり、Symfony Mailerを追加したときにmail catcherを追加したりします）

* ``--cloud``: プロジェクトをPlatform.shにデプロイしたい場合、このオプションを有効にすると、適切なPlatform.shの設定ファイルを自動で作成できます。Platform.shはSymfonyプロジェクトのテスト環境・ステージング環境・本番環境をクラウド上に構築する、最もシンプルでおすすめの環境です。

スケルトンの GitHub のリポジトリを見てみると、ほとんど何もないことに気づくでしょう。 ``composer.json`` のみです。しかし、 ``guestbook`` ディレクトリはファイルがたくさん入っています 。どうやってやっているのでしょうか？答えは ``symfony/flex`` パッケージです。 Symfony Flex は Composer のプラグインで、インストールの処理をフックしています。Symfony Flex が *レシピ* を検知すると、実行してくれるのです。

この Symfony レシピのマニフェストファイルで、Symfony アプリケーション内のパッケージを自動登録するように記述してあります。README を読まなくても Symfony のパッケージをインストールす ることができます。自動化が Symfony の鍵となる機能ですから。

Git が自分の開発パソコンにインストールされていれば、 ``symfony new`` コマンドは Git リポジトリも作成してくれ、最初のコミットも追加してくれます。

ディレクトリ構造を見てみましょう:

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

``bin/`` ディレクトリは、よく使う CLI コマンドの ``console`` が入っています。これからたくさん使うことなります。

``config/`` ディレクトリは、デフォルトと注意が必要な設定の一式が入っています。各パッケージで1つのファイルとなります。ほとんど変更することもないと思います。デフォルト設定を使用するのは良いアイデアですね。

``public/`` ディレクトリは、Webルートのディレクトリで、すべての HTTP のリソースのエントリーポイントである ``index.php`` ファイルがあります。

``src/`` ディレクトリは、あなたが書くことになるコードが入る場所で、開発時のほとんどはここを使用することになります。デフォルトでは、このディレクトリに入る全てのクラスは ``App`` ネームスペースを使用することになります。

``var/`` ディレクトリは、キャッシュやログやアプリケーションによってラインタイムで生成されるファイルが格納されます。触る必要はありません。このディレクトリのみが本番において、書き込み可能な場所になります。

``vendor/`` ディレクトリは Symfony 自体も含め、Composer によってインストールされたすべてのパッケージが格納されます。ここがより生産的になるのに重要な秘密兵器になります。車輪の再発明は止めましょう。大変な作業は既存のライブラリに任せる方が良いです。このディレクトリは Composer によって管理されているので触らないでください。

現段階で、知る必要があるのはこれだけです。

公開するファイルの作成
---------------------------------

``public/`` 配下のファイルはブラウザからアクセスが可能です。例えば、アニメーションGIFファイルを ``public/images/`` ディレクトリに移動したなら、 ``https://localhost/images/under-construction.gif`` のような URL で参照できるでしょう。

GIF 画像をここからダウンロードしてください:

.. code-block:: terminal

    $ mkdir public/images/
    $ php -r "copy('http://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

ローカルのWebサーバの起動
------------------------------------

.. index::
    single: Symfony CLI;server:start

``symfony`` CLI コマンドは、開発用に最適化されたWebサーバとしても機能します。Symfony とうまく連携してくれるのですが、開発用としての使用のみで、決して本番環境では使用してはいけません。

プロジェクトのディレクトリからバックグラウンドでWebサーバを動かしましょう (``-d`` フラグ):

.. code-block:: terminal

    $ symfony server:start -d

サーバは 8000 番からはじまる使用可能なポートで立ち上がります。ショートカットを使用して、 CLI からブラウザでwebサイトを開いてみましょう:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local

あなたのデフォルトのブラウザが立ち上がり、次のようなページが表示されると思います:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    トラブルシューティングの際は、 ``symfony server:log``; コマンドを使用しましょう。このコマンドはWebサーバや PHP やあなたのアプリケーションのログを tail してくれます。

``/images/under-construction.gif`` を見てください。こんな感じになりましたか？

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

では、今作成したものをコミットしましょう。

.. code-block:: terminal
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

favicon を追加する
-----------------------

favicon がないので、ブラウザからのリクエストでによってログに 404 HTTP エラー が書かれしまうので、追加しましょう:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/favicon.ico', 'public/favicon.ico');"
    $ git add public/
    $ git commit -m'Add a favicon'

本番の準備
---------------

.. index::
    single: Platform.sh;Initialization

本番に今までの作業内容をデプロイしてみましょう。まだユーザーにウェルカムを表示するための HTML ページもないのはわかっています。しかし、まず、 "under construction" イメージを表示できるようにすることは、最初のステップとしては良いものだと思います。そして、 *速く頻繁にデプロイする* というモットーですね。

PHP をサポートしているどんなプロバイダーでもこのアプリケーションをホストすることが可能です。つまり、世の中のほとんどのホスティングプロバイダーが対象となります。しかし、少しチェックすることがあります。PHP のバージョンが最新であり、データベースやキューなどのサービスをホストできるプロバイダーが良いですね。

私が選択したのは `Platform.sh`_ です。 Platform.sh は私達が必要なものをすべて提供してくれますし、Symfony の開発の資金ともなってもいます。

.. index::
    single: Symfony CLI;project:init

``--cloud`` オプションを有効にしてプロジェクトを作成したため、``.platform/services.yaml`` 、 ``.platform/routes.yaml`` 、 ``.platform.app.yaml`` といった必要な設定ファイルが作られており、Platform.shにプロジェクトが初期化されています。

本番へ
---------

.. index::
    single: Symfony CLI;cloud:project:create
    single: Symfony CLI;cloud:deploy

デプロイの時間？

新しい Platform.sh リモートプロジェクトを作成してください:

.. code-block:: terminal

    $ symfony cloud:project:create --title="Guestbook" --plan=development

このコマンドはたくさんのことを行います:

* はじめてこのコマンドを使用すると、Platform.sh のクレデンシャルの認証をまだしていなかった場合は、認証を行います。

* 新しい Platform.sh のプロジェクトを用意します(初めて作成した開発プロジェクトでは、30日間は *無料* で使用できます)。

デプロイしましょう:

.. code-block:: terminal

    $ symfony cloud:deploy

Git リポジトリにプッシュされ、コードはデプロイされます。コマンドの最後に、アクセス可能なドメイン名を一つ持つことになります。

.. index::
    single: Symfony CLI;cloud:url

デプロイがうまくいったかチェックしましょう:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

404 ページになるはずですが、``/images/under-construction.gif`` が表示されます。

Platform.sh 上ではきれいなデフォルトの Symfony のページは表示されません。それは Symfony は環境の機能があり、Platform.sh は、自動的にコードを本番環境としてデプロイしているからです。

.. index::
    single: Symfony CLI;cloud:project:delete

.. tip::

    Platform.sh のプロジェクトを削除したいときは、``cloud:project:delete`` コマンドを使用してください。

.. sidebar:: より深く学ぶために

    * `公式 Symfony レシピ`_ のリポジトリと 自分のレシピをポストできる `コミュニティによるレシピ`_, があります;

    * `Symfony のローカルのWebサーバー`_;

    * `Platform.sh のドキュメント`_.

.. _`これ`: http://clipartmag.com/images/website-under-construction-image-6.gif
.. _`プロジェクトのスケルトン`: https://github.com/symfony/skeleton
.. _`Platform.sh`: https://platform.sh
.. _`公式 Symfony レシピ`: https://github.com/symfony/recipes
.. _`コミュニティによるレシピ`: https://github.com/symfony/recipes-contrib
.. _`Symfony のローカルのWebサーバー`: https://symfony.com/doc/current/setup/symfony_server.html
.. _`Platform.sh のドキュメント`: https://symfony.com/doc/cloud
