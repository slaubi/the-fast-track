Ridimensionamento delle immagini
================================

Nel design della pagina della conferenza, le foto sono limitate a una dimensione massima di 200 x 150 pixel. Che ne dite di ottimizzare le immagini e ridurne le dimensioni se l'immagine originale caricata è più grande dei limiti?

Questo è un lavoro perfetto da aggiungere al workflow dei commenti, probabilmente subito dopo che il commento è stato convalidato e subito prima della sua pubblicazione.

Aggiungiamo un nuovo stato ``ready`` e una transizione ``optimize``:

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

Generare una rappresentazione visiva della nuova configurazione del workflow per assicurarci che descriva ciò che vogliamo:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Ottimizzare le immagini con Imagine
-----------------------------------

.. index::
    single: Imagine

L'ottimizzazione delle immagini sarà fatta grazie a `GD`_ (verificare che l'installazione locale di PHP abbia l'estensione GD abilitata) e `Imagine`_:

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.2"

Il ridimensionamento di un'immagine può essere eseguito tramite la seguente classe:

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

Dopo aver ottimizzato la foto, salviamo il nuovo file al posto di quello originale. Tuttavia, si potrebbe voler mantenere l'immagine originale da qualche parte.

Aggiungere un nuovo passo al workflow
-------------------------------------

Modificare il workflow per gestire il nuovo stato:

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

Si noti che ``$photoDir`` viene automaticamente iniettato, avendo in precedenza definito un *parametro* nel container per tale variabile:

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

Salvare i dati caricati in produzione
-------------------------------------

.. index::
    single: Platform.sh;File Service

Abbiamo già definito una speciale cartella in lettura e scrittura per i file caricati in ``.platform.app.yaml``. Ma il mount è locale. Se vogliamo che il container e il consumer dei messaggi siano in grado di accedere allo stesso mount, dobbiamo  creare un *servizio per i file*:

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

Usarlo per la cartella di caricamento delle foto:

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

Questo dovrebbe essere sufficiente per farlo funzionare in produzione.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
