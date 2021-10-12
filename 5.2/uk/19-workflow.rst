Прийняття рішень за допомогою робочого процесу
=======================================================================================

.. index::
    single: Components;Workflow
    single: Workflow

Наявність стану для моделі є досить поширеним явищем. Стан коментаря визначається лише засобом перевірки на спам. Що, якщо ми додамо більше факторів прийняття рішень?

Ми можемо дозволити адміністратору веб-сайту модерувати всі коментарі після перевірки на спам. Цей процес буде виглядати приблизно так:

* Почнемо зі стану ``submitted``, коли користувач відправив коментар;

* Нехай засіб перевірки на спам проаналізує коментар і перемкне стан на будь-який із наступних: ``potential_spam``, ``ham``, або ``rejected``;

* Якщо не відхилено, зачекаємо, поки адміністратор веб-сайту вирішить, чи достатньо хороший коментар, перемкнувши стан на ``published`` або ``rejected``.

Реалізація цієї логіки не надто складна, але можна уявити, що додавання більшої кількості правил значно збільшить складність. Замість того щоб програмувати логіку самостійно, ми можемо використовувати компонент Symfony Workflow:

.. code-block:: bash

    $ symfony composer req workflow

Опис робочих процесів
----------------------------------------

Робочий процес коментаря може бути описаний у файлі ``config/packages/workflow.yaml``:

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

Для перевірки робочого процесу згенеруйте візуальне представлення:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    Команда ``dot`` є частиною утиліти `Graphviz`_.

Використання робочого процесу
--------------------------------------------------------

Замініть поточну логіку в обробнику повідомлень на нову, з використанням робочого процесу:

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

Нова логіка виглядає наступним чином:

* Якщо для коментаря у повідомленні доступний перехід у стан ``accept``, перевіряємо повідомлення на спам;

* Залежно від результату, вибираємо правильний перехід для застосування;

* Викликаємо ``apply()``, щоб оновити коментар за допомогою виклику методу ``setState()``;

* Викликаємо ``flush()``, щоб зафіксувати зміни в базі даних;

* Повторно відправляємо повідомлення, щоб запустити робочий процес для визначення наступного переходу.

Оскільки ми не реалізували перевірку адміністратором, наступного разу, коли повідомлення буде опрацьовано, в журнал запишеться "Dropping comment message".

Реалізуймо автоматичну перевірку до наступного розділу:

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

Виконайте ``symfony server:log`` і додайте коментар на фронтенді, щоб побачити всі переходи, що відбуваються один за іншим.

Пошук сервісів із контейнера впровадження залежностей
-----------------------------------------------------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

Використовуючи впровадження залежностей ми отримуємо сервіси з контейнера впровадження залежностей за типом, що вказує на інтерфейс чи, іноді, за конкретним іменем класу реалізації. Але коли інтерфейс має кілька реалізацій, Symfony не може здогадатися, яка саме вам потрібна. Нам потрібен спосіб бути відвертими.

Ми щойно натрапили на такий приклад з впровадженням ``WorkflowInterface`` у попередньому розділі.

Оскільки ми впроваджуємо будь-який екземпляр універсального інтерфейсу ``WorkflowInterface`` в конструктор, як Symfony може вгадати, яку реалізацію робочого процесу використовувати? Symfony використовує домовленість, засновану на імені аргументу: ``$commentStateMachine`` відноситься до робочого процесу ``comment`` в конфігурації (тип якого — ``state_machine``). Спробуйте використовувати будь-яке інше ім'я аргументу, і це викличе помилку.

Якщо ви не пам'ятаєте домовленості, використовуйте команду ``debug:container``. Пошук всіх сервісів, що містять "workflow":

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

Зверніть увагу на вибір ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` який повідомляє, що використання ``$commentStateMachine`` у якості імені аргументу має особливе значення.

.. note::

    Ми могли б використовувати команду ``debug:autowiring``, як показано в попередньому розділі:

    .. code-block:: bash

        $ symfony console debug:autowiring workflow

.. sidebar:: Йдемо далі

    * `Робочі процеси й скінченні автомати <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ та коли вибрати кожного з них;

    * `Документація по Symfony Workflow <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
