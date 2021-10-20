Making Decisions with a Workflow
================================

.. index::
    single: Components;Workflow
    single: Workflow

Having a state for a model is quite common. The comment state is only determined by the spam checker. What if we add more decision factors?

We might want to let the website admin moderate all comments after the spam checker. The process would be something along the lines of:

* Start with a ``submitted`` state when a comment is submitted by a user;

* Let the spam checker analyze the comment and switch the state to either ``potential_spam``, ``ham``, or ``rejected``;

* If not rejected, wait for the website admin to decide if the comment is good enough by switching the state to ``published`` or ``rejected``.

Implementing this logic is not too complex, but you can imagine that adding more rules would greatly increase the complexity. Instead of coding the logic ourselves, we can use the Symfony Workflow Component:

.. code-block:: bash

    $ symfony composer req workflow

Describing Workflows
--------------------

The comment workflow can be described in the ``config/packages/workflow.yaml`` file:

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

To validate the workflow, generate a visual representation:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    The ``dot`` command is a part of the `Graphviz`_ utility.

Using a Workflow
----------------

Replace the current logic in the message handler with the workflow:

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

The new logic reads as follows:

* If the ``accept`` transition is available for the comment in the message, check for spam;

* Depending on the outcome, choose the right transition to apply;

* Call ``apply()`` to update the Comment via a call to the ``setState()`` method;

* Call ``flush()`` to commit the changes to the database;

* Re-dispatch the message to allow the workflow to transition again.

As we haven't implemented the admin validation, the next time the message is consumed, the "Dropping comment message" will be logged.

Let's implement an auto-validation until the next chapter:

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

Run ``symfony server:log`` and add a comment in the frontend to see all transitions happening one after the other.

Finding Services from the Dependency Injection Container
--------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

When using dependency injection, we get services from the dependency injection container by type hinting an interface or sometimes a concrete implementation class name. But when an interface has several implementations, Symfony cannot guess which one you need. We need a way to be explicit.

We have just come across such an example with the injection of a ``WorkflowInterface`` in the previous section.

As we inject any instance of the generic ``WorkflowInterface`` interface in the contructor, how can Symfony guess which workflow implementation to use? Symfony uses a convention based on the argument name: ``$commentStateMachine`` refers to the ``comment`` workflow in the configuration (which type is ``state_machine``). Try any other argument name and it will fail.

If you don't remember the convention, use the ``debug:container`` command. Search for all services containing "workflow":

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

Notice choice ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` which tells you that using ``$commentStateMachine`` as an argument name has a special meaning.

.. note::

    We could have used the ``debug:autowiring`` command as seen in a previous chapter:

    .. code-block:: bash

        $ symfony console debug:autowiring workflow

.. sidebar:: Going Further

    * `Workflows and State Machines <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ and when to choose each one;

    * The `Symfony Workflow docs <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
