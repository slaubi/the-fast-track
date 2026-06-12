ユーザーインターフェイスをスタイリングする
==========================================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

ここまで、ユーザーインターフェイスのデザインに時間をかけてきませんでした。プロのようにスタイリングするために、 *AssetMapper* をベースにしたモダンなスタックを使用します。AssetMapper は、この書籍の最初のステップからアセットを管理してきた Symfony コンポーネントです。

AssetMapper は、モダンな Web 標準を採用しています。JavaScript ファイルと CSS ファイルはそのまま配信され、 *importmap* によって相互に連携されるので、ブラウザはネイティブの *ES モジュール* を直接ロードできます。バンドラーも、ビルドステップも、Node.js も不要です。

プロジェクトのルートディレクトリにある ``importmap.php`` ファイルを見てみてください。アプリケーションで使用される JavaScript パッケージが記述されています。 ``templates/base.html.twig`` で呼ばれている Twig の ``importmap()`` ファンクションが、これらをブラウザに公開します。

Bootstrap を活用する
--------------------

.. index::
    single: Bootstrap

良いデフォルト値で始めて、レスポンシブな Web サイトを構築するためには、 `Bootstrap`_ のような CSS フレームワークが大いに役立ちます。importmap パッケージとしてインストールしてください:

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

このコマンドは、パッケージを ``importmap.php`` に登録し、（依存パッケージの ``@popperjs/core`` と一緒に） ``assets/vendor/`` にダウンロードします。アプリケーションは実行時に CDN に依存しません。

JavaScript のメインエントリーポイントで Bootstrap をインポートしてください（合わせてデフォルトのウェルカムメッセージも削除しています）:

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -5,6 +5,6 @@ import './stimulus_bootstrap.js';
      * This file will be included onto the page via the importmap() Twig function,
      * which should already be in your base.html.twig.
      */
    +import 'bootstrap';
    +import 'bootstrap/dist/css/bootstrap.min.css';
     import './styles/app.css';
    -
    -console.log('This log comes from assets/app.js - welcome to AssetMapper! 🎉');

``app.css`` は Bootstrap のスタイルの *後* にインポートされていることに注意してください。こうすることで、私たちのカスタマイズが優先されます。

Symfony のフォームシステムは、特別なテーマでBootstrap をネイティブでサポートしていますので、有効にしてください:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

HTML をスタイリングする
-----------------------

アプリケーションをスタイリングする準備ができましたので、アーカイブをダウンロードし、プロジェクトのルートディレクトリに展開してください:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

テンプレートを見てみてください。Twig のトリックを 1 つや 2 つ学べるかもしれません。

アセットを配信する
------------------

.. index::
    single: AssetMapper;asset-map:compile

ビルドするものは何もありません。ページをリロードすれば、変更はすぐに反映されます。開発環境では、AssetMapper がアセットファイルを直接配信します。

少し時間をかけて視覚的な変更を発見してみてください。ブラウザで新しいデザインを見てみてください。

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Maker バンドルは、デフォルトで Bootstrap CSS クラスを使用していますので、生成されたログインフォームはスタイルが適用されているはずです:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

本番環境では、Upsun がビルドフェーズで自動的に ``asset-map:compile`` コマンドを実行します。全てのアセットは、ファイル名にバージョンハッシュを付けて ``public/assets/`` にコピーされ、安全で長期間有効な HTTP キャッシュが可能になります。

.. sidebar:: より深く学ぶために

    * `AssetMapper コンポーネントのドキュメント`_;

    * `importmap の仕様`_;

    * `Bootstrap のドキュメント`_.

.. _`Bootstrap`: https://getbootstrap.com/
.. _`AssetMapper コンポーネントのドキュメント`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`importmap の仕様`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`Bootstrap のドキュメント`: https://getbootstrap.com/docs/
