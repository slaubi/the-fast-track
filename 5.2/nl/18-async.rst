Async gaan
==========

.. index::
    single: Async

Het controleren op spam tijdens afhandelen van het formulier kan tot problemen leiden. Als de Akismet API traag wordt, dan zal onze website ook traag worden voor gebruikers. Maar erger nog, als er een time-out optreedt of als de Akismet API niet beschikbaar is, dan kunnen we reacties kwijtraken.

Idealiter slaan we de ingediende gegevens op zonder ze te publiceren en sturen we onmiddellijk een response terug. Het controleren op spam kan dan later gebeuren.

Reacties markeren
-----------------

.. index::
    single: Command;make:entity

We moeten een ``state`` voor reacties introduceren: ``submitted``, ``spam`` en ``published``.

Voeg de ``state`` property toe aan de ``Comment`` class:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Maak een databasemigratie aan:

.. code-block:: bash

    $ symfony console make:migration

Maak de migratie zo dat bestaande reacties standaard ``published`` zijn:

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

Migreer de database:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

We moeten er ook voor zorgen dat de ``state`` standaard ``submitted`` is:

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

Update de EasyAdmin-configuratie om de status van de reactie te kunnen zien:

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

Vergeet niet om ook de tests bij te werken door de ``state`` toe te voegen aan de fixtures:

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

Simuleer de validatie voor de controller tests:

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

Vanuit een PHPUnit-test  kan je elke service uit de container halen via ``self::$container->get()``; dit geeft ook toegang tot niet-publieke services.

De Messenger begrijpen
----------------------

.. index::
    single: Messenger
    single: Components;Messenger

Het beheren van asynchrone code met Symfony is de taak van de Messenger-component:

.. code-block:: bash

    $ symfony composer req messenger

Wanneer logica asynchroon moet worden uitgevoerd, stuur dan een *message* naar een *messenger bus*. De bus voegt het bericht toe aan een *queue* en keert onmiddellijk terug om de flow van operaties zo snel mogelijk te laten hervatten.

Een *consumer* loopt continu op de achtergrond om nieuwe berichten in de queue te lezen en de bijbehorende logica uit te voeren. De consumer kan op dezelfde server draaien als de webapplicatie of op een aparte server.

Het lijkt sterk op de manier waarop HTTP-request worden behandeld, behalve dat we geen responses hebben.

Bouwen van een Message Handler
------------------------------

Een bericht is een data object class die geen logica mag bevatten en die wordt geserialiseerd om in een queue te worden opgeslagen, dus voeg er alleen "simpele" serialiseerbare gegevens aan toe.

Maak de ``CommentMessage`` class:

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

In de Messenger wereld hebben we geen controllers, maar message handlers.

Maak een ``CommentMessageHandler`` class aan onder een nieuwe ``App\MessageHandler`` namespace die weet hoe een ``CommentMessage`` verwerkt moet worden:

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

``MessageHandlerInterface`` is een *marker* interface. Het helpt Symfony alleen bij het automatisch registreren en automatisch configureren van de class als een Messenger handler. Volgens de conventie hoort logica van een handler in de ``__invoke()`` methode te staan. De ``CommentMessage`` type hint op het enige argument van deze methode, vertelt Messenger welke class er afgehandeld moet worden.

Werk de controller bij om ervoor te zorgen dat deze het nieuwe systeem gebruikt:

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

In plaats van afhankelijk te zijn van de Spam Checker, sturen we nu een message naar de bus. De handler beslist dan wat er mee te doen.

We hebben iets onverwachts bereikt. We hebben onze controller losgekoppeld van de Spam Checker en de logica verplaatst naar een nieuwe class, de handler. Dit is een perfecte use case voor de bus. Test de code, het werkt. Alles wordt nog steeds synchroon gedaan, maar de code is waarschijnlijk al "beter".

Beperken van getoonde reacties
------------------------------

Werk de weergavelogica bij om te voorkomen dat niet-gepubliceerde reacties in de frontend verschijnen:

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

Echt async werken
-----------------

Standaard worden handlers synchroon aangeroepen. Om async te gaan, moet je expliciet configureren welke queue voor welke handler gebruikt moet worden. Dit kan in het configuratiebestand ``config/packages/messenger.yaml``:

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

De configuratie vertelt de bus om instanties van ``App\Message\CommentMessage`` naar de ``async`` queue te sturen, die wordt gedefinieerd door een DSN (``MESSENGER_TRANSPORT_DSN``), die verwijst naar Doctrine zoals geconfigureerd in ``.env``. In gewoon Nederlands gebruiken we PostgreSQL als wachtrij voor onze berichten.

Stel PostgreSQL-tabellen en -triggers in:

.. code-block:: bash

    $ symfony console make:migration

En migreer de database:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. tip::

    Achter de schermen gebruikt Symfony het ingebouwde, efficiënte, schaalbare en transactionele pub/subsysteem van PostgreSQL (``LISTEN``/``NOTIFY``). Je kunt ook het RabbitMQ-hoofdstuk lezen als je het in plaats van PostgreSQL als message-broker wilt gebruiken.

Messages consumeren
-------------------

Als je probeert een nieuwe reactie toe te voegen, dan zal de spamchecker niet meer aangeroepen worden. Voeg een ``error_log()`` aanroep toe in de ``getSpamScore()`` methode om dit te bevestigen. In plaats daarvan wacht er een bericht in de queue, klaar om door bepaalde processen te worden geconsumeerd.

.. index::
    single: Command;messenger:consume

Zoals je je kan voorstellen, komt Symfony met een consumer command. Voer dat nu uit:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Het bericht van de ingevoerde reactie zou direct geconsumeerd moeten worden:

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

De activiteit van de message consumer is gelogd, maar je krijgt direct feedback in de console door de ``-vv`` flag mee te geven. Je zal zelfs de Akismet API call zien voorbijkomen.

Gebruik ``Ctrl+C`` om de consumer te stoppen.

Workers op de achtergrond laten draaien
---------------------------------------

In plaats van de consumer iedere keer te starten en te stoppen als er een reactie wordt geplaatst, willen we dat het proces continu op de achtergrond draait, zonder dat we te veel terminal windows of tabbladen open hebben.

De Symfony CLI kan dit soort background commando's of workers managen door de daemon flag (``-d``) te gebruiken bij het ``run`` commando.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Voer de message consumer opnieuw uit, maar laat hem dit keer in de achtergrond draaien:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

De ``--watch`` optie vertelt Symfony dat het commando opnieuw moet worden gestart wanneer er een wijziging op het bestandssysteem in de ``config/``, ``src/``, ``templates/``, of ``vendor/`` directories optreedt.

.. note::

    Gebruik ``-vv`` niet, anders zie je dubbele berichten in ``server:log`` (gelogde berichten en console berichten).

Als de consumer om de een of andere reden stopt met werken (geheugenlimiet, bug, ....), dan wordt deze automatisch opnieuw opgestart. En als de consumer te snel faalt, dan geeft de Symfony CLI het op.

.. index::
    single: Symfony CLI;server:log

De logs worden gestreamd via ``symfony server:log`` samen met alle andere logs die afkomstig zijn van PHP, de webserver en de applicatie:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Gebruik het ``server:status`` commando om een lijst te maken van alle workers die op de achtergrond draaien voor het huidige project:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Om een worker te stoppen, stop je de webserver of kill je de PID die door het ``server:status`` commando wordt teruggegeven:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Mislukte berichten opnieuw proberen
-----------------------------------

En wat als Akismet down is terwijl er een bericht wordt geconsumeerd? Er is geen impact voor mensen die een reactie geven, maar het bericht gaat verloren en spam wordt niet gecontroleerd.

Messenger heeft een retry mechanisme voor het geval er een exception optreedt tijdens de afhandeling van een bericht. Laten we dit configureren:

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

Als er zich een probleem voordoet bij het afhandelen van een bericht, dan zal de consumer dit 3 maal opnieuw proberen voordat deze het opgeeft. Maar in plaats van het bericht te verwijderen, zal het bericht permanent worden opgeslagen in de ``failed`` queue, die gebruik maakt van een andere database tabel.

Inspecteer de mislukte berichten en probeer ze opnieuw aan te bieden via de volgende commando's:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Workers draaien op SymfonyCloud
-------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Om berichten van PostgreSQL te consumeren, moeten we het ``messenger:consume`` commando continu uitvoeren. Op SymfonyCloud is dit de rol van een *worker*:

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

Net als bij de Symfony CLI, beheert SymfonyCloud de restarts en logs.

.. index::
    single: Symfony CLI;logs

Om logs voor een worker te krijgen, gebruik:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Verder gaan

    * `SymfonyCasts Messenger tutorial <https://symfonycasts.com/screencast/messenger>`_;

    * De `Enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ architectuur en het `CQRS patroon <https://martinfowler.com/bliki/CQRS.html>`_;

    * De `Symfony Messenger documentatie <https://symfony.com/doc/current/messenger.html>`_;
