Tomando Decisões com um Workflow
=================================

.. index::
    single: Components;Workflow
    single: Workflow

Ter um estado para um modelo é bastante comum. O estado do comentário é determinado apenas pelo verificador de spam. E se adicionarmos mais fatores de decisão?

Nós podemos querer deixar o administrador do site moderar todos os comentários após o verificador de spam. O processo seria algo como:

* Comece com um estado ``submitted`` quando um comentário é enviado por um usuário;

* Deixe o verificador de spam analisar o comentário e mudar o estado para ``potential_spam``, ``ham`` ou ``rejected``;

* Se não for rejeitado, aguarde o administrador do site decidir se o comentário é bom o suficiente, mudando o estado para ``published`` ou ``rejected``.

Implementar essa lógica não é muito complexo, mas você pode imaginar que adicionar mais regras aumentaria bastante a complexidade. Em vez de programar a lógica, podemos usar o Componente Workflow do Symfony:

.. code-block:: bash

    $ symfony composer req workflow

Descrevendo Workflows
---------------------

O workflow dos comentários pode ser descrito no arquivo ``config/packages/workflow.yaml``:

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

Para validar o workflow, gere uma representação visual:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    O comando ``dot`` faz parte do utilitário `Graphviz`_.

Usando um Workflow
------------------

Substitua a lógica atual no manipulador de mensagens pelo workflow:

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

A nova lógica diz o seguinte:

* Se a transição ``accept`` estiver disponível para o comentário na mensagem, verifique se há spam;

* Dependendo do resultado, escolha a transição correta a ser aplicada;

* Chame ``apply()`` para atualizar o Comment através de uma chamada ao método ``setState()``;

* Chame ``flush()`` para fazer o commit das alterações no banco de dados;

* Reenvie a mensagem para permitir que o workflow faça a transição novamente.

Como não implementamos a validação do administrador, na próxima vez que a mensagem for consumida, a mensagem "Dropping comment message" será registrada no log.

Vamos implementar uma validação automática até o próximo capítulo:

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

Execute ``symfony server:log`` e adicione um comentário no frontend para ver todas as transições acontecendo uma após a outra.

.. sidebar:: Indo Além

    * `Workflows e Máquinas de Estado <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ e quando escolher cada um;

    * A `documentação do Workflow do Symfony <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
