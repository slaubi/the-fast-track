Изменение размера изображений
========================================================

По дизайну страницы конференции фотографии должны быть размером не более 200x150 пикселей. Может тогда нам стоит оптимизировать и уменьшать изображения, в случае если загруженные фотографии превышают указанный максимальный размер?

Идеальным решением будет включить такую операцию в бизнес-процесс комментария: после проверки и перед публикацией комментария.

Давайте добавим новое состояние ``ready`` и переход ``optimize``:

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

Посмотрим, как визуально выглядит измененный бизнес-процесс, чтобы убедиться, что он корректно составлен:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Оптимизация изображений с помощью Imagine
-----------------------------------------------------------------------

.. index::
    single: Imagine

Оптимизировать изображения будем через модуль `GD`_ (должен быть установлен на вашем компьютере) и `Imagine`_:

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.2"

Изменение размера изображения будет происходить в отдельном сервисном классе:

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

После оптимизации фотографии заменяем исходный файл на новый. Хотя, возможно, по каким-либо причинам вы решите оставить оригинальное изображение.

Добавление нового шага в бизнес-процесс
-------------------------------------------------------------------------

Добавьте в бизнес-процесс обработку нового состояния:

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

Обратите внимание на автоматически доступную переменную ``$photoDir``, так как ранее в одном из предыдущих шагов к этому имени переменной в контейнере мы привязали соответствующий *параметр*:

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

Хранение загруженных данных в продакшене
----------------------------------------------------------------------------

.. index::
    single: Platform.sh;File Service

В конфигурационном файле ``.platform.app.yaml`` уже указана специальная директория для чтения и записи загруженных фотографий. Однако она смонтирована только локально. Чтобы к данной директории имели доступ веб-контейнер и воркер для обработки сообщений в очереди, нужно создать *файловый сервис*:

.. code-block:: diff
    :caption: patch_file

    --- i/.platform/services.yaml
    +++ w/.platform/services.yaml
    @@ -12,3 +12,7 @@ varnish:
             vcl: !include
                 type: string
                 path: config.vcl
    +
    +files:
    +    type: network-storage:2.0
    +    disk: 256

Используйте его для директории загруженных фотографий:

.. code-block:: diff
    :caption: patch_file

    --- i/.platform.app.yaml
    +++ w/.platform.app.yaml
    @@ -31,7 +31,7 @@ web:

     mounts:
         "/var/cache": { source: local, source_path: var/cache }
    -    "/public/uploads": { source: local, source_path: uploads }
    +    "/public/uploads": { source: service, service: files, source_path: uploads }


     relationships:

Этого должно быть достаточно для работы в продакшене.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
