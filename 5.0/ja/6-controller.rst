コントローラーを作成する
====================================

.. index::
    single: Controller
    single: Routing;Route

ちょっとごまかしがありますが、私たちのゲストブックのプロジェクトは、本番サーバーで実際に動くようになりました。まだプロジェクトはページが一つもありません。ホームページは 404 エラーページとなっていますので、直してみましょう。

ホームページへ(``http://localhost:8000/``)のような、HTTP リクエストが来ると、 Symfony は *リクエストされたパス* (ここでは ``/``) にマッチする *ルート* を探そうとします。 *ルート* は、リクエストのパスと対応する HTTP *レスポンス* を作成する関数の *PHP の実行可能コード* をリンクします。

これらの実行可能なコードを "コントローラー" と呼びます。 Symfony では、ほとんどのコントローラーは PHP のクラスで実装します。クラスは手動で作成することが可能ですが、もっと早くするため Symfony がやってくれることを見てみましょう。

Maker Bundle で楽をする
----------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

少ない努力でコントローラーを生成するのに ``symfony/maker-bundle`` パッケージを使用することができます:

.. code-block:: bash

    $ symfony composer req maker --dev

Maker Bundle は開発時のみで便利なバンドルで、本番では有効にしたくないので、 ``--dev`` フラグをつけるのを忘れないでください。

Maker Bundle はたくさんのクラスを生成してくれます。この書籍では、常に使うことになります。各 "ジェネレーター" は、コマンドに定義されており、全てのコマンドは、 ``make`` コマンドのネームスペースにあります。

.. index::
    single: Command;list

Symfony Console の ``list`` コマンドは、指定のネームスペース以下の全てのコマンドを一覧で表示します; Maker Bundle で使用可能なすべてのジェネレーターを調べてみてください:

.. code-block:: bash
    :class: ignore

    $ symfony console list make

設定のフォーマットを選ぶ
------------------------------------

プロジェクトの最初のコントローラーを作成する前に、使用したい設定のフォーマットを決める必要があります。Symfony は YAML, XML, PHP, アノテーションを最初からサポートしています。

*関連するパッケージの設定* では、 *YAML* がベストな選択です。*YAML* フォーマットは、 ``config/`` ディレクトリ内で使用されています。多くの場合、新しいパッケージをインストールすると、パッケージのレシピは ``.yaml`` という拡張子の新規ファイルをこのディレクトリに追加します。

*PHP コードに関連する設定* では、コードに隣接して定義することができる *アノテーション* がベターな選択です。例で説明しましょう。リクエストが来ると、設定は Symfony にどのリクエストパスがどのコントローラー（PHPクラス）によって処理するか伝える必要があります。YAML や XML や PHP フォーマットでは、2つのファイルを必要とします（設定ファイルと PHP のコントローラーファイル）。アノテーションを使用すれば、設定はコントローラークラスで直接設定可能です。

アノテーションを使用するために、依存を追加する必要があります:

.. code-block:: bash

    $ symfony composer req annotations

インストールする必要のあるパッケージ名をどうやって判断するか疑問に思ったかもしれません。ほとんどの場合、知る必要はありません。Symfony はエラーメッセージ中にインストールが必要なパッケージを表示しています。 ``annotations`` パッケージのない状態で ``symfony make:controller`` コマンドを実行すると、正しいパッケージをインストールするヒントを含んだ例外を見ることができます。

コントローラーを生成する
------------------------------------

.. index::
    single: Command;make:controller

``make:controller`` コマンドで最初の *コントローラー* を作成しましょう:

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;@Route

このコマンドは ``src/Controller`` ディレクトリ以下に ``ConferenceController`` クラスを作成します。生成されたクラスはちゃんと動くようなボイラープレートが既に入っています:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 10

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

    class ConferenceController extends AbstractController
    {
        /**
         * @Route("/conference", name="conference")
         */
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

``@Route("/conference", name="conference")`` アノテーションは、コントローラーの ``index()`` メソッドを作成します（設定は、コードに隣接しています）。

``/conference`` をブラウザで開くと、このコントローラが実行され、レスポンスが返されます。

ホームページにマッチするようにルートを微調整します:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,7 +9,7 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
         /**
    -     * @Route("/conference", name="conference")
    +     * @Route("/", name="homepage")
          */
         public function index(): Response
         {

コード内でホームページを参照したいときは、ルートの ``名前`` が便利です。 ``/`` パスをハードコードせずに、 ルート名を使いましょう。

デフォルトで表示されるページの代わりに、シンプルな HTML のページを返すようにしましょう:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -13,8 +13,13 @@ class ConferenceController extends AbstractController
          */
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +<html>
    +    <body>
    +        <img src="/images/under-construction.gif" />
    +    </body>
    +</html>
    +EOF
    +        );
         }
     }

ブラウザを更新します:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

コントローラーの主な責務は、リクエストに対応する HTTP ``レスポンス`` を返すことです。

.. _easter-egg:

イースターエッグを追加します
------------------------------------------

どうやってリクエストの情報からレスポンスが作られるかを見るために、 小さな `Easter egg`_ を追加してみましょう。ホームページに ``?hello=Fabien`` のようなクエリー文字列が含まれていたら、挨拶ができるようにしてみましょう:

.. code-block:: diff
    :emphasize-lines: 16
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,6 +3,7 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

    @@ -11,11 +12,17 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/", name="homepage")
          */
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        $greet = '';
    +        if ($name = $request->query->get('hello')) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

Symfony は ``Request`` オブジェクトを通してリクエストされたデータを取得することができます。Symfony は、コントローラーの引数にこの型宣言があると、自動的に渡すことができます。 クエリー文字列にある ``name`` の値を取得して ``<h1>`` のタイトルに追加することができます。

ブラウザで、``/`` へアクセスして、それから ``/?hello=Fabien`` へ変えて、違いを見てみてください。

.. note::

    XSS の問題を避けるために ``htmlspecialchars()`` が呼ばれているのに気づきましたか。適切なテンプレートエンジンへスイッチすると、自動的にサニタイズされます。

また、URL の一部の name を指定することができます:

.. code-block:: diff
    :emphasize-lines: 8,11
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,13 +9,19 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
         /**
    -     * @Route("/", name="homepage")
    +     * @Route("/hello/{name}", name="homepage")
          */
    -    public function index(): Response
    +    public function index(string $name = ''): Response
         {
    +        $greet = '';
    +        if ($name) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

ルーティングの　``{name}`` の部分は、 ダイナミックな *ルートパラメーター* です。ワイルドーカードのようなものです。これでブラウザで ``/hello`` と開いてから ``hello/Fabien`` と変えても同じ結果が出すことができます。コントローラーに *name* と同じ引数（``$name``）を指定することで、 ``{name}`` パラメーターの *値* を取得することができます。

.. sidebar:: より深く学ぶために

    * Symfony の `ルーティング <https://symfony.com/doc/current/routing.html>`_ システム;

    * `SymfonyCasts: ルート、コントローラーとページのチュートリアル <https://symfonycasts.com/screencast/symfony/route-controller>`_;

    * `PHP でのアノテーション <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_;

    * `HttpFoundation <https://symfony.com/doc/current/components/http_foundation.html>`_ コンポーネント;

    * `XSS (クロスサイトスクリプティング) <https://owasp.org/www-community/attacks/xss/>`_ セキュリティ攻撃;

    * `Symfony Routing チートシート <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`Easter egg`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
