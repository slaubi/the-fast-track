Prendre des décisions avec un workflow
=======================================

.. index::
    single: Components;Workflow
    single: Workflow

Avoir un état pour un modèle est assez commun. L'état du commentaire n'est déterminé que par le vérificateur de spam. Et si on ajoutait d'autres critères de décision ?

Nous pourrions laisser l'admin du site modérer tous les commentaires après le vérificateur de spam. Le processus serait quelque chose comme :

* Commencez par un état ``submitted`` lorsqu'un commentaire est soumis par un internaute ;

* Laissez le vérificateur de spam analyser le commentaire et changer l'état en ``potential_spam``, ``ham`` ou ``rejected``

* S'il n'est pas rejeté, attendez que l'admin du site décide si le commentaire est suffisamment utile en changeant l'état pour ``published`` ou ``rejected``.

La mise en œuvre de cette logique n'est pas trop complexe, mais vous pouvez imaginer que l'ajout de règles supplémentaires augmenterait considérablement la complexité. Au lieu de coder la logique nous-mêmes, nous pouvons utiliser le composant Symfony Workflow :

.. code-block:: terminal

    $ symfony composer req workflow

Décrire des workflows
----------------------

Le workflow de commentaires peut être décrit dans le fichier ``config/packages/workflow.yaml`` :

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

Pour valider le workflow, générez une représentation visuelle :

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    La commande ``dot`` fait partie de l'utilitaire `Graphviz`_.

Utiliser un workflow
--------------------

Remplacez la logique actuelle dans le gestionnaire de messages par le workflow :

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

La nouvelle logique se lit comme ceci :

* Si la transition ``accept`` est disponible pour le commentaire dans le message, vérifiez si c'est un spam ;

* Selon le résultat, choisissez la bonne transition à appliquer ;

* Appellez ``apply()`` pour mettre à jour le Comment via un appel à la méthode ``setState()`` ;

* Appelez ``flush()`` pour valider les changements dans la base de données ;

* Réexpédiez le message pour permettre au workflow d'effectuer une nouvelle transition.

Comme nous n'avons pas implémenté la fonctionnalité de validation par l'admin, la prochaine fois que le message sera consommé, le message "Dropping comment message" sera enregistré.

Mettons en place une validation automatique en attendant le prochain chapitre :

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

Exécutez ``symfony server:log`` et ajoutez un commentaire sur le site pour voir toutes les transitions se produire les unes après les autres.

Trouver des services depuis le conteneur d'injection de dépendances
--------------------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Quand nous utilisons l'injection de dépendances, nous récupérons des services depuis le conteneur d'injection de dépendances en utilisant le typage par interface ou parfois par une implémentation de classe concrète. Mais quand une interface à plusieurs implémentations, Symfony ne peut deviner celle dont vous avez besoin. Nous avons besoin d'être explicite.

Nous venons juste de rencontrer un cas semblable avec l'injection de ``WorkflowInterface`` dans la section précédente.

Comme nous injectons n'importe quelle instance de l'interface générique ``WorkflowInterface`` dans le constructeur, comment Symfony peut savoir quelle implémentation du workflow utiliser ? Symfony utilise une convention basée sur le nom de l'argument : ``$commentStateMachine`` fait référence au workflow ``comment`` de la configuration (dont le type est ``state_machine``). Essayez n'importe quel autre argument et l'injection échouera.

Si vous ne vous rappelez pas de la convention, utilisez la commande ``debug:container``. Cherchez tous les services contenant "workflow" :

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

Remarquez le choix ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` qui vous indique qu'utiliser ``$commentStateMachine`` comme argument nommé a une signification particulière.

.. note::

    Nous aurions pu utiliser la commande ``debug:autowiring`` comme vu dans un précédent chapitre :

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Aller plus loin

    * `Workflows et State Machines`_ et quand les choisir ;

    * La `documentation du composant Symfony Workflow`_.

.. _`Graphviz`: https://www.graphviz.org/
.. _`Workflows et State Machines`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`documentation du composant Symfony Workflow`: https://symfony.com/doc/current/workflow.html
