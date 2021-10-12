Redimensionando imágenes
========================

En el diseño de la página de la conferencia, las fotos están limitadas a un tamaño máximo de 200 por 150 píxeles. ¿No sería una buena idea optimizar las imágenes y reducir su tamaño si el original enviado supera estos límites?

Se trata de una tarea que se puede añadir perfectamente al workflow de los comentarios, posiblemente justo después de validar el comentario e inmediatamente antes de que se publique.

Añadamos un nuevo estado ``ready`` y una transición de tipo ``optimize``:

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

Genera una representación visual de la nueva configuración del workflow para validar que describe lo que queremos:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Optimizando imágenes con Imagine
--------------------------------

.. index::
    single: Imagine

Las optimizaciones de las imágenes se harán gracias a `GD`_ (comprueba que tu instalación local de PHP tenga la extensión GD habilitada) e `Imagine`_:

.. code-block:: bash

    $ symfony composer req "imagine/imagine:^1.2"

El redimensionamiento de una imagen se puede realizar a través de la siguiente clase de servicio:

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

Después de optimizar la foto, almacenaremos el nuevo archivo en lugar del original. Sin embargo, es posible que deseemos conservar la imagen original.

Añadiendo un nuevo paso en el workflow
--------------------------------------

Modifica el workflow para gestionar el nuevo estado:

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

Ten en cuenta que ``$photoDir`` se inyecta automáticamente ya que en un paso anterior definimos esa variable como *bind* en el contenedor:

.. code-block:: yaml
    :caption: config/packages/services.yaml
    :class: ignore

    services:
        _defaults:
            bind:
                $photoDir: "%kernel.project_dir%/public/uploads/photos"

Almacenando los datos subidos en producción
-------------------------------------------

.. index::
    single: SymfonyCloud;File Service

Ya hemos definido en ``.symfony.cloud.yaml`` un directorio especial de lectura-escritura para los archivos que se vayan subiendo, pero es un punto de montaje local. Si queremos que tanto el contenedor web y el *worker* consumidor de mensajes puedan acceder al mismo punto de montaje, necesitamos crear un *servicio de ficheros* :

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -11,3 +11,7 @@ varnish:
             vcl: !include
                 type: string
                 path: config.vcl
    +
    +files:
    +    type: network-storage:1.0
    +    disk: 256

Úsalo para el directorio de subida de las fotos:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -37,7 +37,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    -    "/public/uploads": { source: local, source_path: uploads }
    +    "/public/uploads": { source: service, service: files, source_path: uploads }

     hooks:
         build: |

Esto debería ser suficiente para que funcione en producción.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
