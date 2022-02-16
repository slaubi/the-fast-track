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

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Powinniśmy także upewnić się, że domyślna wartość atrybutu ``state`` jest ustawiona na ``submitted``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -38,8 +38,8 @@ class Comment
         #[ORM\Column(type: 'string', length: 255, nullable: true)]
         private $photoFilename;

    -    #[ORM\Column(type: 'string', length: 255)]
    -    private $state;
    +    #[ORM\Column(type: 'string', length: 255, options: ["default" => "submitted"])]
    +    private $state = 'submitted';

         public function __toString(): string
         {

.. index::
    single: Command;make:migration

Utwórz migrację bazy danych:

.. code-block:: terminal

    $ symfony console make:migration

Zmodyfikuj migrację tak, by zaktualizowała status wszystkich istniejących komentarzy na ``published``:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -21,6 +21,7 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs
             $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) DEFAULT \'submitted\' NOT NULL');
    +        $this->addSql("UPDATE comment SET state='published'");
         }

         public function down(Schema $schema): void

.. index::
    single: Command;doctrine:migrations:migrate

Uruchom migrację bazy danych:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

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

Zaktualizuj konfigurację EasyAdmin tak, by móc zobaczyć stan komentarza:

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
    +        $comment = self::getContainer()->get(CommentRepository::class)->findOneByEmail($email);
    +        $comment->setState('published');
    +        self::getContainer()->get(EntityManagerInterface::class)->flush();
    +
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }

Z poziomu testu PHPUnit można uzyskać dowolną usługę z kontenera za pośrednictwem ``self::getContainer()->get()``; daje to również dostęp do usług niepublicznych.

Zrozumienie komponentu Messenger
--------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Zarządzanie kodem asynchronicznym w Symfony jest zadaniem komponentu Messenger:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

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

Zamiast polegać na kontrolerze spamu, wysyłamy teraz wiadomość do magistrali. Następnie obiekt obsługi decyduje, co z nią zrobić.

Osiągnęliśmy coś nieoczekiwanego. Odłączyliśmy nasz kontroler od kontrolera spamu i przenieśliśmy schemat działania do nowej klasy – obsługi (ang. handler). Jest to idealne zastosowanie dla magistrali. Przetestuj kod: działa. Wszystko jest nadal wykonywane synchronicznie, ale kod jest już prawdopodobnie "lepszy".

Idziemy w prawdziwą asynchroniczność
---------------------------------------

Domyślnie, obsługa wywoływana jest synchronicznie. Aby przejść do trybu asynchronicznego, należy skonfigurować, której kolejki użyć dla każdego obiektu obsługującego wiadomości (ang. handler) w pliku konfiguracyjnym ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -21,4 +21,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

Konfiguracja każe magistrali wysyłać instancje ``App\Message\CommentMessage`` do kolejki ``async``, która jest zdefiniowana przez DSN (``MESSENGER_TRANSPORT_DSN``), przechowywany w pliku zmiennych środowiskowych ``.env``. Krótko mówiąc, używamy PostgreSQL jako kolejki dla naszych wiadomości.

Skonfiguruj tabele i wyzwalacze w PostgreSQL:

.. code-block:: terminal

    $ symfony console make:migration

I zmigruj bazę danych:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. tip::

    Za kulisami Symfony używa wbudowanego PostgreSQL, wydajnego, skalowalnego i transakcyjnego systemu pub/sub (``LISTEN``/``NOTIFY``). Możesz także przeczytać rozdział o RabbitMQ, jeśli chcesz go używać zamiast PostgreSQL jako brokera wiadomości.

Przetwarzanie wiadomości
-------------------------

Jeśli spróbujesz przesłać nowy komentarz, kontroler spamu nie będzie już wywoływany. Dodaj wywołanie ``error_log()`` w metodzie ``getSpamScore()``, aby to potwierdzić. Zamiast tego, wiadomość czeka w kolejce, gotowa do skonsumowania przez jakiś proces.

.. index::
    single: Command;messenger:consume

Jak możesz się domyślić, Symfony posiada polecenie przetwarzania wiadomości. Uruchom je teraz:

.. code-block:: terminal
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

Robotnicy (ang. workers) działający w tle
-------------------------------------------

Zamiast uruchamiać konsumenta za każdym razem, gdy zamieszczamy komentarz, i zatrzymywać go natychmiast po tym, chcemy uruchomić go w sposób ciągły, bez otwierania zbyt wielu okien lub zakładek terminala.

Symfony CLI może zarządzać takimi poleceniami w tle (robotnikami) używając flagi demona (``-d``) na poleceniu ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Uruchom ponownie konsumenta wiadomości, ale umieść go w tle:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async -vv

Opcja ``--watch`` mówi Symfony, że polecenie musi zostać zrestartowane za każdym razem, gdy dochodzi do zmiany plików w katalogach ``config/``, ``src/``, ``templates/``, lub ``vendor/``.

.. note::

    Nie używaj ``-vv``, ponieważ otrzymasz zduplikowane wiadomości w ``server:log`` (wiadomości zarówno w logach i w konsoli).

Jeśli konsument przestanie pracować z jakiegoś powodu (limit pamięci, błąd, itp.), to zostanie automatycznie uruchomiony ponownie. Ale jeśli konsument przestanie pracować natychmiast po uruchomieniu – Symfony CLI podda się.

.. index::
    single: Symfony CLI;server:log

Logi są przesyłane strumieniowo przez ``symfony server:log`` wraz z wszystkimi innymi logami pochodzącymi z PHP, serwera WWW i aplikacji:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Użyj polecenia ``server:status``, aby wyświetlić listę wszystkich robotników pracujących w tle w bieżącym projekcie:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Aby zatrzymać robotnika, zatrzymaj serwer WWW lub zabij PID podany przez polecenie ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Ponawianie dostarczenia niedostarczonych wiadomości
----------------------------------------------------

A co, jeśli API Akismet nie działa podczas przetwarzania wiadomości? Twórca komentarza nie ma o tym pojęcia, ale wiadomość zostaje utracona i komentarz, którego dotyczyła, nie zostaje sprawdzony pod kątem bycia spamem.

Messenger posiada mechanizm ponawiania przetwarzania wiadomości w przypadku wystąpienia wyjątku podczas jej obsługi:

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 3,12-15
    :class: ignore

    framework:
        messenger:
            failure_transport: failed

            transports:
                # https://symfony.com/doc/current/messenger.html#transport-configuration
                async:
                    dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                    options:
                        use_notify: true
                        check_delayed_interval: 60000
                    retry_strategy:
                        max_retries: 3
                        multiplier: 2
                failed: 'doctrine://default?queue_name=failed'
                # sync: 'sync://'

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

Jeśli pojawi się problem podczas obsługi wiadomości, konsument spróbuje ponownie trzy razy przed rezygnacją. A potem zamiast odrzucić wiadomość, umieści ją w trwałej kolejce nieudanych przetworzeń (ang. ``failed``), która wykorzystuje inną tabelę w bazie danych.

Przejrzyj nieudane wiadomości, a następnie ponów przetwarzanie za pomocą następujących poleceń:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Uruchamianie robotników na Platform.sh
---------------------------------------

.. index::
    single: Platform.sh;Workers
    single: Workers

Aby przetwarzać wiadomości od PostgreSQL, musimy bezustannie uruchamiać polecenie ``messenger:consume``. Na Platform.sh jest to rola *robotnika* (ang. worker):

.. code-block:: yaml
    :caption: .platform.app.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Podobnie jak Symfony CLI, Platform.sh zarządza restartami i logami.

.. index::
    single: Symfony CLI;cloud:logs

Aby odczytać logi robotnika, użyj:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:logs --worker=messages all

.. sidebar:: Idąc dalej

    * `Samouczek SymfonyCasts Messenger`_;

    * 'Korporacyjna magistrala usług <https://pl.wikipedia.org/wiki/Enterprise_Service_Bus>'_ (ang. `enterprise service bus`_) oraz `wzorzec CQRS`_;

    * `Dokumentacja Symfony Messenger`_;

.. _`Samouczek SymfonyCasts Messenger`: https://symfonycasts.com/screencast/messenger
.. _`enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`wzorzec CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`Dokumentacja Symfony Messenger`: https://symfony.com/doc/current/messenger.html
