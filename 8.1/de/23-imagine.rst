Bilder skalieren
================

Das Layout der Konferenzseite beschränkt die Fotos auf eine maximale Größe von 200 x 150 Pixel. Warum optimieren wir die Bilder nicht, indem wir sie verkleinern, falls das ursprünglich hochgeladene Bild größer ist als unsere Vorgabe?

Diese Aufgabe eignet sich perfekt, um zum Kommentar-Workflow hinzugefügt zu werden; wahrscheinlich direkt, nachdem der Kommentar validiert wird und bevor er publiziert wird.

Lass uns einen neuen ``ready``-Zustand und einen ``optimize``-Übergang hinzufügen:

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

Erzeuge eine Visualisierung der neuen Workflow-Konfiguration, um zu überprüfen, ob sie beschreibt, was wir wollen:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Bilder mit Imagine optimieren
-----------------------------

.. index::
    single: Imagine

Die Bildoptimierungen werden mittels `GD`_ (überprüfe, ob Deine lokale PHP-Installation die GD-Erweiterung aktiviert hat) und `Imagine`_ durchgeführt:

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.5"

Die Skalierung eines Bildes kann über die folgende Service-Klasse erfolgen:

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

Nach der Optimierung des Fotos speichern wir die neue Datei anstelle der ursprünglichen. Das Originalbild solltest du jedoch eventuell behalten.

Einen neuen Schritt zum Workflow hinzufügen
--------------------------------------------

Ändere den Workflow, um den neuen Zustand abzubilden und anzuwenden:

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

Beachte: ``$photoDir`` wird automatisch injiziert, da wir in einem vorhergehenden Schritt einen Container-*Parameter* für diesen Variablennamen festgelegt haben:

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

Hochgeladene Dateien auf dem Produktivsystem speichern
------------------------------------------------------

.. index::
    single: Upsun;File Service

Wir haben bereits ein spezielles Verzeichnis mit Lese- und Schreibberechtigung für hochgeladene Dateien in der ``.upsun/config.yaml`` definiert. Der Mount ist jedoch nur für den Anwendungscontainer lokal. Wenn wir wollen, dass sowohl der Web-Container als auch der Message-Consumer-Worker auf denselben Mount zugreifen können, müssen wir einen Datei-Service (*file service*) anlegen:

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

Verwende ihn für das Foto-Upload-Verzeichnis:

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

Dies sollte ausreichen, damit das Feature im Produktivsystem funktioniert.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
