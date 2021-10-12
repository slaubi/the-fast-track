Tomando decisiones con un *workflow*
====================================

.. index::
    single: Components;Workflow
    single: Workflow

Que un modelo tenga un estado es algo bastante habitual. El estado de un comentario está determinado ahora mismo únicamente por el verificador de spam. ¿Y si incluimos más elementos de decisión?

Puede que queramos dejar que el administrador del sitio web modere todos los comentarios después de que lo haga el verificador de spam. El proceso sería algo así como:

* Comienza con un estado ``submitted`` cuando un usuario envía un comentario;

* Deja que el verificador de spam analice el comentario y cambie el estado a ``potential_spam``, ``ham``, o ``rejected``;

* Si no se rechaza, espera a que el administrador del sitio web decida si el comentario es lo suficientemente bueno cambiando el estado a ``published`` o ``rejected``.

La implementación de esta lógica no es demasiado compleja, pero te puedes imaginar que añadir más reglas aumentaría enormemente la complejidad. En lugar de codificar la lógica nosotros mismos, podemos usar el componente Workflow de Symfony.

.. code-block:: bash

    $ symfony composer req workflow

Describiendo *workflows*
------------------------

El flujo de trabajo de comentarios se puede describir en el archivo``config/packages/workflow.yaml`` :

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

Para comprobar que el flujo de trabajo es correcto, genera una representación visual:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    El comando ``dot`` es parte de la utilidad `Graphviz`_.

Utilizando un *workflow*
------------------------

Reemplaza la lógica actual del manejador de mensajes por el *workflow*:

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

La nueva lógica es la siguiente:

* Si la transición ``accept`` está disponible para el comentario en el mensaje, comprueba si hay spam;

* Dependiendo del resultado, elige la transición correcta a aplicar;

* Ejecuta ``apply()`` para actualizar el Comentario a través de una llamada al método ``setState()``;

* Ejecuta ``flush()`` para confirmar los cambios en la base de datos;

* Vuelve a enviar el mensaje para permitir que el *workflow* vuelva a cambiar.

Como no hemos implementado la validación de administrador, la próxima vez que se consuma el mensaje, se registrará el mensaje "Dropping comment message" (descartando el mensaje del comentario).

Vamos a implementar una auto-validación que modificaremos en el próximo capítulo:

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

Ejecuta ``symfony server:log`` y añade un comentario en el *frontend* para ver todas las transiciones que van sucediendo, una tras otra.

.. sidebar:: Yendo más allá

    * `Workflows y máquinas de estado <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ y cuándo elegir cada una de ellas;

    * Los `documentación de Workflow de Symfony <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
