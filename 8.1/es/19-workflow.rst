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

.. code-block:: terminal

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

.. code-block:: terminal

    $ symfony console workflow:dump comment --dump-format=mermaid

.. image:: images/workflow.png
    :align: center

.. note::

    El comando ``dot`` es parte de la utilidad `Graphviz`_.

Utilizando un *workflow*
------------------------

Reemplaza la lógica actual del manejador de mensajes por el *workflow*:

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

Ejecuta ``symfony server:log`` y añade un comentario en el *frontend* para ver todas las transiciones que van sucediendo, una tras otra.

Búsqueda de servicios en el Contenedor de Inyección de Dependencias
---------------------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Cuando utilizamos inyección de dependencias, obtenemos servicios del contenedor de inyección de dependencias mediante el tipo que indica una interfaz o, a veces, un nombre de clase de implementación concreto. Pero cuando una interfaz tiene varias implementaciones, Symfony no puede adivinar cuál necesitas. Necesitamos una forma de ser concretos.

Acabamos de encontrarnos con un ejemplo de este tipo con la inyección de una ``WorkflowInterface`` en la sección anterior.

Como inyectamos cualquier instancia de la interfaz genérica ``WorkflowInterface`` en el constructor, ¿cómo puede Symfony adivinar qué implementación de flujo de trabajo usar? Symfony usa una convención basada en el nombre del argumento: ``$commentStateMachine`` se refiere al flujo de trabajo de ``comment`` en la configuración (cuyo tipo es `` state_machine``). Prueba con cualquier otro nombre de argumento y fallará.

Si no recuerdas la convención, usa el comando ``debug:container``. Busca todos los servicios que contengan "workflow":

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

Observa la opción ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` que te indica que utilizar ``$commentStateMachine`` como nombre de argumento tiene un significado especial.

.. note::

    Podríamos haber usado el comando ``debug:autowiring`` como se vio en un capítulo anterior:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Yendo más allá

    * `Workflows y máquinas de estado <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ y cuándo elegir cada una de ellas;

    * Los `documentación de Workflow de Symfony <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
