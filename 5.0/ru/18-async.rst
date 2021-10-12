Переход к асинхронности
============================================

.. index::
    single: Async

Во время проверки формы на наличие спама могут возникнуть проблемы. В случае, если ответ от Akismet API будет долгим, наша страница с формой также станет работать медленнее. Худший сценарий — мы можем совсем потерять комментарии из-за достижения тайм-аута или недоступности Akismet API.

В идеале нам нужно немедленно возвращать ответ на отправленную форму, а приложение должно сохранять данные без попытки их сразу же опубликовать. Проверить на спам можно и позже.

Установка статусов комментариев
------------------------------------------------------------

.. index::
    single: Command;make:entity

Реализуем статусы (``state``) для комментариев: ``submitted`` (отправлен), ``spam`` (спам) и ``published`` (опубликован).

Добавим свойство ``state`` в класс ``Comment``:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Создайте миграцию базы данных:

.. code-block:: bash

    $ symfony console make:migration

Измените миграцию, чтобы все существующие комментарии были по умолчанию со статусом ``published``:

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

Выполните миграцию базы данных:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

Убедимся, что для свойства ``state`` по умолчанию задан статус ``submitted``.

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

Обновите конфигурацию EasyAdmin, чтобы отображать статус комментария в административной панели:

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

Не забудьте также обновить тесты, установив новое свойство ``state`` в фикстурах:

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

Для тестирования контроллера смоделируйте проверку комментария:

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

В тесте PHPUnit вы можете получить любой сервис из контейнера с помощью метода ``self::$container->get()``, который также даёт доступ к непубличным сервисам.

Внедрение компонента Messenger
-------------------------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Компонент Messenger управляет асинхронным кодом в Symfony:

.. code-block:: bash

    $ symfony composer req messenger

Для выполнения асинхронных задач, отправьте *сообщение* на *шину обмена сообщениями*. Шина сохраняет сообщение в *очередь* и немедленно возвращает ответ, чтобы исключить любые задержки в ваших приложениях.

*Получатель* работает постоянно в фоновом режиме, читая новые сообщения из очереди и выполняя соответствующие задачи. Получатель может работать как на том же сервере, где находится само приложение, так и на отдельном сервере.

Это напоминает обработку HTTP-запросов, за исключением того, что здесь нет ответов.

Создание обработчика сообщений
----------------------------------------------------------

Сообщение — это класс с данными, в котором не должно быть никакой логики. Он будет сериализован для хранения в очереди, поэтому держите в нем только "простые" сериализуемые данные.

Создайте класс ``CommentMessage``:

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

В мире Messenger нет контроллеров, есть только обработчики сообщений.

Создайте класс ``CommentMessageHandler`` в новом пространстве имён ``App\MessageHandler``, который  знает, как обрабатывать сообщения ``CommentMessage``:

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

``MessageHandlerInterface`` — это *маркирующий* интерфейс. Его наличие помогает Symfony понять, что класс нужно автоматически зарегистрировать и настроить в качестве обработчика Messenger. По соглашению логика обработчика находится в методе ``__invoke()``. Подсказка об используемом типе ``CommentMessage`` для единственного аргумента этого метода сообщает Messenger, какой класс он будет обрабатывать.

Обновите контроллер, чтобы использовать новую систему:

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

Теперь вместо того, чтобы зависеть от скорости работы антиспам-сервиса, мы отправляем сообщение на шину. Затем обработчик решает, что делать с этим сообщением.

Произошло что-то неожиданное: мы отделили наш контроллер от антиспам-сервиса и перенесли логику в новый класс — обработчик. Это идеальный вариант для использования шины. Проверьте код — всё работает. Операции всё ещё выполняются синхронно, но код, вероятно, уже стал "лучше".

Сокращение количества отображаемых комментариев
-------------------------------------------------------------------------------------------

Обновите логику отображения, чтобы на странице показывались только опубликованные комментарии:

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

Переходим к асинхронности по-настоящему
--------------------------------------------------------------------------

.. index::
    single: RabbitMQ

По умолчанию обработчики вызываются синхронно. Для  асинхронного вызова необходимо указать, какую очередь использовать для каждого обработчика в конфигурационном файле ``config/packages/messenger.yaml``:

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

Шина настроена на отсылку сообщений типа ``App\Message\CommentMessage`` в очередь ``async``, которая определена DSN, хранящейся в переменной окружения ``RABBITMQ_DSN``.

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

.. sidebar:: Создание резервной копии и восстановление базы данных

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

Получение сообщений
-------------------------------------

При отправке нового комментария, проверка на спам больше не сработает. Чтобы убедиться в этом, добавьте вызов функции``error_log()`` в метод ``getSpamScore()``. Видно, что  сообщение находится в очереди RabbitMQ и ждёт обработки.

.. index::
    single: Command;messenger:consume

Как вы понимаете, у Symfony есть команда для получения сообщений. Давайте выполним её сейчас:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Команда немедленно получит сообщение, отправленное для комментария:

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

Все действия получателя сообщений пишутся в логи, но эти же действия можно увидеть в консоли в реальном времени, если к команде добавить флаг ``-vv``. Вы даже сможете увидеть вызов к Akismet API.

Для остановки получателя, нажмите ``Ctrl+C``.

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

Запуск воркеров в фоновом режиме
------------------------------------------------------------

Вместо того, чтобы запускать получателя каждый раз после отправки комментария и тут же его останавливать, хотелось бы, чтобы он работал непрерывно, и при этом не нужно было бы открывать слишком много окон или вкладок терминала.

Symfony CLI может управлять такими фоновыми командами или воркерами, если добавить флаг ( ``-d`` ) к команде ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Запустите получателя сообщений ещё раз, но теперь уже в фоновом режиме:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

При использовании параметра ``--watch`` Symfony перезапускает команду каждый раз при изменении файловой системы в директориях ``config/``, ``src/``, ``templates/`` или ``vendor/``.

.. note::

    Не используйте флаг ``-vv``, иначе получите дублирующие сообщения в логах команды ``server:log`` (логированные и консольные сообщения).

Если получатель перестаёт работать по какой-либо причине (недостаточно памяти, баг и т.д.), он будет перезапущен автоматически. Но Symfony CLI не перезапустит получателя в том случае, если получатель выдает ошибку сразу после запуска.

.. index::
    single: Symfony CLI;server:log

Логи можно просматривать через команду ``symfony server:log`` вместе с остальными логами, приходящими от PHP, веб-сервера и приложения:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Команда ``server:status`` выведет все фоновые воркеры для текущего проекта:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Чтобы остановить воркер, нужно либо остановить веб-сервер, либо принудительно завершить процесс по его PID после запуска команды ``server:status``:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Повторная обработка сообщений в случае ошибки
-------------------------------------------------------------------------------------

Что, если Akismet будет недоступен во время получения сообщения? Это не отразится на пользователях, оправляющих комментарии, но грозит потерей сообщения и, как следствие, отсутствием проверки на спам.

У компонента Messenger есть механизм повторных попыток в случае возникновения исключений во время обработки сообщений. Давайте его настроим:

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

Если при обработке сообщения возникнет ошибка, получатель повторит попытку ещё 3 раза, а затем остановится. Но вместо удаления получатель перенесёт сообщение в постоянное хранилище — очередь ``failed``, которая использует базу данных Doctrine.

Просматривать сообщения, давшие сбой и повторить попытку их обработки можно с помощью следующих команд:

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

Выполнение воркеров на SymfonyCloud
-------------------------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Чтобы получать сообщения от RabbitMQ, нам нужно постоянно запускать команду ``messenger:consume``. В SymfonyCloud для этой задачи отведена специальная роль — *воркер*:

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

Подобно Symfony CLI, SymfonyCloud позволяет управлять перезагрузками и логами.

.. index::
    single: Symfony CLI;logs

Чтобы посмотреть логи воркера, воспользуйтесь командой ниже:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Двигаемся дальше

    * `Видеокурс по Messenger на SymfonyCasts <https://symfonycasts.com/screencast/messenger>`_;

    * `Архитектура сервисной шины предприятия <https://ru.wikipedia.org/wiki/Сервисная_шина_предприятия>`_ и `шаблон CQRS <https://martinfowler.com/bliki/CQRS.html>`_;

    * `Документация по Symfony Messenger <https://symfony.com/doc/current/messenger.html>`_;

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
