Управление состоянием с помощью Workflow
====================================================================

.. index::
    single: Components;Workflow
    single: Workflow

Наличие какого-либо состояния у модели — довольно обычное явление. Состояние комментария определяет только антиспам-сервис. Но что, если в будущем у вас появятся больше факторов для изменения состояния?

Возможно, вы захотите дать администратору сайта возможность модерировать все комментарии после того, как они будут проверены антиспам-сервисом. Вот как будет выглядеть этот процесс:

* Начинаем с состояния ``submitted``, когда пользователь отправляет комментарий;

* Делегируем антиспам-сервису проанализировать комментарий и переключим его в зависимости от результата в одно из состояний: ``potential_spam``, ``ham`` или ``rejected``;

* Если комментарий не был отклонён (то есть он не спам), ожидаем, пока администратор не решит, достаточно ли комментарий хорош, изменив его состояние на ``published`` или ``rejected``.

Реализация данной логики — не слишком сложная задача. Однако добавление дополнительных правил значительно усложнит эту задачу. Воспользуемся Symfony-компонентом Workflow, чтобы не писать самим логику с нуля:

.. code-block:: bash

    $ symfony composer req workflow

Определение бизнес-процессов
------------------------------------------------------

Бизнес-процесс комментария можно описать в конфигурационном файле ``config/packages/workflow.yaml``:

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

Чтобы убедиться в правильности построения этого бизнес-процесса, давайте отобразим его визуально:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Команда ``dot``  является частью утилиты `Graphviz`_.

Использование бизнес-процессов
----------------------------------------------------------

Замените текущую логику в обработчике сообщений на новую с использованием определенного ранее бизнес-процесса:

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

Новая логика выглядит следующим образом:

* Если комментарий может перейти в состояние ``accept``, значит проверяем сообщение на спам;

* В зависимости от результата проверки, нужно выбрать подходящий переход;

* Вызываем метод ``apply()``, чтобы обновить состояние для объекта Comment, который в свою очередь вызывает в этом объекте метод ``setState()``;

* Сохраняем данные в базе данных, используя метод ``flush()``;

* Повторно отправляем сообщение на шину, чтобы ещё раз запустить бизнес-процесс комментария для определения следующего перехода.

Так как ещё не реализована возможность проверки сообщения администратором, при следующий обработке сообщения в лог запишется следующее: "Dropping comment message".

Перед тем как начать следующую главу, давайте добавим автоматическую проверку:

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

Выполните команду ``symfony server:log`` и добавьте комментарий к любой конференции, чтобы увидеть в терминале, как один за другим происходят переходы состояний.

.. sidebar:: Двигаемся дальше

    * `Бизнес-процессы и конечные автоматы  <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ и что выбрать из них;

    * `Документация по Symfony Workflow <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
