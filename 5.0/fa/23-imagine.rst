تغییر اندازه‌ی تصاویر
=========================================

در طراحی صفحه‌ی کنفرانس، اندازه تصاویر به ۲۰۰ در ۱۵۰ پیکسل محدود شده است. پس بهینه‌سازی و کاهش حجم تصاویر درصورتی که نسخه‌ی اصلی بارگذاری‌شده از حدود تعیین‌شده بیشتر باشد چه می‌شود؟

این یک کار فوق‌العاده است که می‌تواند به جریان‌کار کامنت اضافه شود. احتمالاً درست پس از اعتبارسنجی کامنت و قبل از انتشار آن.

بیایید یک وضعیت جدید ``ready`` و یک تحول ``optimize`` اضافه کنیم:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/workflow.yaml
    +++ b/config/packages/workflow.yaml
    @@ -16,6 +16,7 @@ framework:
                     - potential_spam
                     - spam
                     - rejected
    +                - ready
                     - published
                 transitions:
                     accept:
    @@ -29,13 +30,16 @@ framework:
                         to:   spam
                     publish:
                         from: potential_spam
    -                    to:   published
    +                    to:   ready
                     reject:
                         from: potential_spam
                         to:   rejected
                     publish_ham:
                         from: ham
    -                    to:   published
    +                    to:   ready
                     reject_ham:
                         from: ham
                         to:   rejected
    +                optimize:
    +                    from: ready
    +                    to:   published

.. index::
    single: Command;workflow:dump

ارائه‌ی بصری پیکربندی جریان‌کار جدید را تولید کنید تا تصدیق نمایید که همان چیزی است که می‌خواهید:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

بهینه‌سازی تصاویر با Imagine
-----------------------------------------------

.. index::
    single: Imagine

بهینه‌سازی تصاویر به لطف `GD`_ (بررسی کنید که افزونه‌ی GD در نصب محلی PHP فعال باشد) و `Imagine`_ انجام می‌گردد:

.. code-block:: bash

    $ symfony composer req "imagine/imagine:^1.2"

تغییر اندازه‌ی تصاویر می‌تواند به کمک کلاس سرویس زیر انجام شود:

.. code-block:: php
    :caption: src/ImageOptimizer.php

    namespace App;

    use Imagine\Gd\Imagine;
    use Imagine\Image\Box;

    class ImageOptimizer
    {
        private const MAX_WIDTH = 200;
        private const MAX_HEIGHT = 150;

        private $imagine;

        public function __construct()
        {
            $this->imagine = new Imagine();
        }

        public function resize(string $filename): void
        {
            list($iwidth, $iheight) = getimagesize($filename);
            $ratio = $iwidth / $iheight;
            $width = self::MAX_WIDTH;
            $height = self::MAX_HEIGHT;
            if ($width / $height > $ratio) {
                $width = $height * $ratio;
            } else {
                $height = $width / $ratio;
            }

            $photo = $this->imagine->open($filename);
            $photo->resize(new Box($width, $height))->save($filename);
        }
    }

ما پس از بهینه‌سازی تصاویر، فایل جدید را در همان مکان تصویر اصلی ذخیره می‌کنیم. البته شما ممکن است بخواهید تصویر اصلی را نگه‌دارید.

افزودن یک گام جدید در جریان‌کار
----------------------------------------------------------

جریان‌کار را اصلاح کنید تا به یک وضعیت جدید رسیدگی کند:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -2,6 +2,7 @@

     namespace App\MessageHandler;

    +use App\ImageOptimizer;
     use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
    @@ -21,10 +22,12 @@ class CommentMessageHandler implements MessageHandlerInterface
         private $bus;
         private $workflow;
         private $mailer;
    +    private $imageOptimizer;
         private $adminEmail;
    +    private $photoDir;
         private $logger;

    -    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, MailerInterface $mailer, string $adminEmail, LoggerInterface $logger = null)
    +    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, MailerInterface $mailer, ImageOptimizer $imageOptimizer, string $adminEmail, string $photoDir, LoggerInterface $logger = null)
         {
             $this->entityManager = $entityManager;
             $this->spamChecker = $spamChecker;
    @@ -32,7 +35,9 @@ class CommentMessageHandler implements MessageHandlerInterface
             $this->bus = $bus;
             $this->workflow = $commentStateMachine;
             $this->mailer = $mailer;
    +        $this->imageOptimizer = $imageOptimizer;
             $this->adminEmail = $adminEmail;
    +        $this->photoDir = $photoDir;
             $this->logger = $logger;
         }

    @@ -64,6 +69,12 @@ class CommentMessageHandler implements MessageHandlerInterface
                     ->to($this->adminEmail)
                     ->context(['comment' => $comment])
                 );
    +        } elseif ($this->workflow->can($comment, 'optimize')) {
    +            if ($comment->getPhotoFilename()) {
    +                $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());
    +            }
    +            $this->workflow->apply($comment, 'optimize');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

توجه کنید که ``$photoDir`` به صورت خودکار تزریق شده است، چرا که ما یک *اتصال (bind)* کانتینر بر روی نام این متغیر تعریف کردیم:

.. code-block:: yaml
    :caption: config/packages/services.yaml
    :class: ignore

    services:
        _defaults:
            bind:
                $photoDir: "%kernel.project_dir%/public/uploads/photos"

ذخیره‌ی داده‌های بارگذاری‌شده در محیط عمل‌آوری
-------------------------------------------------------------------------------------------

.. index::
    single: SymfonyCloud;File Service

ما در حال حاضر یک پوشه‌ی مخصوص قابل خواندن و نوشتن برای فایل‌های باگذاری‌شده در ``.symfony.cloud.yaml`` تعریف کرده‌ایم. اما mount آن محلی است. اگر می‌خواهیم که کانتینر وب و کارگر مصرف‌کننده‌ی پیغام، قادر به دسترسی به همین mount مشترک باشند، لازم است که یک *سرویس فایل* ایجاد کنیم:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -19,3 +19,7 @@ varnish:
             vcl: !include
                 type: string
                 path: config.vcl
    +
    +files:
    +    type: network-storage:1.0
    +    disk: 256

از آن به عنوان پوشه‌ی بارگذاری تصاویر استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -41,7 +41,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    -    "/public/uploads": { source: local, source_path: uploads }
    +    "/public/uploads": { source: service, service: files, source_path: uploads }

     hooks:
         build: |

انجام کارهای اخیر باید برای کارکردن قابلیت‌ها در محیط عمل‌آوری کافی باشد.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
