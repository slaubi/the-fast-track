Зміна розміру зображень
============================================

У дизайні сторінки конференції фотографії обмежені максимальним розміром 200 на 150 пікселів. Як щодо оптимізації зображень та зменшення їх розміру, якщо завантажений оригінал перевищує встановлені обмеження?

Це ідеальне завдання, яке можна додати в робочий процес коментарів, ймовірно, відразу після перевірки коментаря і безпосередньо перед його публікацією.

Додаймо новий стан ``ready`` та перехід ``optimize``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/workflow.yaml
    +++ w/config/packages/workflow.yaml
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

Згенеруйте візуальне представлення нової конфігурації робочого процесу, щоб переконатися, що воно описує те, що ми хочемо:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Оптимізація зображень за допомогою Imagine
-------------------------------------------------------------------------

.. index::
    single: Imagine

Процеси оптимізації зображення будуть здійснені завдяки `GD`_ (переконайтеся, що у вашій локальній збірці PHP увімкнено розширення GD) та `Imagine`_:

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.5"

Зміну розміру зображення можна здійснити за допомогою наступного сервісного класу:

.. code-block:: php
    :caption: src/ImageOptimizer.php

    namespace App;

    use Imagine\Gd\Imagine;
    use Imagine\Image\Box;

    class ImageOptimizer
    {
        private const MAX_WIDTH = 200;
        private const MAX_HEIGHT = 150;

        private readonly Imagine $imagine;

        public function __construct()
        {
            $this->imagine = new Imagine();
        }

        public function resize(string $filename): void
        {
            [$iwidth, $iheight] = getimagesize($filename);
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

Після оптимізації фотографії ми зберігаємо новий файл замість оригінального. Хоча, можливо, ви захочете зберегти оригінальне зображення.

Додавання нового кроку в робочий процес
-------------------------------------------------------------------------

Змініть робочий процес для обробки нового стану:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -2,6 +2,7 @@

     namespace App\MessageHandler;

    +use App\ImageOptimizer;
     use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
    @@ -25,6 +26,8 @@ class CommentMessageHandler
             private WorkflowInterface $commentStateMachine,
             private MailerInterface $mailer,
             #[Autowire('%admin_email%')] private string $adminEmail,
    +        private ImageOptimizer $imageOptimizer,
    +        #[Autowire('%photo_dir%')] private string $photoDir,
             private ?LoggerInterface $logger = null,
         ) {
         }
    @@ -54,6 +57,12 @@ class CommentMessageHandler
                     ->to($this->adminEmail)
                     ->context(['comment' => $comment])
                 );
    +        } elseif ($this->commentStateMachine->can($comment, 'optimize')) {
    +            if ($comment->getPhotoFilename()) {
    +                $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());
    +            }
    +            $this->commentStateMachine->apply($comment, 'optimize');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

Зверніть увагу, що ``$photoDir`` впроваджено автоматично, оскільки ми визначили *параметр* контейнера до імені цієї змінної на попередньому кроці:

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

Зберігання завантажених даних у продакшн
----------------------------------------------------------------------------

.. index::
    single: Upsun;File Service

Ми вже визначили спеціальний каталог, доступний для читання і запису, для завантажених файлів у ``.platform.app.yaml``. Але він монтується локально. Якщо ми хочемо, щоб веб-контейнер і воркер споживача повідомлень мали можливість отримати доступ до того ж монтування, нам потрібно створити *файловий сервіс*:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -15,6 +15,9 @@ services:
                     type: string
                     path: config.vcl

    +    files:
    +        type: network-storage:2.0
    +
     applications:

Використовуйте його для каталогу завантаження фотографій:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -54,7 +54,7 @@ applications:
             mounts:
                 "/var/cache": { source: instance, source_path: var/cache }
                 "/var/share": { source: storage, source_path: var/share }
    -            "/public/uploads": { source: storage, source_path: uploads }
    +            "/public/uploads": { source: service, service: files, source_path: uploads }


             relationships:

Цього має бути достатньо для того, щоб функція працювала в продакшн.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
