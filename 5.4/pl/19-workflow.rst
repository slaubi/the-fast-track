Podejmowanie decyzji przy użyciu komponentu Workflow
=====================================================

.. index::
    single: Components;Workflow
    single: Workflow

Modelowanie procesu za pomocą maszyny stanowej jest dość powszechne. Stan komentarza jest określany tylko przez klasę sprawdzającą spam. Co jeśli dodamy więcej czynników decyzyjnych?

Możemy pozwolić administratorom strony moderować wszystkie komentarze po sprawdzeniu spamu. Proces ten byłby czymś w rodzaju:

* Rozpocznij od stanu ``submitted`` (wysłany), kiedy komentarz zostanie przesłany przez użytkownika;

* Pozwól, aby kontroler spamu przeanalizował komentarz i przełączył stan na ``potential_spam`` (potencjalnie spam), ``ham`` (nie spam), lub ``rejected`` (odrzucono);

* Jeśli nie zostanie odrzucony, poczekaj, aż osoba administrująca stroną zdecyduje, czy komentarz jest wystarczająco dobry, przełączając stan na ``published`` lub ``rejected``.

Wdrożenie tego schematu działania nie jest zbyt skomplikowane, ale można sobie wyobrazić, że dodanie większej liczby reguł znacznie zwiększyłoby jego złożoność. Zamiast samodzielnie kodować schemat działania, możemy użyć komponentu Symfony Workflow:

.. code-block:: terminal

    $ symfony composer req workflow

Opisywanie przepływów pracy (ang. workflow)
---------------------------------------------

Przepływ komentarzy można opisać w pliku: ``config/packages/workflow.yaml``

.. code-block:: yaml
    :caption: config/packages/workflow.yaml
    :emphasize-lines: 3,4,9,11

    framework:
        workflows:
            comment:
                type: state_machine
                audit_trail:
                    enabled: "%kernel.debug%"
                marking_store:
                    type: 'method'
                    property: 'state'
                supports:
                    - App\Entity\Comment
                initial_marking: submitted
                places:
                    - submitted
                    - ham
                    - potential_spam
                    - spam
                    - rejected
                    - published
                transitions:
                    accept:
                        from: submitted
                        to:   ham
                    might_be_spam:
                        from: submitted
                        to:   potential_spam
                    reject_spam:
                        from: submitted
                        to:   spam
                    publish:
                        from: potential_spam
                        to:   published
                    reject:
                        from: potential_spam
                        to:   rejected
                    publish_ham:
                        from: ham
                        to:   published
                    reject_ham:
                        from: ham
                        to:   rejected

.. index::
    single: Command;workflow:dump

Aby sprawdzić poprawność przepływu pracy, wygeneruj jego wizualną reprezentację:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Polecenie ``dot`` jest częścią narzędzia `Graphviz`_.

Korzystanie z przepływu pracy
------------------------------

Zastąp bieżący schemat obsługi komunikatów przepływem pracy (ang. workflow):

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -6,19 +6,28 @@ use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
    +use Psr\Log\LoggerInterface;
     use Symfony\Component\Messenger\Handler\MessageHandlerInterface;
    +use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Workflow\WorkflowInterface;

     class CommentMessageHandler implements MessageHandlerInterface
     {
         private $spamChecker;
         private $entityManager;
         private $commentRepository;
    +    private $bus;
    +    private $workflow;
    +    private $logger;

    -    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository)
    +    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, LoggerInterface $logger = null)
         {
             $this->entityManager = $entityManager;
             $this->spamChecker = $spamChecker;
             $this->commentRepository = $commentRepository;
    +        $this->bus = $bus;
    +        $this->workflow = $commentStateMachine;
    +        $this->logger = $logger;
         }

         public function __invoke(CommentMessage $message)
    @@ -28,12 +37,21 @@ class CommentMessageHandler implements MessageHandlerInterface
                 return;
             }

    -        if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
    -            $comment->setState('spam');
    -        } else {
    -            $comment->setState('published');
    -        }

    -        $this->entityManager->flush();
    +        if ($this->workflow->can($comment, 'accept')) {
    +            $score = $this->spamChecker->getSpamScore($comment, $message->getContext());
    +            $transition = 'accept';
    +            if (2 === $score) {
    +                $transition = 'reject_spam';
    +            } elseif (1 === $score) {
    +                $transition = 'might_be_spam';
    +            }
    +            $this->workflow->apply($comment, $transition);
    +            $this->entityManager->flush();
    +
    +            $this->bus->dispatch($message);
    +        } elseif ($this->logger) {
    +            $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
    +        }
         }
     }

Nowy schemat działania wygląda następująco:

* Jeśli możliwe jest przejście oznaczone jako ``accept`` (zaakceptowano) dla komentarza w wiadomości (ang. message), sprawdź, czy komentarz nie jest spamem;

* W zależności od wyniku, wybierz właściwą drogę przepływu pracy;

* Aby zaktualizować encję Comment wywołaj metodę ``apply()``. Spowoduje to wywołanie metody ``setState()``;

* Wywołaj metodę ``flush()`` w celu zapisania zmian w bazie danych;

* Ponownie wyślij wiadomość (ang. message), aby umożliwić ponowne przejście przepływu pracy.

Ponieważ nie zaimplementowaliśmy walidacji przez osoby posiadające konto administracyjne, następnym razem, gdy wiadomość zostanie przetworzona, zostanie zalogowana wiadomość "Zostawiono komentarz".

Na razie zaimplementujmy automatyczną walidację, którą zmienimy w kolejnym rozdziale:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -50,6 +50,9 @@ class CommentMessageHandler implements MessageHandlerInterface
                 $this->entityManager->flush();

                 $this->bus->dispatch($message);
    +        } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    +            $this->workflow->apply($comment, $this->workflow->can($comment, 'publish') ? 'publish' : 'publish_ham');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

Uruchom ``symfony server:log`` i dodaj komentarz w aplikacji, aby zobaczyć wszystkie przejścia zachodzące jedno po drugim.

Znajdowanie usług z kontenera wstrzykiwania zależności (ang. Dependency Injection Container)
-----------------------------------------------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Kiedy używamy wstrzykiwania zależności, pobieramy usługi z kontenera wstrzykiwania zależności używając podpowiadania typu (ang. type hinting) interfejsu lub konkretnej implementacji w formie nazwy klasy. Jednak, kiedy interfejs posiada wiele implementacji, Symfony nie może zgadnąć, której konkretnie potrzebujesz. Musimy być bardziej precyzyjni.

W poprzedniej sekcji spotkaliśmy się już z takim problemem w przypadku wstrzykiwania ``WorkflowInterface``.

Wstrzykując instancję generycznego interfejsu ``WorkflowInterface`` w konstruktorze, Symfony nie jest w stanie odgadnąć, której implementacji przepływu pracy (ang. workflow) użyć. Symfony używa konwencji bazującej na nazwie argumentu: ``$commentStateMachine``, który odnosi się do przepływu pracy ``comment`` zdefiniowanego w konfiguracji (którego typem jest ``state_machine``). Jeżeli spróbujesz użyć jakiejkolwiek innej nazwy argumentu, zakończy się to niepowodzeniem.

Jeżeli nie pamiętasz jaka jest konwencja, użyj polecenia ``debug:container``. Wyszukaj wszystkie usługi zawierające "workflow":

.. code-block:: terminal
    :emphasize-lines: 12
    :class: ignore

    $ symfony console debug:container workflow

     Select one of the following services to display its information:
      [0] console.command.workflow_dump
      [1] workflow.abstract
      [2] workflow.marking_store.method
      [3] workflow.registry
      [4] workflow.security.expression_language
      [5] workflow.twig_extension
      [6] monolog.logger.workflow
      [7] Symfony\Component\Workflow\Registry
      [8] Symfony\Component\Workflow\WorkflowInterface $commentStateMachine
      [9] Psr\Log\LoggerInterface $workflowLogger
     >

Zwróć uwagę na wybór o numerze ``8`` - ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine``, który mówi nam, że używanie ``$commentStateMachine`` jako argumentu, posiada specjalne znaczenie.

.. note::

    Mogliśmy użyć polecenia ``debug: autowiring``, jak pokazano w poprzednim rozdziale:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Idąc dalej

    * `Przepływy pracy i maszyny stanu`_ oraz informacje, kiedy ich używać;

    * `Dokumentacja Symfony Workflow`_.

.. _`Graphviz`: https://www.graphviz.org/
.. _`Przepływy pracy i maszyny stanu`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`Dokumentacja Symfony Workflow`: https://symfony.com/doc/current/workflow.html
