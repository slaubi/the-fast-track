Asynchrone Verarbeitung
=======================

.. index::
    single: Async

Die Überprüfung auf Spam während der Bearbeitung des Formularübermittlung kann zu Problemen führen. Wenn die Akismet-API langsam ist, wird unsere Website auch für Benutzer*innen langsam. Aber noch schlimmer: Wir könnten Kommentare verlieren, falls wir in einen Timeout laufen oder die Akismet-API nicht verfügbar ist.

Im Idealfall sollten wir die übermittelten Daten speichern, ohne sie zu veröffentlichen, und sofort eine Response zurückliefern. Die Überprüfung auf Spam kann dann unabhängig davon durchgeführt werden.

Kommentare kennzeichnen
-----------------------

.. index::
    single: Command;make:entity

Wir müssen ein ``state``-Feld für Kommentare einführen: ``submitted``, ``spam`` und ``published``.

Füge die ``state``-Property zur ``Comment``-Klasse hinzu:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Erstelle eine Datenbankmigration:

.. code-block:: bash

    $ symfony console make:migration

Passe die Migration an, um alle vorhandenen Kommentare standardmäßig auf ``published`` zu setzen:

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

Führe die Datenbankmigration durch:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

Wir sollten auch sicherstellen, dass der ``state``-Wert standardmäßig auf ``submitted`` gesetzt ist:

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

Aktualisiere die EasyAdmin-Konfiguration, um den Zustand des Kommentars zu sehen:

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

Denk daran, auch die Tests zu aktualisieren, indem Du ``state`` zu den Fixtures hinzufügst:

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

Simuliere die Validierung für die Controller-Tests:

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

Du kannst von einem PHPUnit-Test aus jeden beliebigen Service aus dem Container über ``self::$container->get()`` holen; dies ermöglicht auch den Zugriff auf Services, die nicht public sind.

Messenger verstehen
-------------------

.. index::
    single: Messenger
    single: Components;Messenger

Die asynchrone Verarbeitung mit Symfony ist Aufgabe der Messenger-Komponente:

.. code-block:: bash

    $ symfony composer req messenger

Wenn Logik asynchron ausgeführt werden soll, sende eine *Message* (Nachricht) an einen *Messenger-Bus*. Der Bus speichert die Message in einer *Queue* (Warteschlange) und kehrt sofort zurück, um den Betriebsablauf so schnell wie möglich wieder aufzunehmen.

Ein *Consumer* läuft kontinuierlich im Hintergrund, um neue Messages auf der Queue zu lesen und die zugehörige Logik auszuführen. Der Consumer kann auf dem gleichen Server wie die Webanwendung oder auf einem separaten Server laufen.

Das Ganze ist der Art und Weise, wie HTTP-Requests behandelt werden sehr ähnlich, nur dass wir keine Responses zurückliefern.

Einen Message Handler erstellen
-------------------------------

Eine Message ist eine Datenobjektklasse, die keine Logik enthalten sollte. Sie wird serialisiert, um in einer Queue gespeichert zu werden, also speichere darin nur "einfache" serialisierbare Daten.

Lege die ``CommentMessage``-Klasse an:

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

In der Messenger-Welt haben wir keine Controller, sondern Message-Handler.

Erstelle eine ``CommentMessageHandler``-Klasse unter einem neuen ``App\MessageHandler``-Namespace, die weiß, wie man mit ``CommentMessage``-Messages umgeht:

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

Das ``MessageHandlerInterface`` dient lediglich zur Markierung einer Klasse. Es hilft Symfony nur, die Klasse automatisch zu registrieren und automatisch als Messenger-Handler zu konfigurieren. Nach Konvention lebt die Logik eines Handlers in einer Methode namens ``__invoke()``. Der ``CommentMessage``-Type-Hint auf das eine Argument dieser Methode sagt dem Messenger, welche Klasse diese verarbeiten soll.

Aktualisiere den Controller, damit er das neue System verwendet:

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

Anstatt vom Spam Checker abhängig zu sein, senden wir nun eine Message zum Bus. Der Handler entscheidet dann, was er damit macht.

Wir haben etwas Unerwartetes erreicht. Wir haben unseren Controller vom Spam Checker entkoppelt und die Logik in eine neue Klasse, den Handler, verschoben. Es ist ein perfekter Anwendungsfall für den Bus. Teste den Code, er funktioniert. Alles wird noch synchron gemacht, aber der Code ist wahrscheinlich schon "besser".

Die angezeigten Kommentare einschränken
----------------------------------------

Aktualisiere die Anzeigelogik, um zu vermeiden, dass unveröffentlichte Kommentare im Frontend erscheinen:

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

Echt Asynchron
--------------

.. index::
    single: RabbitMQ

Standardmäßig werden Handler synchron aufgerufen. Um asynchron zu werden, musst Du in der ``config/packages/messenger.yaml``-Konfigurationsdatei für jeden Handler explizit konfigurieren, welche Queue verwendet werden soll:

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

Die Konfiguration weist den Bus an, Instanzen von ``App\Message\CommentMessage`` an die ``async``-Queue zu senden, die durch einen DSN definiert und in der Environment-Variable ``RABBITMQ_DSN`` gespeichert ist.

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

.. sidebar:: Datenbank-Daten exportieren und importieren

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

Messages verarbeiten
--------------------

Wenn Du versuchst, einen neuen Kommentar abzugeben, wird der Spam-Checker nicht mehr aufgerufen. Füge einen ``error_log()``-Aufruf in der ``getSpamScore()``-Methode hinzu, um Dich zu vergewissern. Stattdessen wartet in RabbitMQ eine Message, die von bestimmten Prozessen verarbeitet werden kann.

.. index::
    single: Command;messenger:consume

Selbstverständlich wird Symfony mit einem Verarbeitungsbefehl (Consumer Command) geliefert. Führe diesen jetzt aus:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Er sollte die für den eingereichten Kommentar versendete Message sofort verarbeiten:

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

Die Aktivität des Message Consumers wird geloggt, aber Du erhältst sofortiges Feedback auf der Konsole, indem Du das ``-vv`` Flag übergibst. Du solltest sogar den Aufruf der Akismet-API sehen können.

Drücke ``Ctrl+C``, um den Consumer zu stoppen.

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

Worker im Hintergrund ausführen
--------------------------------

Anstatt den Consumer jedes Mal zu starten, wenn wir einen Kommentar posten und ihn sofort danach stoppen, wollen wir ihn kontinuierlich ausführen, ohne zu viele Terminalfenster oder -tabs geöffnet zu haben.

Die Symfony CLI kann solche Hintergrundbefehle oder Worker verwalten, indem Du das Daemon-Flag (``-d``) zusätzlich zum ``run``-Befehl verwendest.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Führe den Message Consumer erneut aus, aber schiebe ihn in den Hintergrund:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

Die ``--watch``-Option teilt Symfony mit, dass der Befehl neu gestartet werden muss, wenn Dateien in den Verzeichnissen ``config/``, ``src/``, ``templates/`` oder ``vendor/`` verändert werden.

.. note::

    Verwende nicht ``-vv``, da Du sonst in ``server:log`` doppelte Meldungen erhalten würdest (Log- und Konsolenmeldungen).

Wenn der Consumer aus irgendeinem Grund nicht mehr funktioniert (Speicherlimit, Fehler, ...), wird er automatisch neugestartet. Und wenn der Consumer zu schnell versagt, gibt die Symfony CLI auf.

.. index::
    single: Symfony CLI;server:log

Logs werden von ``symfony server:log`` mit allen anderen Logs, die von PHP, dem Webserver und der Anwendung stammen, gesammelt:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Verwende den ``server:status``-Befehl, um alle für das aktuelle Projekt verwalteten Worker aus dem Hintergrund aufzulisten:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Um einen Worker zu stoppen, stoppe den Webserver oder beende die PID, die durch den ``server:status``-Befehl gegeben wird:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Fehlgeschlagene Messages erneut verarbeiten
-------------------------------------------

Was passiert, wenn Akismet während des Verarbeitens einer Message ausgefallen ist? Es gibt keine Auswirkungen für Personen, die Kommentare abgeben, aber die Nachricht geht verloren und Spam wird nicht überprüft.

Der Messenger hat einen Wiederholungsmechanismus, wenn beim Verarbeiten einer Message ein Fehler auftritt. Konfigurieren wir ihn:

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

Wenn ein Problem beim Verarbeiten mit einer Message auftritt, wird der Consumer es dreimal erneut probieren, bevor er aufgibt. Aber anstatt die Message zu verwerfen, wird sie in einem dauerhafteren Speicher, der ``failed``-Queue, gespeichert, die die Doctrine Datenbank verwendet.

Überprüfe fehlgeschlagene Messages und verarbeite sie mit den folgenden Befehlen erneut:

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

Worker in der SymfonyCloud ausführen
-------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Um Messages von RabbitMQ zu bearbeiten, müssen wir den ``messenger:consume``-Befehl kontinuierlich ausführen. In der SymfonyCloud ist dies die Rolle eines *Workers*:

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

Wie Symfony CLI verwaltet die SymfonyCloud Neustarts und Logs.

.. index::
    single: Symfony CLI;logs

Um Logs für einen Worker zu erhalten, verwende:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Weiterführendes

    * `SymfonyCasts Messenger Tutorial <https://symfonycasts.com/screencast/messenger>`_;

    * Die `Enterprise Service Bus-Architektur <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ und das `CQRS-Pattern <https://martinfowler.com/bliki/CQRS.html>`_;

    * Die `Symfony Messenger Dokumentation <https://symfony.com/doc/current/messenger.html>`_;

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
