Luarea deciziilor cu Workflow
=============================

.. index::
    single: Components;Workflow
    single: Workflow

A avea o stare pentru un model este destul de comun. Starea comentariilor este determinată doar de verificatorul de spam. Ce se întâmplă dacă adăugăm mai mulți factori de decizie?

Am putea dori să lăsăm administratorul site-ului să modereze toate comentariile după verificatorul de spam. Procesul ar fi ceva similar:

* Începe cu o stare ``submitted`` când un comentariu este trimis de un utilizator;

* Lasă verificatorul de spam să analizeze comentariul și schimbă starea pe ``potential_spam``, ``ham`` sau ``respins``;

* Dacă nu este respins, așteptă ca administratorul site-ului să decidă dacă comentariul este suficient de bun, comutând statul la ``published`` sau ``rejected``.

Implementarea acestei logici nu este prea complexă, însă adăugarea mai multor reguli ar crește mult complexitatea. În loc să codificăm logica, putem folosi componenta Symfony Workflow:

.. code-block:: bash

    $ symfony composer req workflow

Descrierea fluxurilor de lucru
------------------------------

Fluxul de lucru pentru comentarii poate fi descris în fișierul ``config/packages/workflow.yaml``:

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

Pentru a valida fluxul de lucru, generează o reprezentare vizuală:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Comanda ``dot`` face parte din utilitarul `Graphviz`_.

Utilizarea unui flux de lucru
-----------------------------

Înlocuește logica curentă din manipulatorul de mesaje cu fluxul de lucru:

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

Logică nouă are următoarea structură:

* Dacă tranziția ``accept`` este disponibilă pentru comentariul din mesaj, verifică dacă nu există spam;

* În funcție de rezultat, alege tranziția potrivită pentru a o aplica;

* Apelează ``apply()`` pentru a actualiza comentariul printr-un apel la metoda ``setState()``;

* Apelează ``flush()`` pentru a salva modificările în baza de date;

* Reexpediază mesajul pentru a permite fluxului de lucru să tranziteze din nou.

Deoarece nu am implementat validarea admin, la următoarea consumare a mesajului va fi logat mesajul „Dropping comment message”.

Să implementăm o validare automată până la următorul capitol:

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

Execută ``server symfony:log`` și adaugă un comentariu în frontend pentru a vedea toate tranzițiile care se petrec una după alta.

Găsirea serviciilor în Dependency Injection Container
-------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Când folosim injecția dependențelor, obținem servicii din dependency injection container prin sugerarea tipului unei interfețe sau, uneori, a numelui clasei care este o implementare specifică. Dar când o interfață are mai multe implementări, Symfony nu poate ghici de care ai nevoie. Avem nevoie de un mod de a fi explicit.

Am dat peste un astfel de exemplu prin injectarea unei ``WorkflowInterface`` în secțiunea anterioară.

Pe măsură ce injectăm orice instanță a interfeței generice `WorkflowInterface`` în constructor, cum poate Symfony să ghicească ce implementare a fluxului de lucru să utilizeze? Symfony folosește o convenție bazată pe numele argumentului: ``$commentStateMachine`` se referă la fluxul de lucru ``comment`` din configurație (a cărui tip este ``state_machine``). Încearcă orice alt nume de argument și va eșua.

Dacă nu îți amintești convenția, execută comanda ``debug:container``. Caută toate serviciile care conțin „workflow”:

.. code-block:: bash
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

Observă ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` care îți spune că folosirea ``$commentStateMachine`` ca un argument are o semnificație specială.

.. note::

    Am fi putut folosi comanda ``debug:autowiring`` așa cum am văzut într-un capitol anterior:

    .. code-block:: bash

        $ symfony console debug:autowiring workflow

.. sidebar:: Mergând mai departe

    * `Fluxuri de lucru și mașini de stare <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ și când să le alegi pe fiecare;

    * Documentație `Symfony Workflow <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
