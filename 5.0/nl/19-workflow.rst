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

.. code-block:: bash

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

.. code-block:: bash
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
    @@ -50,6 +50,9 @@ class CommentMessageHandler implements MessageHandlerInterface
                 $this->entityManager->flush();

                 $this->bus->dispatch($message);
    +        } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    +            $this->workflow->apply($comment, $this->workflow->can($comment, 'publish') ? 'publish' : 'publish_ham');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

Draai ``symfony server:log`` en voeg een reactie toe in de frontend om alle overgangen achter elkaar te zien gebeuren.

.. sidebar:: Verder gaan

    * `Workflows en State Machines <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ en wanneer welke te kiezen;

    * De `Symfony Workflow documentatie <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
