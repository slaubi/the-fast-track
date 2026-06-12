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

.. code-block:: terminal

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

Erzeuge eine Visualisierung im Mermaid-Format, um den Workflow zu überprüfen:

.. code-block:: terminal

    $ symfony console workflow:dump comment --dump-format=mermaid

Füge die Ausgabe in den `Mermaid Live Editor`_ ein, um sie zu rendern; GitHub und GitLab rendern Mermaid-Diagramme in Markdown-Dateien ebenfalls nativ:

.. image:: images/workflow.png
    :align: center

Einen Workflow verwenden
------------------------

Ersetze die aktuelle Logik im Message-Handler durch den Workflow:

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

Führe ``symfony server:log`` aus und füge einen Kommentar im Frontend hinzu, um alle Übergänge nacheinander zu sehen.

Services (Dienste) vom Dependency Injection Container finden
------------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Wenn wir Dependency Injection nutzen, bekommen wir Services (Dienste) vom Dependency Injection Container wenn wir als Type-Hint ein Interface oder einen konkreten Klasse-Namen angeben. Aber wenn das Interface mehrere Ausführungen hat, kann Symfony nicht mehr erraten, welches Du meinst. Wir müssen einen Weg finden, um das genau anzugeben.

Gerade solch eine direkte Angabe für die Dependency Injection hatten wir im vorherigen Abschnitt mit der Injection eines ``WorkflowInterface``.

Wenn wir irgendeine Instanz des generischen `WorkflowInterface``-Interface im Contructor angeben, wie kann Symfony dann raten welche Workflow-Anwendung genutzt werden soll? Symfony nutzt eine Konvention basierend auf dem Argument-Namen: ``$commentStateMachine`` bezieht sich auf den  ``comment``-Workflow in der Konfiguration (dessen Typ ``state_machine`` ist). Probiere irgendein anderes Argument und es wird fehlschlagen.

Falls Du die Konventionen nicht mehr weisst, nutze den ``debug:container``-Befehl. Suche nach allen Services (Diensten) die "workflow" beinhalten:

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

Siehst Du die Option ``8``? ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` sagt Dir, dass die Nutzung von ``$commentStateMachine`` als Argument eine besondere Bedeutung hat.

.. note::

    Wir hätten auch den ``debug:autowiring``-Befehl nutzen können, wie wir im vorherigen Kapitel gesehen haben:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Weiterführendes

    * `Workflows und Zustandsmaschinen`_ und wann man was wählen sollte;

    * Die `Symfony Workflow Dokumentation`_.

.. _`Mermaid Live Editor`: https://mermaid.live/
.. _`Workflows und Zustandsmaschinen`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`Symfony Workflow Dokumentation`: https://symfony.com/doc/current/workflow.html
