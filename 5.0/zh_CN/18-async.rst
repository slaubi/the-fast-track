使用异步
============

.. index::
    single: Async

在处理表单提交时来检查垃圾信息会造成一些问题。如果 Akismet 的 API 变得很慢，我们的网站对用户的响应也会很慢。但更糟的是，如果调用 Akismet 的 API 超时了，或者 API 不可用，那我们可能会丢失评论。

理想情况是，你应该存储提交的数据，但先不发布它，并且立刻返回一个应答。检查垃圾信息可以之后进行。

对评论进行标记
---------------------

.. index::
    single: Command;make:entity

我们要为评论引入 ``state`` 信息，它可以取以下值之一：``submitted``、``spam`` 或 ``published``。

把 ``state`` 属性加入 ``Comment`` 类：

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

创建一个数据库结构迁移：

.. code-block:: bash

    $ symfony console make:migration

修改这个迁移，让已有评论的状态默认设为 ``published``：

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

更新数据库结构：

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

我们也要确保 ``state`` 的默认值设为 ``submitted``：

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

更新 EasyAdmin 的配置，这样我们可以看到评论的状态：

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

别忘了也要更新测试，就是在 fixture 里设置 ``state``：

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

在控制器的测试里，模拟验证：

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

在 PHPUnit 测试中，你可以用 ``self::$container->get()`` 获取容器里的任何服务；你也可以用它访问非公共服务。

理解 Messenger
----------------

.. index::
    single: Messenger
    single: Components;Messenger

在 Symfony 中由 Messenger 组件来管理异步代码：

.. code-block:: bash

    $ symfony composer req messenger

当需要异步执行一些逻辑时，发送一个 *消息* 给 *消息总线*。总线会在一个 *队列* 中存储消息，并且立即返回，这样执行流程就能尽快恢复。

一个 *消费者* 程序会在后台持续运行，它会读取队列里的消息，并且执行相关逻辑。消费者程序可以和 web 应用程序运行在同一个服务器上，也可以运行在另一个服务器上。

它和 HTTP 请求的处理方式很像，只是它不会返回应答。

编写一个消息处理器
---------------------------

一个消息是一个仅包含数据的类，它不含任何逻辑。它会被持久化，然后保存在队列中，所以它只包含“简单”的可序列化数据。

创建 ``CommentMessage`` 类：

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

在 Messenger 的世界里，我们没有控制器，但有消息处理器。

在新的 ``App\MessageHandler`` 命名空间下创建 ``CommentMessageHandler`` 类，它知道如何处理 ``CommentMessage`` 类的消息：

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

``MessageHandlerInterface`` 是一个 *标记* 接口。它帮助 Symfony 自动注册和自动配置该类为一个消息处理器。根据惯例，处理逻辑放在名为 ``__invoke()`` 的方法中。该方法中的 ``CommentMessage`` 类型提示告诉 Messenger 它要处理的类是哪一个。

更新控制器来使用新的系统：

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

我们现在分发一个消息到总线上，而不是依赖于垃圾信息检查器。然后消息处理器会决定怎么处理消息。

这达到了一些意想不到的效果。我们把控制器和垃圾信息检查器进行了解耦，把逻辑移到了消息处理器这个新的类中。这是一个使用总线的完美案例。测试一下代码，它能通过。所有的执行仍然是同步的，但代码大概已经变得“更好”了。

限制展示的评论
---------------------

更新评论的展示逻辑，让未发布的评论不出现在前端：

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

真正走向异步
------------------

.. index::
    single: RabbitMQ

默认情况下，消息处理器都是同步调用。为了使用异步方式，你需要在 ``config/packages/messenger.yaml`` 配置文件中为每个消息处理器显式地配置要用到的队列：

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

配置告诉总线把 ``App\Message\CommentMessage`` 类的实例发送到 ``async`` 队列，这个队列用一个 DSN 定义，它的值储存在 ``RABBITMQ_DSN`` 环境变量中。

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

.. sidebar:: 导出和恢复数据库里的内容

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

用消费者程序处理消息
------------------------------

如果你试着提交一个新评论，垃圾检查器不再被调用。在 ``getSpamScore()`` 方法中加上一行 ``error_log()`` 调用来进行验证。新方案是，消息会在 RabbitMQ 中等待，直到有消费者进程来处理。

.. index::
    single: Command;messenger:consume

你可以想象得到，Symfony 自带了一个消费者命令。现在来运行它：

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

它会立刻处理掉那个表单提交后分发的消息：

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

日志会记录消息的处理，但你可以用 ``-vv`` 选项在终端获得即时反馈。你甚至应该可以在日志中发现调用 Akismet API 的记录。

按 ``Ctrl+C`` 可以停止消费者程序。

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

在后台运行 worker 进程
-----------------------------

我们想要在不打开很多终端的情况下就可以让消费者程序持续运行，而不是每次提交一个评论时打开它，提交完后再立刻关闭它。

``symfony`` 命令在执行 ``run`` 命令时，可以通过守护进程选项（``-d``）来管理这种后台运行的命令或工作进程。

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

再执行一次消费者程序，但在后台程序中发送消息：

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

``--watch`` 选项告诉Symfony，每当 ``config/``、``src/``、``templates/`` 或 ``vendor/`` 这些目录下有文件系统的改动时，这个命令必须重启。

.. note::

    不要使用 ``-vv`` 选项，否则你会在执行 ``server:log`` 时信息会重复（日志信息和控制台信息）。

如果消费者程序因为一些原因停止运行了（内存限制，程序错误等待），它会被自动重启。如果运行消费者程序失败得太频繁，``symfony`` 命令就会放弃运行。

.. index::
    single: Symfony CLI;server:log

日志通过 ``symfony server:log`` 流式输出，它会整合所有来自 PHP、web 服务器和应用程序的其它日志：

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

用 ``server:status`` 命令来列出当前项目所管理的全部后台 worker 进程：

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

要停止一个 worker 进程，需要停止 web 服务器，或者根据 ``server:status`` 命令显示的进程 ID 杀死对应进程：

.. code-block:: bash
    :class: ignore

    $ kill 15774

对失败的消息进行重试
------------------------------

如果处理消息时 Akismet 下线了怎么办？提交评论的人不会受影响，但这个消息丢失了，垃圾信息检查也没有做。

Messenger 针对处理消息时抛出异常有一个重试机制。让我们来对它进行配置：

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

当处理消息时出现了问题，消费者程序会在放弃前重试 3 次。放弃后它不会丢掉消息，而是把它放在一个持久化存储中，即使用了 Doctrine 数据库的 ``failed`` 队列中。

查看处理失败的消息，用以下命令来重试处理：

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

在 SymfonyCloud 上运行 worker 进程
----------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

我们需要持续运行 ``messenger:consume`` 来处理来自 RabbitMQ 的消息。在 SymfonyCloud 上，这是 *worker 进程* 的角色：

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

就像本地 ``symfony`` 命令一样，SymfonyCloud 会管理重启和日志。

.. index::
    single: Symfony CLI;logs

用这个命令来查看 worker 进程中的日志：

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: 深入学习

    * `SymfonyCasts 的 Messenger 教程 <https://symfonycasts.com/screencast/messenger>`_；

    * `企业服务总线 <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ 架构和 `CQRS 模式 <https://martinfowler.com/bliki/CQRS.html>`_；

    * `Symfony 的 Messenger 文档 <https://symfony.com/doc/current/messenger.html>`_；

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
