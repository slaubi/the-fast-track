Mit einem Workflow Entscheidungen treffen
=========================================

.. index::
    single: Components;Workflow
    single: Workflow

Einen State (Zustand) für ein Modell zu haben, ist durchaus üblich. Der Kommentar-Zustand wird nur vom Spam-Checker bestimmt. Was passiert, wenn wir weitere Entscheidungsfaktoren hinzufügen?

Vielleicht möchten wir alle Kommentare nach dem Spam-Checker durch Website-Administrator*innen moderieren lassen. Der Prozess würde etwa so aussehen:

* Beginne mit einem ``submitted``-Zustand, wenn ein Kommentar von einem*r Benutzer*in abgegeben wird;

* Lasse den Kommentar vom Spam-Checker analysieren und setze den Zustand entweder auf ``potential_spam``, auf ``ham``, oder auf ``rejected``;

* Wenn der Zustand nicht ``rejected`` ist, warte bis ein*e Website-Administrator*in entscheidet, ob der Kommentar gut genug ist und den Zustand auf ``published`` oder ``rejected`` ändert.

Die Implementierung dieser Logik ist nicht allzu schwierig, aber Du kannst Dir bestimmt vorstellen, dass das Hinzufügen weiterer Regeln die Komplexität deutlich steigern würde. Anstatt diese Logik selbst zu programmieren, können wir die Symfony Workflow Komponente verwenden:

.. code-block:: bash

    $ symfony composer req workflow

Workflows definieren
--------------------

Der Kommentar-Workflow kann in der Datei ``config/packages/workflow.yaml`` definiert werden:

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

Erzeuge eine Visualisierung, um den Workflow zu überprüfen:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Der ``dot``-Befehl ist Teil des `Graphviz`_ -Dienstprogramms.

Einen Workflow verwenden
------------------------

Ersetze die aktuelle Logik im Message-Handler durch den Workflow:

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

Die neue Logik lautet wie folgt:

* Überprüfe die Nachricht auf Spam, wenn der ``accept``-Übergang für den Kommentar in der Nachricht verfügbar ist.

* Abhängig vom Ergebnis wendest Du den entsprechenden Übergang an;

* Führe ``apply()`` aus, um den Kommentar durch einen Aufruf der ``setState()``-Methode zu aktualisieren;

* Rufe ``flush()`` auf, um die Änderungen in der Datenbank zu speichern;

* Versende die Nachricht erneut, damit der nächste Übergang im Workflow stattfinden kann.

Da wir die Admin-Validierung nicht implementiert haben, wird beim nächsten Verarbeiten der Nachricht "Dropping comment message" geloggt.

Implementieren wir bis zum nächsten Kapitel eine automatische Validierung!

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

Führe ``symfony server:log`` aus und füge einen Kommentar im Frontend hinzu, um alle Übergänge nacheinander zu sehen.

.. sidebar:: Weiterführendes

    * `Workflows und Zustandsmaschinen <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ und wann man was wählen sollte;

    * Die `Symfony Workflow Dokumentation <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
