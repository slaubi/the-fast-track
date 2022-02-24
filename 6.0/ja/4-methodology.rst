メソドロジーを適用する
=================================

教えることは何度も同じことを繰り返すことですが、ここでは行いません。各ステップの最後には、あなたは、自分の作業を、セーブすることができるようになるでしょう。Webサイトで、あたかも ``Ctrl+S`` で保存するかのように。

Git の実行
-------------

.. index::
    single: Git;add
    single: Git;commit

各ステップの最後では、あなたの変更をコミットするのを忘れないでください。

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Symfony が ``.gitignore`` ファイルを管理しているので、安全に"すべて"を追加することができます。そして、各パッケージにもっと設定を追加することができます。現在の内容を見てみてください。

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

変わったマーカーは、Symfomy Flex によりが追加されていますが、依存をアンインストールする際に何を除去したらよいか示すためです。退屈な作業は Symfony が代行してくれるようにしています。

あなたのリポジトリを GitHub や GitLab もしくは Bitbucket などのサーバにプッシュしてみてください。

.. note::

    Platform.sh でデプロイするのであれば、 Git リポジトリのコピーが既にあるはずです。 ``cloud:deploy`` コマンドを使っているならPlatform.shは裏側でGitを使っているからです。しかし、Platform.shのGitレポジトリには頼らないでください。これはデプロイのためのものであって、バックアップのためのものではありませんから。

プロダクトを継続的にデプロイすること
------------------------------------------------------

.. index::
    single: Symfony CLI;cloud:deploy

もう一つの良い習慣は、デプロイを頻繁に行うことです。各ステップの最後にデプロイをするのは良い間隔です。

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy
