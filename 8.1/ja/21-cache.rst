パフォーマンス向上のためにキャッシュする
============================================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

パフォーマンスに関する問題はよく訪れます。典型的な例だと、データベースのインデックスの欠如や、ページあたりの大量のSQL発行などです。空のデータベースでは発生しませんが、トラフィックが増えデータが増加すると、発生する可能性があります。

HTTPキャッシュヘッダーの追加
----------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

HTTPキャッシュを利用することは、ほとんど労力をかけずにエンドユーザーへのパフォーマンス向上を最大限に引き出せるすぐれた戦略です。本番環境でリバースプロキシキャッシュを追加して有効にし、`CDN`_ を使用してキャッシュすることで、さらに優れたパフォーマンスを実現します。

ホームページを1時間キャッシュしてみましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -14,6 +14,7 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\Cache;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Messenger\MessageBusInterface;
    @@ -27,6 +28,7 @@ final class ConferenceController extends AbstractController
         ) {
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {

``#[Cache]`` アトリビュートは、 ``smaxage`` 引数を通じてリバースプロキシのキャッシュ有効期限を設定します。ブラウザのキャッシュを制御するには ``maxage`` を使用します。時間は秒単位で設定します。（1時間 = 60分 = 3,600秒）そして、ルーティングやレートリミットと同じように、キャッシュポリシーは適用される場所、つまりコントローラー上で宣言されます。

カンファレンスページのキャッシュはより動的なため、難しいです。誰でもいつでもコメントを追加できますが、誰もが1時間待つことを望んでいません。そのような場合は、 *HTTP validation* を利用します。

Symfony HTTP Cache Kernelをアクティベートする
-------------------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Symfony HTTP リバースプロキシを有効化して、HTTPキャッシュをテストできますが、 "開発" 環境のみで利用してください。（"プロダクション" 環境ではもっと強力な方法を使います） :

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

本格的なHTTPリバースプロキシであることに加え、Symfony HTTPリバースプロキシ（ ``HttpCache`` クラスを介します）はHTTPヘッダーとして有用なデバッグ情報を追加します。これは、設定したキャッシュヘッダーの検証に大いに役立ちます。

ホームページ上で確認する:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

最初のリクエストに対して、キャッシュサーバーはキャッシュが ``miss`` で、レスポンスをキャッシュするために ``store`` を実行したことを返します。``cache-control`` ヘッダーをチェックして、設定されたキャッシュ方法を確認しましょう。

以降のリクエストでは、レスポンスがキャッシュされています。（ ``age`` も更新されています。）

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

ESIを使用してSQLリクエストを回避する
---------------------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

``TwigEventListener`` リスナーは、Twigにグローバル変数と一緒に全てのカンファレンスオブジェクトをWebサイトの各ページごとに注入します。それはおそらく、最適化のための大きな狙いどころとなります。

毎日新しいデータを追加するわけではないので、このコードはデータベースから毎回全く同じデータを取得することになります。

Symfony Cacheを使用してカンファレンスの名前とスラッグをキャッシュすることもできますが、可能であればHTTPキャッシュインフラストラクチャへ依存するのが望ましいです。

ページの一部をキャッシュしたい場合は、 サブリクエストを作成して、現在のHTTPリクエストから別にします。 *ESI* はこのユースケースに最適です。ESIはHTTPリクエストの結果を別のリクエストに埋め込むことができます。

カンファレンス情報のHTMLの一部のみを返すコントローラーを作成する:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,14 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(

対応するテンプレートを作成する:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

``/conference_header`` にアクセスし、全てが正常に動作しているか確認しましょう。

.. index::
    single: Twig;render
    single: Twig;path

トリックを明らかにする時が来ました！先ほど作成したControllerで呼び出しているTwigテンプレートを更新しましょう。

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,11 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

それだけです。ページを更新しても引き続き同じものが表示されます。

.. tip::

    Symfony プロファイラの"Request / Response" を開き、メインリクエストとサブリクエストの詳細を確認します。

これで、ブラウザでページにアクセスするたびに、ヘッダー用とメインページ用の2つのHTTPリクエストが実行されます。 パフォーマンスが低下しました。 おめでとう！

現在、カンファレンスヘッダーのHTTPコールはSymfonyによって内部的に行われています。よってHTTPラウンドトリップは関係ありません。これはHTTPキャッシュヘッダーを使う方法がないことも意味します。

ESIを使って *リアルな* HTTPコールへ変換しましょう。

まずは、ESIサポートを有効化します:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

つぎに、 ``render`` のかわりに ``render_esi`` を使用します :

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,7 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

SymfonyがESIの処理方法を知っているリバースプロキシを検出した場合、自動的にESIサポートを有効にします（そうでない場合、フォールバックしてサブリクエストを同期的にレンダリングします）。

SymfonyリバースプロキシはESIをサポートしているか、ログを確認しましょう（最初にキャッシュを削除します。次節の"キャッシュ削除" を参照ください）。

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

数回再読み込みします。 ``/`` レスポンスはキャッシュされていますが ``/conference_header`` はキャッシュされていません。ページ全体をキャッシュしながら一部を動的に保つことを成し遂げました！

しかしながら、これは私たちが望むものではありません。他のページとは別に、ヘッダーページを１時間キャッシュしましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {

キャッシュは両方のリクエストで有効になりました:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

``x-symfony-cache`` ヘッダーは2つの要素を含みます。メインの ``/`` リクエストとサブリクエスト（ESIの ``conference_header`` ）です。両方ともキャッシュ（ ``fresh`` ）にあります。

キャッシュ設定は、メインページやESIとは異なる設定を行うこともできます。 "about" ページは、キャッシュに1週間保存し、1時間ごとにヘッダーを更新することもできます。

不要になったリスナーを削除します:

.. code-block:: terminal

    $ rm src/EventListener/TwigEventListener.php

テストのためのHTTPキャッシュ削除
----------------------------------------------

ブラウザでのテスト、もしくは自動化したテストには、キャッシュされている箇所は多少難しくなります。

``var/cache/dev/http_cache`` ディレクトリ内を全て消すことで、全てのHTTPキャッシュを削除することができます。

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

一部のURLのみキャッシュを無効にする場合や、ファンクショナルテストでキャッシュを無効化したい場合、前述のやり方ではうまく機能しません。小さな管理画面を追加して、いくつかのURLをキャッシュ無効にしてみましょう。

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -20,6 +20,8 @@ security:
                     login_path: app_login
                     check_path: app_login
                     enable_csrf: true
    +            http_basic: { realm: Admin Area }
    +            entry_point: form_login
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -47,4 +49,16 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]));
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

新しいコントローラーは ``PURGE`` HTTPメソッドに制限されています。このメソッドはHTTPの標準メソッドではありませんが、キャッシュを無効にするために広く使われています。

デフォルトでは、ルーティングのパラメーターに ``/`` を含めることはできません。最後のルーティングパラメーターに  ``uri`` などで ( ``.*`` ) を設定することで、この制限をオーバーライドすることができます。

``HttpCache`` インスタンスを取得する方法も少し奇妙に見えます。"実際の"クラスにアクセスできないため、匿名クラスを使います。 ``HttpCache`` インスタンスはカーネルをラップしており、これは本来であればキャッシュレイヤーを認識しません。

cURLでの呼び出しを介して、ホームページとカンファレンスヘッダーを無効にします:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` サブコマンドはローカルWebサーバーの現在のURLを返します。

.. note::

    コントローラーはコードで参照されることがないため、ルート名がありません。

開発環境でHTTPキャッシュを無効にする
------------------------------------

HTTPキャッシュは、キャッシュヘッダーをバリデートしたり、古いエントリーをパージする方法を学ぶのに大いに役立ちました。しかし、開発環境でリバースプロキシを有効にしたままにするのは普通ではなく、すぐに邪魔になります。コードを書き換えている間もレスポンスはキャッシュから返されますし、HttpCache がファイルレスポンスを扱う際の古くからの制限により、一部のベンダーアセットが空のボディで返されることさえあります。

すべてバリデートできたので、無効にしましょう。本番環境では Varnish が引き継ぎます:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -14,7 +14,3 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    -
    -when@dev:
    -    framework:
    -        http_cache: true

プレフィクスで同様のルーティングをグループ化する
------------------------------------------------------------------------

.. index::
    single: Attributes;Route

管理画面コントローラーには同じ ``/admin`` プレフィクスを持ったルートがあります。全てのルートで繰り返さず、クラス自体にプレフィクスの設定を行ってルートをリファクタリングします。

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -24,7 +25,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -50,7 +51,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

CPU/メモリ集中操作をキャッシュする
-------------------------------------------------

.. index::
    single: Process
    single: Components;Process

CPUやメモリーを集中的に使用するアルゴリズムはこのWebサイト上にはありません。 *ローカルキャッシュ* について説明するために、現在作業中のステップ（正確には、現在のGitコミットにつけられたGitタグ名）を表示するコマンドを作りましょう。

Symfony Processコンポーネントはコマンドを実行して結果を取得できます（標準出力およびエラー出力）。

コマンドを実装します:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand
    {
        public function __invoke(OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Cache
    single: Components;Cache

コマンドの出力を数分間キャッシュするにはどうすればよいでしょうか？Symfony Cacheを利用します。

Symfony は、コントローラーの引数と同じように、コマンドの ``__invoke()`` メソッドにタイプヒントされたサービスをインジェクトします。キャッシュのロジックでコードをラップします:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/StepInfoCommand.php
    +++ w/src/Command/StepInfoCommand.php
    @@ -6,15 +6,21 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand
     {
    -    public function __invoke(OutputInterface $output): int
    +    public function __invoke(OutputInterface $output, CacheInterface $cache): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return Command::SUCCESS;
         }

プロセスは現在、 ``app.current_step`` がキャッシュされていない時のみ呼び出せます。

パフォーマンスをプロファイリングして比較する
------------------------------------------------------------------

盲目的にキャッシュを追加しないでください。キャッシュを追加することで複雑さが増えるということを覚えておいてください。推測で速度を判断するのは大変危険で、キャッシュを使うことでアプリケーションを遅くする状況に陥る可能性があります。

`Blackfire`_ のようなプロファイリングツールで、キャッシュを追加した場合の影響を常に計測しましょう。

Blackfireを使用したデプロイ前のコードをテストする方法の詳細に関しては、 "パフォーマンス" のステップを参照してください。

本番環境でのリバースプロキシキャッシュを設定する
------------------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

プロダクション環境では、Symfonyリバースプロキシの代わりに "より強力" なVarnishリバースプロキシを利用します。

UpsunにVarnishを追加します:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -6,6 +6,15 @@ services:
         database:
             type: postgresql:16

    +    varnish:
    +        type: varnish:9.0
    +        relationships:
    +            application: 'app:http'
    +        configuration:
    +            vcl: !include
    +                type: string
    +                path: config.vcl
    +
     applications:

.. index::
    single: Upsun;Routes

ルーティングのメインエントリーポイントとしてVarnishを利用します:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -1,5 +1,5 @@
     routes:
    -    "https://{all}/": { type: upstream, upstream: "app:http" }
    +    "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
         "http://{all}/": { type: redirect, to: "https://{all}/" }

最後に、Varnishの設定ファイル ``config.vcl`` を作成します:

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

VarnishでESIサポートを有効にする
-------------------------------------------

Varnish上でESIサポートを有効にするにはリクエストごとに明示的に設定する必要があります。Symfonyは標準の ``Surrogate-Capability`` および ``Surrogate-Control`` ヘッダーを使ってESIサポートを要求します。

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

Varnishキャッシュを削除する
-------------------------------------

本番環境でキャッシュを無効化することは、緊急時か ``master`` 以外のブランチをのぞいて、必要になることはないでしょう。キャッシュを頻繁に削除する必要があるのであれば、（TTLを下げるか有効期限の代わりにバリデーションを使用して）キャッシュの設定を微調整する必要があることを意味します。

ともあれ、キャッシュを無効化するためのVarnishの設定を見てみましょう。

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

実際には、 `Varnish docs`_ に記載されているように、IPアドレスによって制限することになるでしょう。

いくつかのURLを削除します:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

``env:url`` で返されるURLは ``/`` で終わっているため、少し奇妙に感じるかもしれません。

.. sidebar:: より深く学ぶために

    * `Cloudflare`_ グローバルなクラウドプラットフォーム;

    * `Varnish HTTP Cache ドキュメント`_;

    * `ESI仕様`_ および `ESI 開発者向けリソース`_;

    * `HTTP cache validation model`_;

    * `HTTP Cache in Upsun`_.

.. _`Blackfire`: https://blackfire.io/
.. _`Varnish docs`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Varnish HTTP Cache ドキュメント`: https://varnish-cache.org/docs/index.html
.. _`ESI仕様`: https://www.w3.org/TR/esi-lang
.. _`ESI 開発者向けリソース`: https://www.akamai.com/us/en/support/esi.jsp
.. _`HTTP cache validation model`: https://symfony.com/doc/current/http_cache/validation.html
.. _`HTTP Cache in Upsun`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
