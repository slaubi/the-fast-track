非同期にする
==================

.. index::
    single: Async

フォーム投稿時にスパムの判定をするのには多少問題があります。例えば、 Akismet API に遅延の問題があったときに、私たちの Web サイトも遅くなってしまいます。さらに、タイムアウトされてしまったり、Akismet API に問題があったときには、コメントを失ってしまうかもしれません。

公開することなく投稿されたデータを保存して、レスポンスを早く返すことが理想とするところです。そのためにスパムのチェックとは独立して実行します。

コメントにフラグを付ける
------------------------------------

.. index::
    single: Command;make:entity

コメントに ``submitted``, ``spam``, ``published`` という ``state`` を追加する必要があります。

``Comment`` クラスに ``state`` プロパティを追加しましょう:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

データベースマイグレーションを追加する:

.. code-block:: bash

    $ symfony console make:migration

既に登録されているコメント全てに、デフォルトの値として ``published`` を指定するようにマイグレーションを修正してください:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version20200714155905 extends AbstractMigration
         public function up(Schema $schema) : void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255)');
    +        $this->addSql("UPDATE comment SET state='published'");
    +        $this->addSql('ALTER TABLE comment ALTER COLUMN state SET NOT NULL');
         }

         public function down(Schema $schema) : void

.. index::
    single: Command;doctrine:migrations:migrate

データベースをマイグレートする:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

また、 ``state`` のデフォルトの値を ``submitted`` としてセットされていることを確認してください:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -55,9 +55,9 @@ class Comment
         private $photoFilename;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, options={"default": "submitted"})
          */
    -    private $state;
    +    private $state = 'submitted';

         public function __toString(): string
         {

EasyAdmin の設定を変更してコメントの状態(state)を見ることができるようにしましょう。

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/easy_admin.yaml
    +++ b/config/packages/easy_admin.yaml
    @@ -18,6 +18,7 @@ easy_admin:
                         - author
                         - { property: 'email', type: 'email' }
                         - { property: 'photoFilename', type: 'image', 'base_path': "/uploads/photos", label: 'Photo' }
    +                    - state
                         - { property: 'createdAt', type: 'datetime' }
                     sort: ['createdAt', 'ASC']
                     filters: ['conference']
    @@ -26,5 +27,6 @@ easy_admin:
                         - { property: 'conference' }
                         - { property: 'createdAt', type: datetime, type_options: { disabled: true } }
                         - 'author'
    +                    - { property: 'state' }
                         - { property: 'email', type: 'email' }
                         - text

フィクスチャに ``state`` をセットして、テストコードを修正しましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/src/DataFixtures/AppFixtures.php
    +++ b/src/DataFixtures/AppFixtures.php
    @@ -37,8 +37,16 @@ class AppFixtures extends Fixture
             $comment1->setAuthor('Fabien');
             $comment1->setEmail('fabien@example.com');
             $comment1->setText('This was a great conference.');
    +        $comment1->setState('published');
             $manager->persist($comment1);

    +        $comment2 = new Comment();
    +        $comment2->setConference($amsterdam);
    +        $comment2->setAuthor('Lucas');
    +        $comment2->setEmail('lucas@example.com');
    +        $comment2->setText('I think this one is going to be moderated.');
    +        $manager->persist($comment2);
    +
             $admin = new Admin();
             $admin->setRoles(['ROLE_ADMIN']);
             $admin->setUsername('admin');

.. index::
    single: Test;Container
    single: Container;Test

コントローラーのテストでは、バリデーションをシミュレートします:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -2,6 +2,8 @@

     namespace App\Tests\Controller;

    +use App\Repository\CommentRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

     class ConferenceControllerTest extends WebTestCase
    @@ -22,10 +24,16 @@ class ConferenceControllerTest extends WebTestCase
             $client->submitForm('Submit', [
                 'comment_form[author]' => 'Fabien',
                 'comment_form[text]' => 'Some feedback from an automated functional test',
    -            'comment_form[email]' => 'me@automat.ed',
    +            'comment_form[email]' => $email = 'me@automat.ed',
                 'comment_form[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
             ]);
             $this->assertResponseRedirects();
    +
    +        // simulate comment validation
    +        $comment = self::$container->get(CommentRepository::class)->findOneByEmail($email);
    +        $comment->setState('published');
    +        self::$container->get(EntityManagerInterface::class)->flush();
    +
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }

PHPUnit のテストからは ``self::$container->get()`` を使えば、全てのサービス（非公開なサービスも含めて）を取得することができます。

メッセンジャーを理解する
------------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Symfony で非同期処理を管理するために、メッセンジャーコンポーネントを使用します:

.. code-block:: bash

    $ symfony composer req messenger

非同期処理が必要な際に、 *メッセージ* を *メッセージバス* に送ってください。このバスは *キュー* に格納され、処理を止めることなくすぐリターンされます。

*コンシューマー* は、継続的ににバックグラウンドで動いており、キューにある新しいメッセージを読み、そのメッセージに関連したロジックを実行します。コンシューマーは、Web アプリケーションと同じサーバーでも別のサーバーにあっても動作します。

レスポンスがない以外は、HTTP リクエストを処理するときととても似ています。

メッセージハンドラーをコーディングする
---------------------------------------------------------

メッセージはデータオブジェクトのクラスで、ロジックを持つべきではありません。シリアライズされ、キューに格納されます。 "シンプルな"シリアライズ可能なデータのみを格納しましょう。

``CommentMessage`` クラスを作成する:

.. code-block:: php
    :caption: src/Message/CommentMessage.php

    namespace App\Message;

    class CommentMessage
    {
        private $id;
        private $context;

        public function __construct(int $id, array $context = [])
        {
            $this->id = $id;
            $this->context = $context;
        }

        public function getId(): int
        {
            return $this->id;
        }

        public function getContext(): array
        {
            return $this->context;
        }
    }

メッセンジャーを使う際には、コントローラーではなくメッセージハンドラーが処理を担います。

``App\MessageHandler`` ネームスペース以下に ``CommentMessage`` メッセージを処理する ``CommentMessageHandler`` クラスを作成してください:

.. code-block:: php
    :caption: src/MessageHandler/CommentMessageHandler.php

    namespace App\MessageHandler;

    use App\Message\CommentMessage;
    use App\Repository\CommentRepository;
    use App\SpamChecker;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Component\Messenger\Handler\MessageHandlerInterface;

    class CommentMessageHandler implements MessageHandlerInterface
    {
        private $spamChecker;
        private $entityManager;
        private $commentRepository;

        public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository)
        {
            $this->entityManager = $entityManager;
            $this->spamChecker = $spamChecker;
            $this->commentRepository = $commentRepository;
        }

        public function __invoke(CommentMessage $message)
        {
            $comment = $this->commentRepository->find($message->getId());
            if (!$comment) {
                return;
            }

            if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
                $comment->setState('spam');
            } else {
                $comment->setState('published');
            }

            $this->entityManager->flush();
        }
    }

``MessageHandlerInterface`` は、 *マーカー* インターフェースです。このインターフェースは、メッセンジャーハンドラのクラスを設定し Symfony の自動登録を行います。規約では、ハンドラーのロジックは、 ``__invoke()`` メソッドに書きます。このメソッドの引数の ``CommentMessage`` 型宣言から、どのクラスを処理するのか知ることができます。

コントローラーを修正します:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -5,14 +5,15 @@ namespace App\Controller;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use App\Form\CommentFormType;
    +use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    -use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Twig\Environment;

    @@ -20,11 +21,13 @@ class ConferenceController extends AbstractController
     {
         private $twig;
         private $entityManager;
    +    private $bus;

    -    public function __construct(Environment $twig, EntityManagerInterface $entityManager)
    +    public function __construct(Environment $twig, EntityManagerInterface $entityManager, MessageBusInterface $bus)
         {
             $this->twig = $twig;
             $this->entityManager = $entityManager;
    +        $this->bus = $bus;
         }

         /**
    @@ -40,7 +43,7 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -58,6 +61,7 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +            $this->entityManager->flush();

                 $context = [
                     'user_ip' => $request->getClientIp(),
    @@ -65,11 +69,8 @@ class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    -                throw new \RuntimeException('Blatant spam, go away!');
    -            }

    -            $this->entityManager->flush();
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

スパムチェッカーに依存するのではなく、メッセージバスにディスパッチするようになりました。そして、ハンドラーにどう処理するかを決めさせます。

コントローラーとスパムチェックを隔離し、ロジックを新しいクラスのハンドラーに移動しました。バスの良いユースケースです。コードをテストして、動作するか確認してください。全て同期的に実行されますが、コードは、 "ベター"になっています。

表示されるコメントを制限する
------------------------------------------

表示ロジックを変更して、公開されていないコメントをフロントエンドへ表示しないようにしましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -27,7 +27,9 @@ class CommentRepository extends ServiceEntityRepository
         {
             $query = $this->createQueryBuilder('c')
                 ->andWhere('c.conference = :conference')
    +            ->andWhere('c.state = :state')
                 ->setParameter('conference', $conference)
    +            ->setParameter('state', 'published')
                 ->orderBy('c.createdAt', 'DESC')
                 ->setMaxResults(self::PAGINATOR_PER_PAGE)
                 ->setFirstResult($offset)

実際に非同期にする
---------------------------

.. index::
    single: RabbitMQ

デフォルトでは、ハンドラは、同期的に処理します。非同期にするために、``config/packages/messenger.yaml`` の設定ファイルに、ハンドラがどのキューを使用するかを明示的に設定してください:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,10 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            # async: '%env(MESSENGER_TRANSPORT_DSN)%'
    +            async: '%env(RABBITMQ_DSN)%'
                 # failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:
                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

この設定では、バスに ``App\Message\CommentMessage`` のインスタンスを ``非同期`` にキューへ送るようにしています。 接続情報は環境変数 ``RABBITMQ_DSN`` に格納したDSNで定義されています。

Adding RabbitMQ to the Docker Stack
-----------------------------------

.. index::
    single: Docker;RabbitMQ

As you might have guessed, we are going to use RabbitMQ:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -12,3 +12,7 @@ services:
         redis:
             image: redis:5-alpine
             ports: [6379]
    +
    +    rabbitmq:
    +        image: rabbitmq:3.7-management
    +        ports: [5672, 15672]

Restarting Docker Services
--------------------------

To force Docker Compose to take the RabbitMQ container into account, stop the containers and restart them:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

.. sidebar:: データベースのデータのダンプとリストア

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

メッセージを取得実行する
------------------------------------

新しくコメントを投稿しても、スパムチェッカーは呼ばれなくなりました。 ``getSpamScore()`` メソッドで ``error_log()`` 関数を追加して確認してみてください。代わりに RabbitMQ にメッセージが入るようになったので、他のプロセスから取得され実行される準備ができました。

.. index::
    single: Command;messenger:consume

もうお分かりかもしれませんが、Symfony は、メッセージを取得し、実行するコマンドがビルトインされていますので、実行してみましょう:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

コマンドを実行するとすぐに、コメント送信でディスパッチされたメッセージが取得され、実行されるはずです:

.. code-block:: text
    :class: ignore

     [OK] Consuming messages from transports "async".

     // The worker will automatically exit once it has received a stop signal via the messenger:stop-workers command.

     // Quit the worker with CONTROL-C.

    11:30:20 INFO      [messenger] Received message App\Message\CommentMessage ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
    11:30:20 INFO      [http_client] Request: "POST https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [http_client] Response: "200 https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [messenger] Message App\Message\CommentMessage handled by App\MessageHandler\CommentMessageHandler::__invoke ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage","handler" => "App\MessageHandler\CommentMessageHandler::__invoke"]
    11:30:20 INFO      [messenger] App\Message\CommentMessage was handled successfully (acknowledging to transport). ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]

メッセージ取得実行の処理がログに書かれますが、 ``--vv`` フラグを渡すことでコンソールに即時的なフィードバックを得ることができます。さらに、Akismet Akismet API の呼び出しを探すこともできます。

メッセージの取得実行は、``Ctrl+C`` でストップします。

Exploring the RabbitMQ Web Management Interface
-----------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

If you want to see queues and messages flowing through RabbitMQ, open its web management interface:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Or from the web debug toolbar:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Use ``guest``/``guest`` to login to the RabbitMQ management UI:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

ワーカーをバックグラウンドで実行する
------------------------------------------------------

コメントを投稿した際に、毎回メッセージ取得の起動と停止を行うのではなく、ターミナルのウィンドウやタブを開くことなく、継続的に実行するようにしましょう。

Symfony CLI は、 ``run`` コマンドに ``-d`` フラグを付けることでデーモンとすることができ、こういったバックグラウンドで実行するコマンドを管理することができます。

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

メッセージ取得実行をもう一度走らせてください。今度はバックグラウンドで送信しましょう:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

``--watch`` オプションを付けることで、``config/``, ``src/``, ``templates/``, ``vendor/`` ディレクトリ内のファイルシステムに変更があった際に、Symfony にコマンドをリスタートさせることができます。

.. note::

    ``server:log`` でメッセージを重複させたくない際は、``--vv`` オプションは使用しないでください（ログされたメッセージとコンソールのメッセージ）。

メモリ制限やバグなどでメッセージの取得実行が停止した際は、自動的に再起動します。また、メッセージの取得実行の失敗が暴走した際は、 Symfony CLI は処理を停止します。

.. index::
    single: Symfony CLI;server:log

``symfony server:log`` コマンドで、PHP やWebサーバー、アプリケーションの全てのログのストリームを見ることができます:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

``server:status`` コマンドを使えば、現在のプロジェクトで管理されているバックグランドのワーカーの全ての一覧を表示できます:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

ワーカーを停止するには、Webサーバーを止めるか、``server:status`` コマンドで得られる PID をキルしてください:

.. code-block:: bash
    :class: ignore

    $ kill 15774

メッセージの失敗をリトライする
---------------------------------------------

メッセージ取得実行の際に、Akismet が落ちていたらどうしますか？コメントの投稿者には何も影響はありませんが、メッセージを失うことになり、スパムはチェックされません。

メッセンジャーには、メッセージのハンドリングで例外になったらリトライする機構があるので、設定してみましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,17 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            async: '%env(RABBITMQ_DSN)%'
    -            # failed: 'doctrine://default?queue_name=failed'
    +            async:
    +                dsn: '%env(RABBITMQ_DSN)%'
    +                retry_strategy:
    +                    max_retries: 3
    +                    multiplier: 2
    +
    +            failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

    +        failure_transport: failed
    +
             routing:
                 # Route your messages to the transports
                 App\Message\CommentMessage: async

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

メッセージのハンドリングに問題が起きた際に、メッセージの取得実行は諦めるまでに3回リトライをします。ただし、メッセージを廃棄するのではなく、 ``failed`` キューとして、Doctrine データベースなどのより恒久的なストレージに格納します。

失敗したメッセージを調べ、再実行するには次のコマンドを使用します:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Deploying RabbitMQ
------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Adding RabbitMQ to the production servers can be done by adding it to the list of services:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -5,3 +5,8 @@ db:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

Reference it in the web container configuration as well and enable the ``amqp`` PHP extension:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - amqp
             - redis
             - pdo_pgsql
             - apcu
    @@ -26,6 +27,7 @@ disk: 512
     relationships:
         database: "db:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"

     web:
         locations:

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

When the RabbitMQ service is installed on a project, you can access its web management interface by opening the tunnel first:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

SymfonyCloud でワーカーを実行する
-------------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

RabbitMQ からメッセージ取得実行をするには、 ``messenger:consume`` コマンドを継続的に実行する必要があります。これは SymfonyCloud の *ワーカー* の役割です:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -54,3 +54,8 @@ hooks:
             set -x -e

             (>&2 symfony-deploy)
    +
    +workers:
    +    messages:
    +        commands:
    +            start: symfony console messenger:consume async -vv --time-limit=3600 --memory-limit=128M

Symfony CLI のように、SymfonyCloud マネージャーはログをリスタートします。

.. index::
    single: Symfony CLI;logs

ワーカーのログを取得するには、以下のようにしてください:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: より深く学ぶために

    * `SymfonyCasts Messenger チュートリアル <https://symfonycasts.com/screencast/messenger>`_;

    * `Enterprise サービスバス <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ アーキテクチャと `CQRS パターン <https://martinfowler.com/bliki/CQRS.html>`_;

    * `Symfony Messenger ドキュメント <https://symfony.com/doc/current/messenger.html>`_;

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
