パフォーマンスを管理する
====================================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    早すぎる最適化は諸悪の根源です。

過去にこの引用を見たことがあるかもしれませんが、完全な形で引用したいと思います:

.. epigraph::

    だいたい97%くらいのケースでは、小さな効率化に関しては忘れるべきです。早まった最適化は諸悪の根源です。しかしながら、残りのクリティカルな3%内の効率化の機会を捨てないようにしましょう。

    --   Donald Knuth

EコマースのようなWebサイトでは、小さなパフォーマンス改善が効いてくることがあります。ゲストブックアプリケーションが、大人気になったときのこと考えて、パフォーマンスの調べ方を見ていきましょう。

パフォーマンスの最適化を調べる最善の方法は、 *プロファイラー* を使うことです。現時点で最も人気のあるプロファイラーは、  `Blackfire <https://blackfire.io>`_ です（*免責事項*: 私は Blackfire プロジェクトの創設者でもあります）。

Blackfire を使ってみる
----------------------------

Blackfire は複数のパーツから構成されています:

* プロファイルをトリガーする *クライアント* （Blackfire CLI ツールもしくは Google Chrome や Firefox のブラウザ拡張）;

* blackfire.io でデータの表示をするために送る前の、データを準備して集計する *エージェント*;

* PHPコードを測定するPHP 拡張（ *probe* ）

Blackfire を使うには、まず `ユーザー登録 <https://blackfire.io/signup>`_ する必要があります。

次のインストールスクリプトを実行してローカルマシンに Blackfire をインストールしてください:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

このインストーラーは、 Blackfire CLI をダウンロードし、使用可能な PHP バージョンの全てに PHP probe をインストールします（有効にはしていません）。

プロジェクトで PHP Probe を有効化してください:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Web サーバーをリスタートして、PHP に Blackfire をロードさせます:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Blackfire CLI ツールは、あなたの **クライアント** のクレデンシャルで設定する必要があります（あなたの個人アカウント以下のプロジェクトプロファイルに格納されます）。 ``Settings/Credentials`` `ページ <https://blackfire.io/my/settings/credentials>`_  の上部にありますので、次のコマンド内のプレースホルダーを入れ替えて実行してください:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    全てインストールするには、 `公式のインストールガイド <https://blackfire.io/docs/up-and-running/installation>`_ に沿ってください。Blackfire をサーバーにインストールする際に便利です。

Docker に Blackfire のエージェントをセットアップする
---------------------------------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

最後に、 Blackfire エージェントのサービスを Docker Compose のスタックに追加します:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -12,3 +12,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

サーバーと通信するために、あなた個人の **サーバー** クレデンシャルが必要です（これらのプロファイルを格納するクレデンシャルのIDは、各プロジェクト毎に作成することができます）; ``Settings/Credentials`` `ページ <https://blackfire.io/my/settings/credentials>`_ の最下部にあります。ローカルの ``.env.local`` ファイルに書いておきましょう:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

これで新しいコンテナを起動することができます:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Blackfire のインストール失敗を修正する
----------------------------------------------------

プロファイリングの際にエラーになったら、 Blackfire のログレベルを変更し、より詳細の情報が吐かれるようにします:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/php.ini
    +++ b/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Webサーバーをリスタートしてください:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

ログを tail してください:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

もう一度プロファイルして、ログの出力をチェックしてください。


本番で Blackfire を設定する
-----------------------------------

.. index::
    single: SymfonyCloud;Blackfire

全ての SymfonyCloud のプロジェクトでは、デフォルトで Blackfire が使用できます。

*サーバー* のクレデンシャルを、環境変数でセットアップしてください:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

PHP probe を他の PHP 拡張と同じように有効化してください:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - pdo_pgsql
             - apcu

Blackfire 用に Varnish を設定する
----------------------------------------

.. index::
    single: SymfonyCloud;Varnish

プロファイリング開始のためのデプロイをする前に、HTTP キャッシュの Varnish へのバイパスが必要です。バイパスなしでは、 Blackfire は、 PHP アプリケーションまで到達できません。ローカルマシンからのプロファイルのリクエストだけ許可を与えます。

あなたの現在の IP アドレスを調べてください:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

そして、そのIPアドレスを使って Varnish を設定してください:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "a.b.c.d";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

これでデプロイができます。

Webページをプロファイルする
---------------------------------------

.. index::
    single: Profiling;Web Pages

Firefox や Google Chrome の `専用の拡張 <https://blackfire.io/docs/integrations/browsers/index>`_ で、従来の Web ページのプロファイルをすることができます。

ローカルマシーンでは、``config/packages/framework.yaml`` 内で HTTP キャッシュを無効化するのを忘れないでください。有効なままですと、コードのプロファイルではなく、 Symfony の HTTP Cache のプロファイルをすることになってしまいます:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -16,4 +16,4 @@ framework:
         php_errors:
             log: true

    -    http_cache: true
    +    #http_cache: true

本番でアプリケーションのパフォーマンスをより詳細に測るために、  "本番" 環境でもプロファイルすべきです。デフォルトでは、ローカルの環境は、オーバーヘッドの大きい "development" 環境を使用しています（WebデバッグツールバーやSymfony Profilerでデータを集めています）。

.. index::
    single: Symfony CLI;server:prod

ローカルマシンを本番環境にスイッチするには、``.env.local`` ファイルの環境変数 ``APP_ENV`` を変更します:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

もしくは、 ``server:prod`` コマンドを使用することができます:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

プロファイリングのセッションが終わったら、dev へ戻すのを忘れないでください:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

API のリソースをプロファイルする
----------------------------------------------

.. index::
    single: Profiling;API

API や SPA のプロファイリングは、先程インストールした Blackfire CLI ツールから行った方がベターです:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

``blackfire curl`` コマンドは、 `cURL <https://curl.haxx.se/docs/manpage.html>`_ コマンドと同じ引数とオプションを受け付けます。

パフォーマンスを比較する
------------------------------------

"キャッシュ" でのステップでは、コードのパフォーマンスを改善するためにキャッシュレイヤーを追加しましたが、それによるパフォーマンスのインパクトを調べたり、測ったりはしていませんでした。何が早くて何が遅いかを想像でしか見ていなかったので、いくつかの最適化が実際はアプリケーションを遅くしている可能性もあります。

常にプロファイラーで最適化のインパクトを測るべきです。 Blackfire の `比較機能 <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_ を使うと視覚的に見ることができます。

ブラックボックスなファンクショナルテストを書く
---------------------------------------------------------------------

.. index::
    single: Blackfire;Player

Symfony でファンクショナルテストを書く方法を見てきましたが、 `Blackfire player <https://blackfire.io/player>`_ を使えば、ブラウジングのシナリオを書くことができます。新しいコメントを投稿し、開発環境ではメールのリンクで、本番環境では管理者によって、バリデートするシナリオを書いてみましょう。

次の内容で ``.blackfire.yaml`` ファイルを作成してください:

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment_form[author] 'Fabien'
                param comment_form[email] 'me@example.com'
                param comment_form[text] 'Such a good conference!'
                param comment_form[photo] file(fake('image', '/tmp', 400, 300, 'cats'), 'awesome-cat.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    include login
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin/?entity=Comment&action=list')
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Blackfire プレイヤーをダウンロードして、ローカルでシナリオを実行できるようにします:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

開発環境でシナリオを実行してください:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

本番でも実行してください:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Blackfire のシナリオは、 ``--blackfire`` フラグを付けることで、各リクエストでプロファイリングをトリガーし、パフォーマンステストを実行することもできます。

パフォーマンスチェックを自動化する
---------------------------------------------------

パフォーマンスを管理することは既存のコードのパフォーマンスを改善するだけではなく、パフォーマンスが悪くなっていないかチェックすることでもあります。

前のセクションで書いたシナリオは、自動的にCI ワークフローで実行され、また、本番でも定期的に実行されます。

SymfonyCloud では、新しくブランチを作った際、または、本番にデプロイされた際に、`run the scenarios <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ で、新しいコードのパフォーマンスを自動的にチェックします。

.. sidebar:: より深く学ぶために

    * `Blackfire の書籍: PHP のコードパフォーマンスの説明 <https://blackfire.io/book>`_;

    * `SymfonyCasts Blackfire のチュートリアル <https://symfonycasts.com/screencast/blackfire>`_.
