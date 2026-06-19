调整图片尺寸
==================

在会议页面的设计中，照片被限定为不超过 200 像素宽和 150 像素高。如果上传的原始图片比这个尺寸大，那我们如何来优化图片，调整它们的大小，怎么样？

这个工作是加入评论工作流的不二之选，很可能把它加在评论验证之后，发布它之前。

我们来添加 ``ready`` 状态和 ``optimize`` 状态转移：

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

为这个新的工作流配置生成一个图形，用它来验证这是我们想要的流程：

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

用 Imagine 来优化图片
---------------------------

.. index::
    single: Imagine

图片优化需要用 `GD`_ （检查你本地安装的PHP是否启用了 GD 扩展）和 `Imagine`_ ：

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.5"

可以用下面这个服务类来调整图片大小：

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

图片优化完成后，我们存储新的图片文件来代替原始图片。但你可能也会想要在某个地方保留原始图片。

在工作流里增加一个新步骤
------------------------------------

修改工作流，让它可以处理新的状态：

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

注意 ``$photoDir`` 是自动注入的，因为在之前的步骤中我们已经在容器的 *bind* 配置里绑定了这个变量名：

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

在生产环境中存储上传的数据
---------------------------------------

.. index::
    single: SymfonyCloud;File Service

我们已经在 ``.symfony.cloud.yaml`` 里为上传文件定义了一个特殊的可读写目录。但是挂载是本地的。如果我们想要让 web 容器和消息消费者的 worker 进程也可以访问同一个挂载，我们需要创建一个 *文件服务*：

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

把它用于照片上传目录：

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

在生产环境中，这么做就足以让该功能正常运行了。

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
