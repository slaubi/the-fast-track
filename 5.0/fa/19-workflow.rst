تصمیم‌گیری با یک جریان‌کار
===================================================

.. index::
    single: Components;Workflow
    single: Workflow

داشتن یک وضعیت برای یک مدل، کاملاً عادی است. وضعیت یک کامنت، تنها توسط بررسی‌کننده‌ی محتوای هرز تعیین شده است. اگر ما فاکتورهای تصمیم‌گیری بیشتری اضافه کنیم، چه اتفاقی می‌افتد؟

ما ممکن است بخواهیم که پس از بررسی‌کننده‌ی داده‌های هرز، مدیر سایت نیز کامنت‌ها را تعدیل کند. فرآیند رویه‌ای مشابه زیر خواهد داشت:

* هنگامی که یک کامنت ارسال شد، با وضعیت ``submitted`` شروع کن؛

* بگذار که بررسی‌کننده‌ی داده‌های هرز، کامنت را تحلیل کند و وضعیت را به یکی از وضعیت‌های ``potential_spam``، ``ham`` یا ``rejected`` تغییر دهد.

* اگر کامنت رد نشد، صبر کن تا مدیر وب‌سایت با تغییر وضعیت به ``published`` یا ``rejected``، تصمیم بگیرد که آیا کامنت به اندازه‌ی کافی خوب است یا خیر.

پیاده‌سازی این منطق خیلی پیچیده نیست، اما شما می‌توانید تصور کنید که اضافه‌شدن قواعد بیشتر، پیچیدگی را به میزان زیادی افزایش خواهد داد. ما می‌توانیم به جای اینکه خودمان این منطق را کدنویسی کنیم، از کامپوننت جریان‌کار سیمفونی اسفاده کنیم:

.. code-block:: bash

    $ symfony composer req workflow

توصیف جریان‌کارها
----------------------------------

جریان‌کار کامنت می‌تواند در فایل ``config/packages/workflow.yaml`` توصیف گردد:

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

برای اعتبارسنجی جریان‌کار، یک ارائه‌ی تصویری تولید کنید:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow.png
    :align: center

.. note::

    فرمان ``dot`` بخشی از مجموعه ابزار `Graphviz`_ است.

استفاده از یک جریان‌کار
--------------------------------------------

منطق موجود در رسیدگی‌کننده‌ی پیغام را با جریان‌کار جایگزین کنید:

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

منطق جدید به این صورت خوانده می‌شود:

* اگر برای کامنت موجود در پیغام، تحولِ ``accept`` در دسترس است، هرز‌بودن محتوای آن را بررسی کن؛

* بسته به نتیجه‌ی این بررسی، تحول (transition) صحیح را برای اعمال‌کردن انتخاب کن؛

* ``apply()`` را فراخوانی کن تا از طریق فراخوانی متد ``setState()``، شیءِ Comment را به‌روزرسانی کند؛

* ``flush()`` را فرخوانی کن تا تغییرات در پایگاه‌داده اعمال شود؛

* پیغام را بازاعزام (Re-dispatch) کن تا اجازه داده شود که جریان‌کار دوباره به کار بیافتد؛

از آنجایی که هنوز اعتبارسنجی مدیر را پیاده‌سازی نکرده‌ایم، دفعه‌ی بعدی که پیغام مصرف شود، «Dropping comment message» لاگ خواهد شد.

بیایید تا قبل از یک پیاده‌سازی واقعی در فصل آینده، فعلاً یک اعتبارسنجی خودکار را پیاده‌سازی کنیم:

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

``symfony server:log`` را اجرا کنید و یک کامنت در جلوی صحنه ایجاد کنید تا تمام وقوع تحولات را یکی پس از دیگری ببینید.

.. sidebar:: بیشتر بدانید

    * `جریان‌کارها و ماشین‌های حالت (State Machines) <https://symfony.com/doc/current/workflow/workflow-and-state-machine.html>`_ و اینکه هر کدام را باید در چه شرایطی انتخاب کرد؛

    * `مستندات جریان‌کار سیمفونی <https://symfony.com/doc/current/workflow.html>`_.

.. _`Graphviz`: https://www.graphviz.org/
