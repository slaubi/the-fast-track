Procesând asincron
===================

.. index::
    single: Async

Verificarea spamului în timpul procesării expedierii formularului poate genera unele probleme. Dacă API-ul Akismet devine lent, site-ul nostru va fi de asemenea lent pentru utilizatori. Dar și mai rău, dacă atingem un interval de timp sau dacă API-ul Akismet nu este disponibil, am putea pierde comentarii.

În mod ideal, ar trebui să stocăm datele expediate fără să le publicăm și să returnăm un răspuns imediat. Verificarea spamului poate fi făcut independent.

Marcarea comentariilor
----------------------

.. index::
    single: Command;make:entity

Trebuie să introducem un ``state`` pentru comentarii: ``submitted``, ``spam`` și ``published``.

Adaugă proprietatea ``state`` la clasa ``Comment``:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Creează o migrare a bazei de date:

.. code-block:: bash

    $ symfony console make:migration

Modifică migrarea pentru a actualiza toate comentariile existente cu statutul ``published`` în mod implicit:

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

Migrează baza de date:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

De asemenea, ar trebui să ne asigurăm că, în mod implicit, ``state`` este setată pe ``submitted``:

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

Actualizează configurația EasyAdmin pentru a putea vedea starea comentariului:

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

Nu uita să actualizezi și testele prin setarea parametrului ``state`` a datelor de test:

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

Pentru testele controlerului, simulează validarea:

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

De la un test PHPUnit, poți obține orice serviciu din container prin ``self::$container->get()``; aceasta oferă, de asemenea, acces la servicii non-publice.

Componenta Messenger
--------------------

.. index::
    single: Messenger
    single: Components;Messenger

Gestionarea codului asincron cu Symfony este sarcina componentei Messenger:

.. code-block:: bash

    $ symfony composer req messenger

Când o anumită logică ar trebui să fie executată în mod asincron, expediază un *mesaj* la un *bus messenger*. Bus-ul stochează mesajul într-o *coadă de mesaje* (queue) și încetează execuția imediat pentru a permite reluarea fluxului de operații cât mai repede posibil.

Un *consumator* rulează continuu în fundal pentru a citi mesaje noi din coada de mesaje și pentru a executa logica asociată. Consumatorul poate rula pe același server ca și aplicația web sau pe unul separat.

Este foarte similar cu modul în care sunt gestionate cererile HTTP, cu excepția faptului că nu avem răspunsuri.

Dezvoltarea unui Handler Messenger
----------------------------------

Un mesaj este obiectul cu date al unei clase care nu ar trebui să dețină nicio logică. Acesta va fi serializat pentru a fi stocat într-o coadă de mesaje, astfel încât să stocăm doar date simple serializabile.

Creează clasa ``CommentMessage``:

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

În lumea Messenger, nu avem controlere, ci handlere de mesaje.

Creează o clasă ``CommentMessageHandler`` sub un nou spațiu de nume ``App\MessageHandler`` care știe să gestioneze mesajele ``CommentMessage``:

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

``MessageHandlerInterface`` este o interfață *marker*. Aceasta ajută Symfony la înregistrarea și la configurarea automată a clasei ca un handler (manipulator) de mesagerie. Prin convenție, logica unui handler este implementată într-o metodă numită ``__invoke()``. Indicația de tip ``CommentMessage`` pe argumentul acestei metode indică Messenger-ului clasa care va manipula acest mesaj.

Actualizează controlerul pentru a utiliza noul sistem:

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

În loc să depindem de Spam Checker, expediem un mesaj în bus. Handler-ul decide apoi ce să facă cu acesta.

Am obținut ceva neașteptat. Am decuplat controlerul nostru de la Spam Checker și am mutat logica într-o clasă nouă: handler. Este un caz perfect pentru utilizarea bus-ului. Testează codul, pur și simplu funcționează. Totul este încă făcut în mod sincron, dar probabil codul este deja "mai bun".

Restricționarea comentariilor afișate
---------------------------------------

Actualizează logica afișajului pentru a evita apariția comentariilor nepublicate pe frontend:

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

Devenind cu adevărat asincron
------------------------------

În mod implicit, manipulatorii sunt apelați în mod sincron. Pentru a-i face să execute în mod asincron, trebuie să configurezi explicit ce coadă de mesaje să fie folosită pentru fiecare handler din fișierul de configurare ``config/packages/messenger.yaml``:

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

Configurația spune bus-ului să expedieze instanțe de ``App\Message\CommentMessage`` la coada de mesaje ``async``, care este definită de DSN (``MESSENGER_TRANSPORT_DSN``), care arată către Doctrine așa cum este configurată în ``.env``. Pe limba română, folosim PostgreSQL ca o coadă pentru mesajele noastre.

Configurează tabelele și declanșatoarele PostgreSQL:

.. code-block:: bash

    $ symfony console make:migration

Și migrează baza de date:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. tip::

    În culise, Symfony folosește sistemul PostgreSQL încorporat, performant, scalabil și tranzacțional pub/sub („NOTIFY” / „LISTEN”). De asemenea, puteți citi capitolul RabbitMQ dacă doriți să-l utilizați în locul PostgreSQL ca broker de mesaje.

Consumând mesajele
-------------------

Dacă încerci să expediezi un nou comentariu, verificatorul de spam nu va mai fi apelat. Adaugă un apel ``error_log()`` în metoda ``getSpamScore()`` pentru a confirma asta. În schimb, un mesaj este în așteptare la coadă, gata să fie consumat de unele procese.

.. index::
    single: Command;messenger:consume

După cum îți poți imagina, Symfony vine cu o comandă de consumator. Ruleaz-o acum:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Acesta ar trebui să consume imediat mesajul trimis pentru comentariul expediat:

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

Activitatea consumatorului de mesaje este înregistrată, dar primești feedback instantaneu pe consolă utilizând opțiunea ``-vv``. Ar trebui să fii în măsură să detectezi apelul către API-ul Akismet.

Pentru a opri consumatorul, apasă ``Ctrl + C``.

Executarea comenzilor în fundal
--------------------------------

În loc să lansăm consumatorul de fiecare dată când postăm un comentariu și îl oprim imediat după aceea, preferăm să-l rulăm continuu, fără a avea prea multe ferestre sau file deschise.

Symfony CLI poate gestiona astfel de comenzi de fundal sau workers folosind opțiunea (``-d``) cu comanda ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Pornește din nou consumatorul de mesaje, dar trimite-l în fundal:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

Opțiunea ``--watch`` indică Symfony că comanda trebuie repornită ori de câte ori există o schimbare a sistemului de fișiere în ``config/``, ``src/``, ``templates /`` sau ``vendor/`` directoare.

.. note::

    Nu folosi ``-vv`` deoarece vei duplica mesajele în ``server:log`` (mesaje logate și mesaje de consolă).

Dacă consumatorul nu mai funcționează din anumite motive (limită de memorie, eroare, ...), acesta va fi repornit automat. Și dacă consumatorul eșuează prea repede, Symfony CLI se va opri.

.. index::
    single: Symfony CLI;server:log

Jurnalele sunt transmise prin ``server symfony:log`` cu toate celelalte provenind de la PHP, serverul web și aplicație:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Folosește comanda ``server: status`` pentru a enumera toate comenzile de fundal gestionate pentru proiectul curent:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Pentru a opri un worker, oprește serverul web sau distruge PID-ul dat de comanda ``server:status``:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Reîncearcă expedierea mesajelor eșuate
-----------------------------------------

Ce se întâmplă dacă Akismet nu răspunde în timp ce consumă un mesaj? Nu există niciun impact pentru persoanele care trimit comentarii, dar mesajul este pierdut și spamul nu este verificat.

Messenger are un mecanism de reîncercare pentru situația când apare o excepție în timpul manipulării unui mesaj. Să-l configurăm:

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

Dacă apare o problemă în timpul manipulării unui mesaj, consumatorul va încerca de trei ori înainte de a renunța. Dar, în loc să elimine mesajul, îl va stoca permanent în coada de mesaje ``failed``, care folosește alt tabel din baza de date.

Verifică mesajele eșuate și încearcă-le din nou prin următoarele comenzi:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Executarea lucrătorilor pe SymfonyCloud
----------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Pentru a consuma mesajele din PostgreSQL, trebuie să executăm comanda ``messenger:consume`` continuu. Pe SymfonyCloud, acesta este rolul unui *worker*:

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

Ca și pentru Symfony CLI, SymfonyCloud gestionează repornirile și jurnalele.

.. index::
    single: Symfony CLI;logs

Pentru a obține jurnalul unui worker, utilizează:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Mergând mai departe

    * `Tutorial SymfonyCasts Messenger <https://symfonycasts.com/screencast/messenger>`_;

    * Arhitectura `Enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ și `modelul CQRS <https://martinfowler.com/bliki/CQRS.html>`_;

    * Documentele `Symfony Messenger <https://symfony.com/doc/current/messenger.html>`_;
