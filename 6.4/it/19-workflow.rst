Prendere decisioni con un Workflow
==================================

.. index::
    single: Components;Workflow
    single: Workflow

Avere uno stato per un modello è abbastanza comune. Lo stato del commento è determinato solo dallo spam checker. E se aggiungessimo altri fattori sui quali prendere delle decisioni?

Potremmo fare in modo che sia l'amministratore del sito a moderare tutti i commenti dopo il controllo dello spam. Il tutto si tradurrebbe in qualcosa di simile:

* Iniziamo con l'assegnare un stato ``submitted`` quando un commento viene inviato da un utente;

* Lasciamo che lo spam checker analizzi il commento e modifichi lo stato su ``potential_spam``, ``ham`` oppure ``rejected``;

* Se il commento non viene rifiutato dallo spam checker, aspettiamo che sia l'amministratore del sito a decidere se sia buono o meno, modificando lo stato in ``published`` oppure ``rejected``.

L'implementazione di questa logica è abbastanza semplice, ma come si può immaginare, all'aumentare del numero delle regole la complessità aumenterà. Invece di codificare noi stessi la logica, possiamo usare il componente Workflow di Symfony:

.. code-block:: terminal

    $ symfony composer req workflow

Descrivere i workflow
---------------------

Il workflow dei commenti può essere descritto nel file ``config/packages/workflow.yaml``:

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

Per convalidare il workflow, generiamo una rappresentazione visiva:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Il comando ``dot`` fa parte dell'utility `Graphviz`_.

Utilizzare un workflow
----------------------

Sostituire la logica corrente nel gestore dei messaggi con il workflow:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -6,7 +6,10 @@ use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
    +use Psr\Log\LoggerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
    +use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Workflow\WorkflowInterface;

     #[AsMessageHandler]
     class CommentMessageHandler
    @@ -15,6 +18,9 @@ class CommentMessageHandler
             private EntityManagerInterface $entityManager,
             private SpamChecker $spamChecker,
             private CommentRepository $commentRepository,
    +        private MessageBusInterface $bus,
    +        private WorkflowInterface $commentStateMachine,
    +        private ?LoggerInterface $logger = null,
         ) {
         }

    @@ -25,12 +31,18 @@ class CommentMessageHandler
                 return;
             }

    -        if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
    -            $comment->setState('spam');
    -        } else {
    -            $comment->setState('published');
    +        if ($this->commentStateMachine->can($comment, 'accept')) {
    +            $score = $this->spamChecker->getSpamScore($comment, $message->getContext());
    +            $transition = match ($score) {
    +                2 => 'reject_spam',
    +                1 => 'might_be_spam',
    +                default => 'accept',
    +            };
    +            $this->commentStateMachine->apply($comment, $transition);
    +            $this->entityManager->flush();
    +            $this->bus->dispatch($message);
    +        } elseif ($this->logger) {
    +            $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }
    -
    -        $this->entityManager->flush();
         }
     }

La nuova logica è la seguente:

* Se la transizione ``accept`` è disponibile per il commento nel messaggio, controllare la presenza di spam;

* A seconda del risultato, scegliere la transizione giusta da applicare;

* Chiamare ``apply()`` per aggiornare il commento tramite una chiamata al metodo ``setState()``;

* Chiamare ``flush()`` per salvare le modifiche nel database;

* Inviare nuovamente il messaggio per avviare la transizione del workflow.

Poiché non abbiamo implementato la validazione  da parte dell'amministratore, la prossima volta che il messaggio sarà consumato, verrà generato il seguente messaggio di log: "Dropping comment message".

Implementiamo un'auto-validazione fino al prossimo capitolo:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -41,6 +41,9 @@ class CommentMessageHandler
                 $this->commentStateMachine->apply($comment, $transition);
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
    +        } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    +            $this->commentStateMachine->apply($comment, $this->commentStateMachine->can($comment, 'publish') ? 'publish' : 'publish_ham');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

Eseguire ``symfony server:log`` e aggiungere un commento nel frontend per vedere tutte le transizioni che si susseguono una dopo l'altra.

Trovare servizi nel Dependency Injection Container
--------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Quando si utilizza la dependency injection, otteniamo i servizi dal dependency injection container attraverso il type hinting di un'interfaccia, oppure alcune volte attraverso il nome di una classe che rappresenta una implementazione concreta dell'interfaccia. Ma quando un'interfaccia ha più di una implementazione concreta, Symfony non può indovinare quale implementazione utilizzare. Abbiamo bisogno di un modo per essere espliciti.

Ci siamo appena imbattuti, nella sezione precedente, in un esempio in cui abbiamo iniettato l'interfaccia ``WorkflowInterface``.

Sicome stiamo iniettando l'interfaccia generica ``WorkflowInterface`` nel costruttore, come può Symfony indovinare quale implementazione del workflow utilizzare? Symfony usa una convenzione basata sul nome dell'argomento: ``$commentStateMachine`` si riferisce alla variabile di configurazione del workflow ``comment`` (il cui tipo è ``state_machine``). Provate con un altro argomento e fallirà.

Se non si ricorda la convezione, usare il comando ``debug::container``. Cerchiamo tutti i servizi che contengono "workflow":

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

Si noti la scelta ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine``, che dice che l'uso di ``$commentStateMachine`` come nome di parametro ha un significato speciale.

.. note::

    Avremmo potuto usare il comando ``debug:autowiring`` come visto in un capitolo precedente:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Andare oltre

    * Documentazione su `Workflows e State Machine`_ e quando scegliere una o l'altra;

    * Documentazione su `Symfony Workflow`_.

.. _`Graphviz`: https://www.graphviz.org/
.. _`Workflows e State Machine`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`Symfony Workflow`: https://symfony.com/doc/current/workflow.html
