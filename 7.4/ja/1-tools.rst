作業環境を確認する
===========================

プロジェクトで作業を始める前に、適切な作業環境を持っているかどうか確認する必要があります。これは重要なことです。今の開発者が使うツールセットは、10年前のそれとはずいぶん変わりました。ツールは大きな進化を遂げてきたのです。それらを活用しない手はありません。良い道具を使って、道を開いていきましょう。

この手順は省かないでください。少なくとも Symfony CLI に関する最後のセクションは読んでください。

コンピューター
---------------------

コンピューターが要りますね。macOS、Windows、Linux といった一般的なOSで実行することができるので安心です。Symfony やこれから使うすべてのツールはこれらのOSで利用可能です。

技術の選定
---------------

最良の選択をして、速く動きだしたいと思います。私はこの本のためにあえて技術選定をしました。

`PostgreSQL`_ は、データベースからキュー、キャッシュからセッションストレージまで、すべてにおいて私たちの選択となるでしょう。ほとんどのプロジェクトでは、PostgreSQLは最高のソリューションであり、拡張性が高く、管理するサービスが1つだけでインフラストラクチャをシンプルにすることができます。

本書の最後では、キューのための `RabbitMQ`_  、セッションのための `Redis`_  の使い方について学んでいきます。

IDE
---

.. index:: IDE

お望みであれば Notepad を使うこともできるかもしれませんが、おすすめしません。

私は昔 Textmate で仕事をしていましたが、今はもう違います。「本物の」IDEを使う快適さはきわめて有用であり、他では代えられません。オートコンプリート、 自動的にソートされ追加される ``use`` 文、あるファイルから別のファイルへのジャンプ等、生産性を高めてくれる多くの機能を持っています。

`Visual Studio Code`_ か `PhpStorm`_ を使うのが良いでしょう。前者は無料で、後者は有料ですが Symfony とのより良い統合機能を持っています。（ `Symfony Support Plugin`_ のおかげです）。あなた次第です。私が使っている IDE がどちらなのかを知っておきたいでしょうね。私はこの本を Visual Studio Code で書いています。

ターミナル
---------------

.. index:: Terminal

IDEからコマンドラインに切り替えることがよくあります。IDEのビルトインターミナルを使うこともできますが、私はそれではなくて、より多くの作業スペースを使うことのできる実際のターミナルの方を好んで使っています。

Linux には ``Terminal`` がビルトインであります。macOS なら `iTerm2`_ を使いましょう。Windows なら、 `Hyper`_ がうまく機能します。

Git
---

.. index:: Git

バージョン管理には今では誰でも使っている `Git`_ を使います。

Windows なら、 `Git bash`_ をインストールしてください。

Git の一般的な操作、 ``git clone`` 、 ``git log`` 、 ``git show`` 、 ``git diff`` 、 ``git checkout`` … などの実行方法をおさえておいてください。

PHP
---

.. index::
    single: PHP
    single: PHP extensions

サービスに Docker を使うことになりますが、私はローカルコンピューターにPHPをインストールして使うのが好きです。パフォーマンス、安定性、シンプルさがその理由です。やり方が古いと言われてしまうかもしれませんが、ローカルインストールした PHPと Docker サービスを組み合わせるやり方が理想的だと考えています。

Use PHP 8.3 and check that the following PHP extensions are installed or
install them now: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``,
``openssl``, ``sodium``, and ``iconv``. Optionally install ``redis``, ``curl``,
and ``zip`` as well.

現在有効になっている拡張は ``php -m`` で確認できます。

プラットフォームがサポートしていれば ``php-fpm`` も用意して下さい。  ``php-cgi`` でも同等のことができます。

Composer
--------

.. index:: Composer

依存関係の管理は今や Symfony プロジェクトのすべてと言えるほど重要なものです。PHPのパッケージ管理ツール `Composer`_ の最新バージョンを入手してください。

Composer に慣れていない場合は、Composer に関してドキュメントをじっくりと読んでください。

.. tip::

    コマンド名をフルワード入力でタイピングする必要はありません： ``composer req`` は ``composer require`` と同じことですし、 ``composer rem`` を ``composer remove`` の代わりに使えば良いです。他も同様です。

NodeJS
------

JavaScriptのコードはあまり書きませんが、アセットの管理を行うためにJavaScript/NodeJSを利用します。 `NodeJS`_ がインストールされているか確認してください。

Docker と Docker Compose
-------------------------

.. index:: Docker,Docker Compose

サービスは Docker と Docker Compose を使って管理されます。 `それらをインストールして`_  、Docker を起動してください。Docker を初めて使うのであれば、ツールに慣れておきましょう。とまどう必要はありません。使い方はとても簡単ですからどうか安心してください。凝った設定や複雑なセットアップは一切出てきません。

Symfony CLI
-----------

.. index:: Symfony CLI

最後ですが重要なところとして、 ``symfony`` コマンドを使って生産性を高めます。ローカルWebサーバーの提供から、完全な Docker 統合、 Upsun を使ったクラウドのサポートまで、大幅な時間短縮を実現できます。

`Symfony CLI`_  をインストールしてください。

HTTPS をローカルで使うため、`認証局(CA)もインストールして`_ 、TLSサポートを有効にする必要があります。次のコマンドを実行してください:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: terminal
    :class: ignore

    $ symfony server:ca:install

次のコマンドを実行して、コンピューターに必要なすべての要件が満たされていることを確認します:

.. code-block:: terminal
    :class: ignore

    $ symfony book:check-requirements

もし複雑なことをしたい場合は、 `Symfony proxy`_ を実行することもできます。オプションですが、末尾に  ``.wip`` を付したローカルドメイン名をプロジェクトで取得することができます。

ターミナルでコマンドを実行する際は、ほとんどの場合でプレフィックス ``symfony`` を付けることになります。たとえば、普通の ``composer`` ではなく、 ``symfony composer`` を、``./bin/console`` ではなく ``symfony console`` を使う、といった具合です。

その主な理由は、Symfony CLI が Docker で実行されるサービスに対していくつかの環境変数を自動的に設定するためです。環境変数はローカルWebサーバーに自動登録され、HTTPリクエストで利用できるようになります。CLIで  ``symfony`` を使えば、どの環境でも同じように動作することを保証できるわけです。

さらに、Symfony CLI はプロジェクトに「最良の」PHPバージョンを自動的に選択します。

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Redis`: https://redis.io/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Composer`: https://getcomposer.org/
.. _`NodeJS`: https://nodejs.org/
.. _`それらをインストールして`: https://docs.docker.com/install/
.. _`Symfony CLI`: https://symfony.com/download
.. _`認証局(CA)もインストールして`: https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls
.. _`Symfony Support Plugin`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
.. _`Symfony proxy`: https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy
