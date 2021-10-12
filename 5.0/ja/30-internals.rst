Symfony の内部を知る
==========================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

ここまで長い間 Symfony を使ってアプリケーションを開発してきましたが、ほとんどのコードは Symfony 内部から実行されています。実際書いたコードは200行か300行のコードくらいで、Symfony 内部には数千行のコードがあります。

舞台裏でどうやって動いているか理解したいですね。どうやって動いているかを理解するのに役に立つツールにはいつも感心させられます。初めてデバッガーのステップ実行を使ったときや ``ptrace`` を使ってみたときの記憶は魔法のような感覚でした。

Symfony がどうやって動いているかより理解したくなりませんか？Symfony があなたのアプリケーションを動かす仕組みを調べてみましょう。理論的な視点から HTTP リクエストをどう扱っているかを説明するのではなく、 Blackfire を使って視覚的な表現を使い、さらに高度なトピックを調べてみましょう。

Blackfire で Symfony の内部を理解する
----------------------------------------------

全ての HTTP リクエストは、``public/index.php`` ファイルが受けているのはご存知だと思いますが、次に何が起きるか、どうやってコントローラーが呼ばれるかわかりますか？

Blackfire のブラウザ拡張を使用して、本番の英語のホームページのプロファイリングをしてみましょう:

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

もしくは直接コマンドラインで:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

プロファイルの "Timeline" へ行くと、次のような画面が表示されるはずです:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

タイムラインでは、色の付いたバーにマウスをポインタをホバーすれば、各呼び出しについてのより詳細な情報を見ることができ、Symfony がどうやって動いているかを知ることができます。

* メインエントリーポイントは、``public/index.php`` です;

* ``Kernel::handle()`` メソッドがリクエストを扱います;

* イベントをディスパッチする ``HttpKernel`` を呼び出します;

* 最初のイベントは、 ``RequetEvent`` です;

* ``ControllerResolver::getController()`` メソッドは、やってきた URL からどのコントロラーが呼ばれるか決定します;

* ``ControllerResolver::getArguments()`` メソッドは、コントローラーにどの引数を渡すか決定します（Param Converter はここで呼ばれます）;

* ``ConferenceController::index()`` メソッドが呼ばれます。この呼び出しでほとんどの私たちが書いたコードが実行されます;

* ``ConferenceRepository::findAll()`` メソッドが、データベースから全てのカンファレンスを取得します（``PDO::_construct()`` でデータベース接続をしています）;

* ``Twig\Environment::render()`` メソッドがテンプレートをレンダリングします;

* ``ResponseEvent`` と ``FinishRequestEvent`` がディスパッチされますが、実行速度が早すぎてリスナーが何も登録されていないかのように見えます。

タイムラインは、コードがどうやって動くかを理解するのに良い方法です; 他の誰かが開発したプロジェクトを受け取ったときにとても便利です。

開発環境で、ローカルマシンから同じページをプロファイルしましょう:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

プロファイルを開いてください。リクエストはすぐ終わり、タイムラインはほとんど空のため、コールグラフ画面へリダイレクトされるはずです:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

何が起きているか説明しましょう。HTTPキャッシュが有効化されているので、 Symfony HTTP キャッシュのレイヤーをプロファイルしていしまっているのです。ページはキャッシュ（``HttpCache\Store::restoreResponse()``）にあり HTTP レスポンスはキャッシュから取得されるのでコントローラーは呼ばれることがないのです。

前回のステップでやったように、もう一度  ``public/index.php`` 内のキャッシュレイヤーを無効にしてみてください。今度は全く違ったプロファイルになったはずです:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

主な違いは次の通りです:

* 本番では、実行時間をかなり占める ``TerminateEvent`` は見えません。なぜなら ``TerminateEvent`` は、 Symfony プロファイラーがリクエストを受けた際に集めたデータを格納する責務のイベントだからです;

* ``ConferenceController::index()`` コールの下に ``SubRequestHandler::handle()`` メソッドがあります。このメソッドは ESI をレンダリングします。 ``Profiler::saveProfile()`` が二度呼ばれているのはそのためです。一度目はメインのリクエスト用、もう一つは ESI 用です。

さらに詳しく知るために、タイムラインを使って、コールグラフ画面から同データの他の表示に切り替えたりしてみてください。

ここまで見てきたように、開発環境と本番ではコードはかなり違って実行されます。開発環境では、Symfony プロファイラーがデバッグをしやすいように多くのデータを集めようとしています。そのため、ローカルでも本番環境でプロファイルをした方が良いです。

興味深い実験として、 エラーページや ``/`` ページ（リダイレクトのページ）、API リソースをプロファイルしてみてください。各プロファイルから Symfony がどうやって動いているか、少し知ることができます。例えば、どのクラスのメソッドが呼ばれ、何が時間がかかっていたり、何がすぐ終わっていたりするか、などです。

Blackfire のデバッグアドオンを使用する
----------------------------------------------------

.. index::
    single: Blackfire;Debug Addon

デフォルトでは、Blackfire は、大きいペイロードやグラフを避けるためにあまり重要でないメソッド呼び出しを全て除去します。しかし、デバッグツールとして Blackfire を使用する際にには全ての呼び出しを捨てないでおくことがベターです。デフォルトではできませんが、デバッグのアドオンを入れることで可能になります。

コマンドラインからは、 ``--debug`` フラグを使用してください:

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

本番では、 ``.env.local.php`` という名前のファイルがロードされるのが確認できます:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

SymfonyCloud は、Symfony アプリケーションをデプロイする際に、 最適化を行います。Composer のオートローダーを最適化するのと同じようなものです (``--optimize-autoloader --apcu-autoloader --classmap-authoritative``)。また、  ``.env.local.php`` ファイルによって生成される  ``.env`` ファイルに定義してある環境変数も最適化してリクエスト毎にファイルをパースしないようにします:

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire は コードがどうやってPHP で実行されるかを理解するのにとても役立つツールです。パフォーマンスの向上は、プロファイラーの使い道の1つに過ぎません。
