Going Async
===========

.. index::
    single: Async

Checking for spam during the handling of the form submission might lead to some problems. If the Akismet API becomes slow, our website will also be slow for users. But even worse, if we hit a timeout or if the Akismet API is unavailable, we might lose comments.

Ideally, we should store the submitted data without publishing it, and immediately return a response. Checking for spam can then be done out of band.

Flagging Comments
-----------------

.. index::
    single: Command;make:entity

We need to introduce a ``state`` for comments: ``submitted``, ``spam``, and ``published``.

Add the ``state`` property to the ``Comment`` class:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Create a database migration:

.. code-block:: bash

    $ symfony console make:migration

Modify the migration to update all existing comments to be ``published`` by default:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255)');
    +        $this->addSql("UPDATE comment SET state='published'");
    +        $this->addSql('ALTER TABLE comment ALTER COLUMN state SET NOT NULL');
         }

         public function down(Schema $schema): void

.. index::
    single: Command;doctrine:migrations:migrate

Migrate the database:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

We should also make sure that, by default, the ``state`` is set to ``submitted``:

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

Update the EasyAdmin configuration to be able to see the comment's state:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/CommentCrudController.php
    +++ b/src/Controller/Admin/CommentCrudController.php
    @@ -51,6 +51,7 @@ class CommentCrudController extends AbstractCrudController
                 ->setLabel('Photo')
                 ->onlyOnIndex()
             ;
    +        yield TextField::new('state');

             $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
                 'html5' => true,

Don't forget to also update the tests by setting the ``state`` of the fixtures:

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

For the controller tests, simulate the validation:

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

From a PHPUnit test, you can get any service from the container via ``self::$container->get()``; it also gives access to non-public services.

Understanding Messenger
-----------------------

.. index::
    single: Messenger
    single: Components;Messenger

Managing asynchronous code with Symfony is the job of the Messenger Component:

.. code-block:: bash

    $ symfony composer req messenger

When some logic should be executed asynchronously, send a *message* to a *messenger bus*. The bus stores the message in a *queue* and returns immediately to let the flow of operations resume as fast as possible.

A *consumer* runs continuously in the background to read new messages on the queue and execute the associated logic. The consumer can run on the same server as the web application or on a separate one.

It is very similar to the way HTTP requests are handled, except that we don't have responses.

Coding a Message Handler
------------------------

A message is a data object class that should not hold any logic. It will be serialized to be stored in a queue, so only store "simple" serializable data.

Create the ``CommentMessage`` class:

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

In the Messenger world, we don't have controllers, but message handlers.

Create a ``CommentMessageHandler`` class under a new ``App\MessageHandler`` namespace that knows how to handle ``CommentMessage`` messages:

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

``MessageHandlerInterface`` is a *marker* interface. It only helps Symfony auto-register and auto-configure the class as a Messenger handler. By convention, the logic of a handler lives in a method called ``__invoke()``. The ``CommentMessage`` type hint on this method's one argument tells Messenger which class this will handle.

Update the controller to use the new system:

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

         #[Route('/', name: 'homepage')]
    @@ -36,7 +39,7 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -54,6 +57,7 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +            $this->entityManager->flush();

                 $context = [
                     'user_ip' => $request->getClientIp(),
    @@ -61,11 +65,8 @@ class ConferenceController extends AbstractController
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

Instead of depending on the Spam Checker, we now dispatch a message on the bus. The handler then decides what to do with it.

We have achieved something unexpected. We have decoupled our controller from the Spam Checker and moved the logic to a new class, the handler. It is a perfect use case for the bus. Test the code, it works. Everything is still done synchronously, but the code is probably already "better".

Restricting Displayed Comments
------------------------------

Update the display logic to avoid non-published comments from appearing on the frontend:

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

Going Async for Real
--------------------

By default, handlers are called synchronously. To go async, you need to explicitly configure which queue to use for each handler in the ``config/packages/messenger.yaml`` configuration file:

.. code-block:: diff
    :caption: patch_file

    --- a/.env
    +++ b/.env
    @@ -29,7 +29,7 @@ DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=13&charset=utf8"

     ###> symfony/messenger ###
     # Choose one of the transports below
    -# MESSENGER_TRANSPORT_DSN=doctrine://default
    +MESSENGER_TRANSPORT_DSN=doctrine://default
     # MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages
     # MESSENGER_TRANSPORT_DSN=redis://localhost:6379/messages
     ###< symfony/messenger ###
    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,15 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            # async: '%env(MESSENGER_TRANSPORT_DSN)%'
    +            async:
    +                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                options:
    +                    auto_setup: false
    +                    use_notify: true
    +                    check_delayed_interval: 60000
                 # failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:
                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

The configuration tells the bus to send instances of ``App\Message\CommentMessage`` to the ``async`` queue, which is defined by a DSN (``MESSENGER_TRANSPORT_DSN``), which points to Doctrine as configured in ``.env``. In plain English, we are using PostgreSQL as a queue for our messages.

Setup PostgreSQL tables and triggers:

.. code-block:: bash

    $ symfony console make:migration

And migrate the database:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. tip::

    Behind the scenes, Symfony uses the PostgreSQL builtin, performant, scalable, and transactional pub/sub system (``LISTEN``/``NOTIFY``). You can also read the RabbitMQ chapter if you want to use it instead of PostgreSQL as a message broker.

Consuming Messages
------------------

If you try to submit a new comment, the spam checker won't be called anymore. Add an ``error_log()`` call in the ``getSpamScore()`` method to confirm. Instead, a message is waiting in the queue, ready to be consumed by some processes.

.. index::
    single: Command;messenger:consume

As you might imagine, Symfony comes with a consumer command. Run it now:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

It should immediately consume the message dispatched for the submitted comment:

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

The message consumer activity is logged, but you get instant feedback on the console by passing the ``-vv`` flag. You should even be able to spot the call to the Akismet API.

To stop the consumer, press ``Ctrl+C``.

Running Workers in the Background
---------------------------------

Instead of launching the consumer every time we post a comment and stopping it immediately after, we want to run it continuously without having too many terminal windows or tabs open.

The Symfony CLI can manage such background commands or workers by using the daemon flag (``-d``) on the ``run`` command.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Run the message consumer again, but send it in the background:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

The ``--watch`` option tells Symfony that the command must be restarted whenever there is a filesystem change in the ``config/``, ``src/``, ``templates/``, or ``vendor/`` directories.

.. note::

    Do not use ``-vv`` as you would have duplicated messages in ``server:log`` (logged messages and console messages).

If the consumer stops working for some reason (memory limit, bug, ...), it will be restarted automatically. And if the consumer fails too fast, the Symfony CLI will give up.

.. index::
    single: Symfony CLI;server:log

Logs are streamed via ``symfony server:log`` with all the other logs coming from PHP, the web server, and the application:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Use the ``server:status`` command to list all background workers managed for the current project:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

To stop a worker, stop the web server or kill the PID given by the ``server:status`` command:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Retrying Failed Messages
------------------------

What if Akismet is down while consuming a message? There is no impact for people submitting comments, but the message is lost and spam is not checked.

Messenger has a retry mechanism for when an exception occurs while handling a message. Let's configure it:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -1,7 +1,7 @@
     framework:
         messenger:
             # Uncomment this (and the failed transport below) to send failed messages to this transport for later handling.
    -        # failure_transport: failed
    +        failure_transport: failed

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    @@ -11,7 +11,10 @@ framework:
                         auto_setup: false
                         use_notify: true
                         check_delayed_interval: 60000
    -            # failed: 'doctrine://default?queue_name=failed'
    +                retry_strategy:
    +                    max_retries: 3
    +                    multiplier: 2
    +            failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

If a problem occurs while handling a message, the consumer will retry 3 times before giving up. But instead of discarding the message, it will store it permanently in the ``failed`` queue, which uses another database table.

Inspect failed messages and retry them via the following commands:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Running Workers on SymfonyCloud
-------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

To consume messages from PostgreSQL, we need to run the ``messenger:consume`` command continuously. On SymfonyCloud, this is the role of a *worker*:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -50,3 +50,8 @@ hooks:
             set -x -e

             (>&2 symfony-deploy)
    +
    +workers:
    +    messages:
    +        commands:
    +            start: symfony console messenger:consume async -vv --time-limit=3600 --memory-limit=128M

Like for the Symfony CLI, SymfonyCloud manages restarts and logs.

.. index::
    single: Symfony CLI;logs

To get logs for a worker, use:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Going Further

    * `SymfonyCasts Messenger tutorial <https://symfonycasts.com/screencast/messenger>`_;

    * The `Enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ architecture and the `CQRS pattern <https://martinfowler.com/bliki/CQRS.html>`_;

    * The `Symfony Messenger docs <https://symfony.com/doc/current/messenger.html>`_;
