Idziemy w asynchroniczność
============================

.. index::
    single: Async

Sprawdzenie, czy komentarz nie jest spamem, podczas obsługi wysyłania formularza może prowadzić do pewnych problemów. Jeśli interfejs API Akismet stanie się powolny, nasza strona również będzie powolna dla użytkowników. Co gorsza, jeżeli przekroczymy limit czasu lub API Akismet jest niedostępne, możemy stracić komentarze.

Idealnie byłoby, gdybyśmy przechowywali przesłane dane bez ich publikowania i natychmiast zwracali odpowiedź. Sprawdzenie, czy nie ma spamu, może zostać wykonane poza głównym wątkiem.

Oznaczanie komentarzy
---------------------

.. index::
    single: Command;make:entity

Musimy wprowadzić atrybut ``state`` określający stan komentarzy, przyjmujący trzy wartości: ``submitted`` dla komentarzy wysłanych, ``published`` dla opublikowanych oraz ``spam`` dla odrzuconych jako spamowe.

Dodaj atrybut ``state`` do klasy ``Comment``:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Utwórz migrację bazy danych:

.. code-block:: bash

    $ symfony console make:migration

Zmodyfikuj migrację tak, by zaktualizowała status wszystkich istniejących komentarzy na ``published``:

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

Uruchom migrację bazy danych:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

Powinniśmy także upewnić się, że domyślna wartość atrybutu ``state`` jest ustawiona na ``submitted``:

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

Zaktualizuj konfigurację EasyAdmin tak, by móc zobaczyć stan komentarza:

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

Nie zapomnij również zaktualizować testów poprzez ustawienie atrybutu ``state`` w danych testowych (ang. fixtures):

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

W przypadku testów kontrolera należy przeprowadzić symulację walidacji:

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

Z poziomu testu PHPUnit można uzyskać dowolną usługę z kontenera za pośrednictwem ``self::$container->get()``; daje to również dostęp do usług niepublicznych.

Zrozumienie komponentu Messenger
--------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Zarządzanie kodem asynchronicznym w Symfony jest zadaniem komponentu Messenger:

.. code-block:: bash

    $ symfony composer req messenger

Gdy jakieś działania powinny być wykonywane asynchronicznie, wyślij *wiadomość* (ang. message) do *magistrali komunikacyjnej* (ang. messenger bus). Magistrala dodaje wiadomość do *kolejki* (ang. queue) i natychmiast zwraca wynik, aby umożliwić wykonywanie od razu kolejnych operacji.

*Konsument* (ang. consumer) pracuje stale w tle, czytając nowe wiadomości w kolejce i wykonuje związane z nimi schematy działań. Konsument może działać na tym samym serwerze co aplikacja internetowa lub osobnym.

Jest to bardzo podobne do sposobu obsługi żądań HTTP, z wyjątkiem tego, że nie mamy odpowiedzi.

Kodowanie obsługi wiadomości (ang. message handler)
-----------------------------------------------------

Wiadomość jest klasą obiektu danych, z którą nie są związane żadne reguły działania. Będzie ona zserializowana, aby być przechowywana w kolejce, więc przechowuj tylko "proste" dane, które można serializować.

Stwórz klasę ``CommentMessage``:

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

W świecie Messengera nie mamy kontrolerów, lecz obiekty obsługi wiadomości (ang. message handlers).

Utwórz klasę ``CommentMessageHandler`` w nowej przestrzeni nazw ``App\MessageHandler``, która wie, jak obsługiwać wiadomości ``CommentMessage``:

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

``MessageHandlerInterface`` jest interfejsem *znacznikowym* (ang. marker interface). Pomaga on Symfony tylko w automatycznej rejestracji i konfiguracji klasy do obsługi wiadomości. Zgodnie z konwencją, reguły obsługi znajdują się w metodzie ``__invoke()``. Podpowiedź typu ``CommentMessage`` do jednego z argumentów tej metody mówi Messengerowi, którą klasę będzie obsługiwać.

Zaktualizuj kontroler, aby korzystać z nowego systemu:

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

Zamiast polegać na kontrolerze spamu, wysyłamy teraz wiadomość do magistrali. Następnie obiekt obsługi decyduje, co z nią zrobić.

Osiągnęliśmy coś nieoczekiwanego. Odłączyliśmy nasz kontroler od kontrolera spamu i przenieśliśmy schemat działania do nowej klasy – obsługi (ang. handler). Jest to idealne zastosowanie dla magistrali. Przetestuj kod: działa. Wszystko jest nadal wykonywane synchronicznie, ale kod jest już prawdopodobnie "lepszy".

Ograniczenie wyświetlanych komentarzy
--------------------------------------

Zaktualizuj reguły wyświetlania, aby uniknąć pojawienia się nieopublikowanych komentarzy na stronie:

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

Idziemy w prawdziwą asynchroniczność
---------------------------------------

.. index::
    single: RabbitMQ

Domyślnie, obsługa wywoływana jest synchronicznie. Aby przejść do trybu asynchronicznego, należy skonfigurować, której kolejki użyć dla każdego obiektu obsługującego wiadomości (ang. handler) w pliku konfiguracyjnym ``config/packages/messenger.yaml``:

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

Konfiguracja każe magistrali wysyłać instancje ``App\Message\CommentMessage`` do kolejki ``async``, która jest zdefiniowana przez DSN, przechowywany w zmiennej środowiskowej ``RABBITMQ_DSN``.

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

.. sidebar:: Zrzucanie (ang. dump) i przywracanie (ang. restore) bazy danych

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

Przetwarzanie wiadomości
-------------------------

Jeśli spróbujesz przesłać nowy komentarz, kontroler spamu nie będzie już wywoływany. Dodaj wywołanie ``error_log()`` w metodzie ``getSpamScore()``, aby to potwierdzić. Zamiast tego, wiadomość czeka w RabbitMQ, gotowa do skonsumowania przez jakiś proces.

.. index::
    single: Command;messenger:consume

Jak możesz się domyślić, Symfony posiada polecenie przetwarzania wiadomości. Uruchom je teraz:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Powinno ono natychmiast przetworzyć wiadomość wysłaną w związku z przesłanymi komentarzami:

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

Aktywność konsumentów jest zapisywana w logach, ale możesz otrzymać natychmiastową informację zwrotną na konsoli, przekazując flagę``-vv``. Możesz dostrzec nawet połączenie do API Akismet.

Aby zatrzymać konsumenta, naciśnij ``Ctrl+C``.

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

Robotnicy (ang. workers) działający w tle
-------------------------------------------

Zamiast uruchamiać konsumenta za każdym razem, gdy zamieszczamy komentarz, i zatrzymywać go natychmiast po tym, chcemy uruchomić go w sposób ciągły, bez otwierania zbyt wielu okien lub zakładek terminala.

Symfony CLI może zarządzać takimi poleceniami w tle (robotnikami) używając flagi demona (``-d``) na poleceniu ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Uruchom ponownie konsumenta wiadomości, ale umieść go w tle:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

Opcja ``--watch`` mówi Symfony, że polecenie musi zostać zrestartowane za każdym razem, gdy dochodzi do zmiany plików w katalogach ``config/``, ``src/``, ``templates/``, lub ``vendor/``.

.. note::

    Nie używaj ``-vv``, ponieważ otrzymasz zduplikowane wiadomości w ``server:log`` (wiadomości zarówno w logach i w konsoli).

Jeśli konsument przestanie pracować z jakiegoś powodu (limit pamięci, błąd, itp.), to zostanie automatycznie uruchomiony ponownie. Ale jeśli konsument przestanie pracować natychmiast po uruchomieniu – Symfony CLI podda się.

.. index::
    single: Symfony CLI;server:log

Logi są przesyłane strumieniowo przez ``symfony server:log`` wraz z wszystkimi innymi logami pochodzącymi z PHP, serwera WWW i aplikacji:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Użyj polecenia ``server:status``, aby wyświetlić listę wszystkich robotników pracujących w tle w bieżącym projekcie:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Aby zatrzymać robotnika, zatrzymaj serwer WWW lub zabij PID podany przez polecenie ``server:status``:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Ponawianie dostarczenia niedostarczonych wiadomości
----------------------------------------------------

A co, jeśli API Akismet nie działa podczas przetwarzania wiadomości? Twórca komentarza nie ma o tym pojęcia, ale wiadomość zostaje utracona i komentarz, którego dotyczyła, nie zostaje sprawdzony pod kątem bycia spamem.

Messenger posiada mechanizm ponawiania przetwarzania wiadomości w przypadku wystąpienia wyjątku podczas jej obsługi. Skonfigurujmy go:

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

Jeśli pojawi się problem podczas obsługi wiadomości, konsument spróbuje ponownie trzy razy przed rezygnacją. A potem zamiast odrzucić wiadomość, umieści ją w bardziej trwałym magazynie, w kolejce nieudanych przetworzeń (ang. ``failed``), która wykorzystuje bazę danych.

Przejrzyj nieudane wiadomości, a następnie ponów przetwarzanie za pomocą następujących poleceń:

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

Uruchamianie robotników na SymfonyCloud
----------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Aby przetwarzać wiadomości od RabbitMQ, musimy bezustannie uruchamiać polecenie ``messenger:consume``. Na SymfonyCloud jest to rola *robotnika* (ang. worker):

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

Podobnie jak Symfony CLI, SymfonyCloud zarządza restartami i logami.

.. index::
    single: Symfony CLI;logs

Aby odczytać logi robotnika, użyj:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Idąc dalej

    * `Samouczek SymfonyCasts Messenger <https://symfonycasts.com/screencast/messenger>`_;

    * 'Korporacyjna magistrala usług <https://pl.wikipedia.org/wiki/Enterprise_Service_Bus>'_ (ang. `enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_) oraz `wzorzec CQRS <https://martinfowler.com/bliki/CQRS.html>`_;

    * `Dokumentacja Symfony Messenger <https://symfony.com/doc/current/messenger.html>`_;

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
