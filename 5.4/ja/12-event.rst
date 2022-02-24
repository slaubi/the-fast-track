イベントをリッスンする
=================================

現在のレイアウトは、ホームページへ戻ったり、個々のカンファレンスページへ移動するナビゲーションヘッダーがありません。

Webサイトのヘッダーを追加する
------------------------------------------

.. index::
    single: Twig;for
    single: Twig;path

全てのWebページに表示されるヘッダーなどのパーツはベースレイアウト内にあった方が良いですね:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -14,6 +14,15 @@
             {% endblock %}
         </head>
         <body>
    +        <header>
    +            <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    +            <ul>
    +            {% for conference in conferences %}
    +                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +            {% endfor %}
    +            </ul>
    +            <hr />
    +        </header>
             {% block body %}{% endblock %}
         </body>
     </html>

レイアウトへこのコードを追加するには、継承する全てのテンプレートで ``conferences`` 変数を定義する必要があります。この変数はコントローラーで作成しテンプレートに渡します。

2つしかコントローラーがないので、次のようにしたくなるかもしれません（この変更はコードに適用しないでください。より良い方法をこの後すぐに学ぶからです）:

.. code-block:: diff
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -29,12 +29,13 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return new Response($this->twig->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,

コントローラーがたくさんあったとするとどうしましょうか？そしてコントローラーを追加する度に同じことをしますか？しかし、それはあまり良いプラクティスではないので、もっと良い方法でした方が良いですね。

Twig はグローバル変数を使うことができます。 *グローバル変数* は、全てのテンプレートで使うことができます。静的な値である必要がありますが、設定ファイルで定義することができます。全てのカンファレンスをTwig のグローバル変数として追加するために、リスナーを追加をしましょう。

Symfony Events を使ってみる
---------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony は Event Dispatcher コンポーネントがビルトインされています。ディスパッチャーは、*リスナー* が、リッスンできる指定のタイミングで、決められた *イベント* をディスパッチします。リスナーはフレームワーク内部へのフックです。

例えば、 HTTPリクエストのライフサイクルにフックされたイベントもあります。リクエストを処理する際の、リクエストが作成された時、コントローラーが実行される時、レスポンスを返す準備ができた時などに、ディスパッチャーは、イベントをディスパッチします。 *リスナー* は、イベントをリッスンすることで、イベントのコンテキストに基づいたロジックを処理することができます。

フレームワークをより汎用に拡張しやすいようにイベントは設計されています。Security, Message, Workflow や Mailer などの Symfony コンポーネントは至るところでイベントを使用しています。

他のビルトインされたイベントとリスナーの例は、コマンドのライフサイクルです: *どんな* コマンド実行前にでもリスナーを作成し、コードを実行させることができます。

パッケージやバンドルはコードを拡張しやすいように自らのイベントをディスパッチすることもできます。

どのイベントをリッスンするかを説明する設定ファイルを作らなくても良いように *サブスクライバー* を作りましょう。サブスクライバーは、 static メソッドの ``getSubscribedEvents()`` で設定を返すことができるリスナーです。こうやってSymfony ディスパッチャーを自動的に登録することができます。

サブスクライバーを実装する
---------------------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Maker バンドルを使ってサブスクライバーを生成しましょう:

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

このコマンドで、リッスンしたいイベントを答えます。コントローラーが呼ばれる直前にディスパッチされる ``Symfony\Component\HttpKernel\Event\ControllerEvent`` イベントを選んでください。Twigがグローバル変数 ``conference`` を参照するのは、コントローラーがテンプレートをレンダーする際ですので、ここが良いタイミングです:

.. code-block:: diff
    :caption: patch_file

    --- a/src/EventSubscriber/TwigEventSubscriber.php
    +++ b/src/EventSubscriber/TwigEventSubscriber.php
    @@ -2,14 +2,25 @@

     namespace App\EventSubscriber;

    +use App\Repository\ConferenceRepository;
     use Symfony\Component\EventDispatcher\EventSubscriberInterface;
     use Symfony\Component\HttpKernel\Event\ControllerEvent;
    +use Twig\Environment;

     class TwigEventSubscriber implements EventSubscriberInterface
     {
    +    private $twig;
    +    private $conferenceRepository;
    +
    +    public function __construct(Environment $twig, ConferenceRepository $conferenceRepository)
    +    {
    +        $this->twig = $twig;
    +        $this->conferenceRepository = $conferenceRepository;
    +    }
    +
         public function onControllerEvent(ControllerEvent $event)
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }

         public static function getSubscribedEvents()

これで新しくコントローラーを追加しても、 Twig 内 ``conference`` 変数が使えるようになりました。

.. note::

    後のステップでよりパフォーマンスを考慮した方法を説明します。

年と都市でカンファレンスをソートする
------------------------------------------------------

年でカンファレンスのリストをソートすることで見やすくなるかもしれません。カンファレンスの一覧を取得してソートするカスタムメソッドを作成することもできますが、 ``findAll()`` メソッドのデフォルトの実装をオーバーライドして、全ての場所でソートされた状態で取得できるようにしてみましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/ConferenceRepository.php
    +++ b/src/Repository/ConferenceRepository.php
    @@ -19,6 +19,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll(): array
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         // /**
         //  * @return Conference[] Returns an array of Conference objects
         //  */

最後のステップとして、Webサイトが次のようになっているか見てみてください:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: より深く学ぶために

    * `Symfony アプリケーションの Request-Response の流れ`_

    * `ビルトインされた Symfony HTTP のイベント`_;

    * `ビルトインされた Symfony Console のイベント`_.

.. _`Symfony アプリケーションの Request-Response の流れ`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`ビルトインされた Symfony HTTP のイベント`: https://symfony.com/doc/current/reference/events.html
.. _`ビルトインされた Symfony Console のイベント`: https://symfony.com/doc/current/components/console/events.html
