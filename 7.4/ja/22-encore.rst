Webpack でユーザーインタフェースにスタイリングする
=======================================================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

ここまで、まだユーザーインタフェースのデザインをあまりしてきていません。プロのようにスタイルをするのに、`Webpack`_ ベースのモダンなスタックを使ってみましょう。アプリケーションとの統合をやりやすくするために Symfony touch を追加してください。 *Webpack Encore* をインストールしましょう:

.. code-block:: terminal

    $ symfony composer rem asset-mapper
    $ symfony composer req encore

完全な Webpack 環境は、既に作成されており、適切なデフォルトの設定で ``package.json`` と ``webpack.config.js`` が生成されています。 Webpack を設定するのに Encore の設定をする ``webpack.config.js`` を開いてください。

``package.json`` ファイルに、毎回使う便利なコマンドが定義されています。

``assets`` ディレクトリには、``styles/app.css`` や ``app.js`` のようなプロジェクトのアセットのメインエントリーポイントが入っています。

Sass を使用する
--------------------

.. index::
    single: Sass

生の CSS を使うのではなく、 `Sass`_ を使ってみましょう:

.. code-block:: terminal

    $ mv assets/styles/app.css assets/styles/app.scss

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -6,4 +6,4 @@
      */

     // any CSS you import will output into a single css file (app.css in this case)
    -import './styles/app.css';
    +import './styles/app.scss';

Sass ローダーをインストールしてください:

.. code-block:: terminal

    $ npm install sass sass-loader --save-dev

Webpack で Sass ローダーを有効化してください:

.. code-block:: diff
    :caption: patch_file

    --- i/webpack.config.js
    +++ w/webpack.config.js
    @@ -57,7 +57,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

インストールするパッケージはどうしたら分かるのでしょうか。もしパッケージインストール無しでアセットをビルドしようとすると、Encore は ``.scss`` ファイルをロードするのに必要な依存パッケージをインストールする ``npm install`` コマンドが必要であることをエラーメッセージで表示してくれます。

Bootstrap でレバレッジする
----------------------------------

.. index::
    single: Bootstrap

適切なデフォルト値でレスポンシブな Webサイトをビルドするには、 `Bootstrap`_ のような CSS フレームワークが良いでしょう。Bootstrap をパッケージとしてインストールしてください:

.. code-block:: terminal

    $ npm install bootstrap @popperjs/core bs-custom-file-input --save-dev

CSSファイルで Bootstrap を require してください（ファイルをクリーンアップもしています）:

.. code-block:: diff
    :caption: patch_file

    --- i/assets/styles/app.scss
    +++ w/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

JS ファイルにも同じようにしてください:

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -7,3 +7,7 @@

     // any CSS you import will output into a single css file (app.css in this case)
     import './styles/app.scss';
    +import 'bootstrap';
    +import bsCustomFileInput from 'bs-custom-file-input';
    +
    +bsCustomFileInput.init();

Symfony のフォームシステムは、特別なテーマでBootstrap をネイティブでサポートしていますので、有効にしてください:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

HTML をスタイリングする
--------------------------------

アプリケーションをスタイリングする準備ができましたので、アーカイブをダウンロードし、プロジェクトのルートディレクトリに展開してください:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-7.4.zip', 'guestbook-7.4.zip');"
    $ unzip -o guestbook-7.4.zip
    $ rm guestbook-7.4.zip

テンプレートを見ると、少し Twig にトリックがあるのに気づくと思います。

アセットをビルドする
------------------------------

.. index::
    single: Symfony CLI;run

Webpack 使用することでの主な違いは、アプリケーションがCSS や JS ファイルを直接使うことができないことです。最初に "コンパイル"してあげる必要があります。

開発環境では、 ``encore dev`` コマンドでアセットをコンパイルすることができます:

.. code-block:: terminal

    $ symfony run npm run dev

JS や CSS の変更時にコマンドを毎回実行するのではなく、JS やCSSの変更を検知して、バックグラウンドで実行させましょう:

.. code-block:: terminal
    :class: ignore

    $ symfony run -d npm run watch

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

本番では、 Platform.sh は自動的に Encore を使用するかを検知し、ビルドフェーズでアセットをコンパイルします。

.. sidebar:: より深く学ぶために

    * `Webpack ドキュメント`_;

    * `Symfony Webpack Encore ドキュメント`_;

    * `SymfonyCasts Webpack Encore チュートリアル`_.

.. _`Webpack`: https://webpack.js.org/
.. _`Sass`: https://sass-lang.com/
.. _`Bootstrap`: https://getbootstrap.com/
.. _`Webpack ドキュメント`: https://webpack.js.org/concepts/
.. _`Symfony Webpack Encore ドキュメント`: https://symfony.com/doc/current/frontend.html
.. _`SymfonyCasts Webpack Encore チュートリアル`: https://symfonycasts.com/screencast/webpack-encore
