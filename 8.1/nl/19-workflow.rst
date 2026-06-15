Beslissingen nemen door middel van een workflow
===============================================

.. index::
    single: Components;Workflow
    single: Workflow

Het hebben van een state voor een model is vrij gebruikelijk. De state van een reactie wordt alleen bepaald door de spam checker. Wat als we meer beslissingsfactoren toevoegen?

We willen een websitebeheerder misschien wel de reacties laten modereren, na het checken op spam. Het proces zou er dan als volgt uit zien:

* We beginnen met een ``submitted`` state wanneer een reactie wordt ingediend door een gebruiker;

* Laat de spamchecker de reactie analyseren en zet de status om naar ``potential_spam``, ``ham`` of ``rejected``;

* Indien niet afgewezen, wacht dan tot de websitebeheerder beslist of de reactie goed genoeg is door de staat op ``published`` of ``rejected`` te zetten.

Het implementeren van deze logica is niet al te complex, maar je kan je voorstellen dat het toevoegen van meer regels de complexiteit aanzienlijk kan vergroten. In plaats van zelf de logica te coderen, kunnen we het Symfony Workflow-component gebruiken:

.. code-block:: terminal

    $ symfony composer req workflow

Workflows beschrijven
---------------------

De reactie-workflow kan in het ``config/packages/workflow.yaml`` bestand worden beschreven:

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

Om de workflow te valideren, genereer je een visuele weergave:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Het ``dot`` command is een onderdeel van de `Graphviz`_ utility.

Een workflow gebruiken
----------------------

Vervang de huidige logica in de message-handler door de workflow:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
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

De nieuwe logica luidt als volgt:

* Als de ``accept``-overgang beschikbaar is voor de reactie in het bericht, controleer dan op spam;

* Kies, afhankelijk van de uitkomst, de juiste overgang om toe te passen;

* Roep ``apply()`` aan, om de reactie bij te werken via een aanroep op de ``setState()`` methode;

* Roep ``flush()`` aan om de wijzigingen in de database vast te leggen;

* Verstuur het bericht opnieuw om de workflow opnieuw toe te passen.

Aangezien we de adminvalidatie niet hebben geïmplementeerd, zal de volgende keer dat het bericht wordt verwerkt, "Dropping comment message" worden gelogd.

Laten we een automatische validatie implementeren tot aan het volgende hoofdstuk:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
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

Draai ``symfony server:log`` en voeg een reactie toe in de frontend om alle overgangen achter elkaar te zien gebeuren.

Services zoeken in de Dependency Injection-container
----------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Bij het gebruik van dependency injection krijgen we services van de dependency injection container door een interface of soms een concrete implementatie-class-naam te typen. Maar als een interface meerdere implementaties heeft, kan Symfony niet raden welke jij nodig hebt. We hebben een manier nodig om expliciet te zijn.

We zijn zojuist zo'n voorbeeld tegengekomen met de injectie van een 'WorkflowInterface' in de vorige paragraaf.

Hoe kan Symfony raden welke workflow implementatie moet worden gebruikt, aangezien we een exemplaar van de generieke ``WorkflowInterface``-interface in de constructor injecteren? Symfony gebruikt een conventie gebaseerd op de argumentnaam: ``$commentStateMachine`` verwijst naar de ``comment`` workflow in de configuratie (die van het type ``state_machine`` is). Wanneer je een andere argumentnaam probeert zal dit mislukken.

Als je de conventie niet meer weet, gebruik dan het commando ``debug: container``. Zoek naar alle services met "workflow":

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

Let op keuze ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` die je vertelt dat het gebruik van ``$commentStateMachine`` als argumentnaam een speciale betekenis heeft.

.. note::

    We hadden het commando ``debug:autowiring`` kunnen gebruiken, zoals we in een vorig hoofdstuk hebben gezien:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Verder gaan

    * `Workflows en State Machines`_ en wanneer welke te kiezen;

    * De `Symfony Workflow documentatie`_.

.. _`Graphviz`: https://www.graphviz.org/
.. _`Workflows en State Machines`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`Symfony Workflow documentatie`: https://symfony.com/doc/current/workflow.html
