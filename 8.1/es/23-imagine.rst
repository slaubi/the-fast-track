Redimensionando imágenes
=========================

En el diseño de la página de la conferencia, las fotos están limitadas a un tamaño máximo de 200 por 150 píxeles. ¿No sería una buena idea optimizar las imágenes y reducir su tamaño si el original enviado supera estos límites?

Se trata de una tarea que se puede añadir perfectamente al workflow de los comentarios, posiblemente justo después de validar el comentario e inmediatamente antes de que se publique.

Añadamos un nuevo estado ``ready`` y una transición de tipo ``optimize``:

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

Genera una representación visual de la nueva configuración del workflow para validar que describe lo que queremos:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Optimizando imágenes con Imagine
---------------------------------

.. index::
    single: Imagine

Las optimizaciones de las imágenes se harán gracias a `GD`_ (comprueba que tu instalación local de PHP tenga la extensión GD habilitada) e `Imagine`_:

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.5"

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

Después de optimizar la foto, almacenaremos el nuevo archivo en lugar del original. Sin embargo, es posible que deseemos conservar la imagen original.

Añadiendo un nuevo paso en el workflow
---------------------------------------

Modifica el workflow para gestionar el nuevo estado:

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

Ten en cuenta que ``$photoDir`` se inyecta automáticamente ya que en un paso anterior definimos esa variable como *parameter* en el contenedor:

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

Almacenando los datos subidos en producción
--------------------------------------------

.. index::
    single: Upsun;File Service

Ya hemos definido en ``.upsun/config.yaml`` un directorio especial de lectura-escritura para los archivos que se vayan subiendo, pero es un punto de montaje local al contenedor de la aplicación. Si queremos que tanto el contenedor web y el *worker* consumidor de mensajes puedan acceder al mismo punto de montaje, necesitamos crear un *servicio de ficheros* :

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

Úsalo para el directorio de subida de las fotos:

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

Esto debería ser suficiente para que funcione en producción.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
